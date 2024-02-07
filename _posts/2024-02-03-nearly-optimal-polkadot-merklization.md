---
title: "Nearly Optimal State Merklization (in Polkadot-SDK)"
layout: post
excerpt: "Building upon ideas for a novel Merkle Trie database"
twitter-image: TODO
---

Recently, my friend and coworker [Sergei (Pepyakin)](https://pep.wtf) sent me an article from Preston Evans on the subject of a more optimal Merkle Trie format designed to be efficient on SSDs. The original article is [here](https://www.prestonevans.me/nearly-optimal-state-merklization/) and I highly recommend it as background reading to this post.

TODO: twitter card https://twitter.com/sovereign_labs/status/1744768837982011472

The optimizations presented in the original post sparked a two-day conversation with Pep in which we discussed how this might be made to work with the [Polkadot-SDK](https://github.com/paritytech/polkadot-sdk) as well. Polkadot-SDK, while it also uses a Merkle Trie to store state, was designed on a differing set of assumptions, and so the original approach would need to be adapted in order to be integrated. This post might be seen as a summary of our conversation, covering some history, some of the original optimizations, the differences in assumptions, and tweaks that may be made in order to maintain full backwards compatibility with the Polkadot-SDK. Some familiarity with [Merkle Tries](https://en.wikipedia.org/wiki/Merkle_tree), especially as they are used as blockchain state databases, will help in comprehending this article, but all are welcome to come along for the ride.

Preston's proposed system, in a nutshell, is a new binary Merkle Trie format and database schema that is extremely low-overhead and amenable to SSDs with most (if not all) disk accesses being predictable with no other information beyond the key being queried. We'll revisit more specifics later, though I highly recommend reading the original blog post for a high-fidelity explanation. 

------

But first, let's cover _why_ optimization of the Merkle Trie is so important. The open secret about scaling serial blockchain execution is that most of the time isn't actually spent on executing transactions: it's on reading and writing state data to and from disk, such as accounts, balances, and smart contract state. Merkle Tries in blockchains are not strictly necessary, but they provide a means of easily proving some of the state of the ledger to a light client. They are a key part of permissionless blockchain systems which seek to include light clients, not just full nodes, as full participants.

Traversing a Merkle Trie to load some particular piece of state is a log(n) operation in the number of items within the trie. Updates to the Merkle Trie are logarithmic as well. The implication is that a larger state, even if the majority of this state is dormant, makes reading and writing more expensive. This is the real reason why Ethereum hasn't "just increased the gas limit" and why Polkadot has required deposits for all stored state: state growth is a huge problem, adds bloat to the Merkle Trie, and slows everything else down as a result. What makes Preston's article so important is that it shows a way for us to maintain the wonderful advantages of merklized state while drastically reducing the overheads associated with previous implementations.

When it comes to the Polkadot-SDK, I see this design being far more useful to Parachains than the Polkadot Relay Chain itself. The Relay Chain has relatively little state, having offloaded most of its work onto System Parachains. For parachains, the benefits will come for two reasons. Reason one is that it's more data-efficient, utilizing less of what we refer to in Polkadot-land as the Proof-of-Validity. Reason two is that (at the time of this writing), work on [Elastic Scaling](https://github.com/paritytech/polkadot-sdk/issues/1829) is underway, and it will in theory bound the throughput of a parachain at the rate a single node is capable of processing transactions. I foresee a future for some parachains where the Merkle Trie will be a bottleneck.

------

In 2016, the first significant project I worked on in the blockchain space was optimizing the Parity-Ethereum node's implementation of Ethereum's Merkle-Patricia Trie. At that time, blockchain node technology was a lot less sophisticated. We were in a friendly rivalry with the geth team, and one way we wanted to get a leg up on geth's performance was by implementing batch writes, where we'd only compute all of the changes in the Merkle-Patricia trie once at the end of the block (actually, the end of each transaction - at that time, transaction receipts all carried an intermediate state root). The status quo was to rather apply updates to trie nodes individually as state changes occurred during transaction execution. This may be hard to believe for node developers of today, but hey, it was 2016, and it worked - and it gave our Ethereum nodes a significant boost in performance.

We've come a long way since 2016, but many of the inefficiencies of the 16-radix Merkle-Patricia trie used in Ethereum and the Polkadot-SDK still persist. They have minor differences in node encoding and formats, but function much the same. The radix of 16 was chosen because it reduced one of the biggest problems with traversing Merkle Tries: random accesses. Child nodes are referenced by their hash, and these hashes are randomly distributed. If your traversal algorithm is naive and you load each child node as you learn its hash, you end up breaking one of the first laws of computer program optimization, which is to maintain data locality. What's even worse is breaking that law in disk access patterns. 16-radix Merkle-Patricia Tries alleviate this issue but definitely do not solve it. State Merkle Tries in Ethereum, Polkadot-SDK, and countless other protocols still work this way today, all with optimizations.

One of the other issues with the 16-radix Merkle-Patricia trie is that it's very space inefficient. All else equal, the 16-trie is more efficient than its binary counterpart in the number of disk accesses that need to be made. Proving the access of a key involves sending the hashes of the up to 15 other children at every visited branch. Binary trie proofs only involve one sibling, so all the extras are additional overhead. In a world of light and stateless blockchain nodes where Merkle proofs need to be submitted over the network, sending all these extra sibling hashes is highly wasteful. 

There have been advancements, such as the [Jellyfish Merkle Trie](https://developers.diem.com/papers/jellyfish-merkle-tree/2021-01-14.pdf?ref=127.0.0.1) pioneered at Diem. To summarize briefly - Jellyfish is pretty clever, and allows the same trie to be represented either in a binary format (over the network), or in a 16-radix format (on disk). It has other optimizations which aim to replace common sequence patterns of nodes with very efficient representations to minimize the required steps in traversal, update, and proving. They also remark on an approach to storing trie nodes on disk which avoids as much write amplification (read: overhead) in a RocksDB-based implementation. Jellyfish is definitely an improvement, but it also makes a key trade-off: the assumption that the keys used in the state trie have a fixed length.

We'll address the subject of fixed-length keys in more detail later, but it's the root of the differences between Preston's assumptions and the ones we'll be working with in this post. This difference in some sense is the real subject of this article. The Polkadot-SDK has taken an alternative path down the "tech tree" stemming from the Merkle-Patricia Trie of Ethereum. While Ethereum only ever uses fixed-length keys in its state trie, the underlying data structure actually supports arbitrary-length keys by virtue of the branch node carrying an optional value. Polkadot-SDK takes full advantage of this property and may actually be the only system to do so.

------

It's now time to visit the optimizations and differing assumptions, and modifications that might be made to apply these same optimizations (in spirit) to the Polkadot-SDK.

Preston's post makes a few implicit assumptions which I would like to make explicit here.

1. **Keys have a fixed length**. As mentioned above, this is a big one. 
1. **Stored keys are close to uniformly distributed across the key space**. This is also important, as it implies that the state trie has a roughly uniform depth at all points and therefore guesses about how long a traversal will be are likely to be accurate across the whole state trie.
1. **Only one version of the trie needs to be stored on disk**. I view this assumption as an implication of Proof-of-Stake, where fast (if not single-slot) finality has become the norm. In particular, this assumption makes sense for Sovereign - they target Celestia as one of their main platforms, and Celestia has single-slot finality.

Polkadot-SDK has made different decisions - in some cases, slightly different, in others, wildly different. Respectively:

1. **Keys do not have a fixed length**. The storage API exposed to the business logic of a Polkadot-SDK chain is a simple mapping from arbitrary byte-strings to arbitrary byte-strings. Keys are not meant to be hashes, though they are allowed to be.
2. **Keys often have long shared prefixes**. Chains built with the Polkadot-SDK are comprised of **modules**. All storage from any particular module has a shared prefix - and all storage from any particular map within a particular module also has a shared prefix. The set of these shared prefixes is relatively small. More on why in a moment.
3. **Only one version of the trie needs to be stored on disk, but finality is not instant**. This is a small difference and it's fairly trivially addressable with in-memory overlays for any unfinalized blocks, but does need to be handled.

Keys not having a fixed length and having long shared prefixes is a huge difference! There is a good reason for this: not having keys be uniformly-distributed enables the tree to be _semantically iterable_. In Polkadot-SDK, you can iterate the entire storage for a module, or for a particular mapping within a module. This is a key part of what enables Polkadot-SDK's to support trustless upgrades: you can deploy migrations that automatically iterate and migrate (or delete) entire swathes of storage without needing anyone to freeze the chain, generate a list of keys to alter or delete, and so on. State systems where the keys for related storage entries are dispersed throughout a forest of other, unrelated keys cannot provide any automated migration mechanisms. This is a very intentional product decision, but it makes Preston's proposal incompatible in a couple ways.

TODO: diagram - uniform distribution vs shared prefixes

-----

Without going into the details of the original approach yet (and I'd encourage having that article open as a reference), I'll lay out two of the properties that are crucial to making it fast. The first is that nodes have _extremely compact_ and _consistent_ representations on disk. The second is that all of the information needed to update the trie can be loaded by simply fetching the data needed to query the changed keys.

Each node has a representation which occupies only 32 bytes: it's a hash, with the first bit taken as a "domain separator" to indicate whether it's a branch or a leaf. Nodes are stored in fixed-size groups that cover predictable parts of the key space as a means to optimize SSDs. Nodes are stored in pages of 126 nodes, with 32 bytes for each node and 32 bytes for the page's unique identifier, for a total size of 4064 bytes. This is just 32 bytes shy of 4096 bytes - many, though not all, SSDs work on 4096-byte pages, so this maps very well onto the physical layout of SSDs. Since the pages needed are predictable from the key itself, all of these pages can be pre-fetched from an SSD in parallel and then traversed. No hopping around.

![](/assets/images/merklization_diagrams/page_1.svg)

This diagram shows a scaled-down version of the page, but the property of having 64 bytes over holds for all N.

When keys are fixed-length, value-carrying nodes can never have children - they are always leaves. Therefore, to query a value stored under a key, you must load all the nodes leading up to that key. Due to the page structure, this also implies loading that node's siblings, as well as all the sibling nodes along the path. Having the path and all the sibling nodes to a key, or set of keys, is all the information that is needed to update a binary Merkle trie. 

One problem that arises in binary merkle tries due to long shared prefixes is long traversals, assuming that your only two kinds of nodes are leaf nodes and branch nodes. Many Polkadot-SDK keys start with 256-bit shared prefixes - with these usage patterns, you'd traverse through 256 layers of the binary trie before even getting to a differentiated part. Luckily, Ethereum and Polkadot-SDK have already worked around this issue with an approach known as **extension nodes**. These nodes encode a long shared run of bits which all descendants of the node contain as part of their path. These work slightly differently in Ethereum and Polkadot-SDK, but achieve the same effect. Systems like Jellyfish have gotten rid of them entirely, because if your keys are uniformly distributed the odds of having long shared prefixes in a crowded trie are pretty small. But in the Polkadot-SDK model they still make sense. 

These properties to uphold, assumptions to relax, and usage patterns to support let us finally arrive to a sketch of the solution. First, we will turn our variable-length keys into fixed-length keys with a **uniform and logically large size** with an **efficient padding mechanism**. Second, we will **introduce extension nodes without substantially increasing disk accesses**.

------

### Padding Bounded-Length Keys to Fixed-Length Lookup Paths 

Turning variable-length keys into fixed-length lookup paths in the general case is impossible. However, if we can assume that all keys we use are less than some fixed length, then this problem is tractable. 

Storage keys in Polkadot-SDK are often longer than 256 bits. They are definitely less than 2^32 bits long, and in all likelihood always less than 2^12 bits long. Let's assume some generic upper bound 2^N and further assume that all keys used have length at most 2^N - N bits. 

Our goal is to create an efficient padding scheme that pads bit-strings that have length at most 2^N - N into unique bit-strings that are exactly length 2^N. We want to do this while preserving the initial key and only appending.

We can do this with the following algorithm:
  1. Take the original key and append `0`s to it until its length is divisible by N
  2. Append the length of the original key represented as an N-bit number
  3. Append `0`s to it until its length equals 2^N

TODO: diagram

This mapping gives us 2 desirable properties:
  1. No ambiguity. There are no 2 inputs which give the same output.
  2. The input key is kept as a prefix of the generated one. This preserves the shared prefixes in the original input keys that allow for iteration under shared prefixes.

However, it does not preserve a strict lexicographic ordering. It is almost perfect, but the lexicographic order when one key is a prefix of another is not preserved. If there are two keys, A and B, where A is a prefix of B, it is possible that pad(B) < pad(A). For example, with N=4 `111` would be padded to `1110_00110_0000_0000` and this will be sorted after the padded version of `11100000`, which is `1110_0000_1000_0000`.

If we were to put the length at the very end of the padding instead of directly after the initial key, we'd have preserved a full lexicographic ordering. But it would also lead to pairs of keys like `111` and `1110` being converted to lookup paths with extremely long shared prefixes - implying longer traversals. In practice this might be acceptable, as Polkadot-SDK doesn't produce these types of pairs storage keys. But it'd have very bad worst-case scenario, even with extension nodes.

While we are then dealing with a trie which in theory has 2^N layers, we will in practice never have keys or traversals anywhere near that long and will encounter leaf nodes much earlier in our traversal.

The structure of a leaf node in the original proposal and this modified version is exactly the same, with one semantic difference. Our data structure maps bounded-length keys to fixed-length lookup paths. Since the key length is not fixed, the encoding of our structure has a variable length. The last 32 bytes are the hash of the stored value, and the preceding bytes are the key itself.

```rust
struct Leaf { key: [u8], value_hash: [u8; 32] }
```

There would be no point in actually constructing the extremely long padded key in memory or hashing it, as our padding mechanism introduces no new entropy and simply maps keys from smaller spaces onto a larger one. They are only lookup paths, not logical keys.

Note that while it is theoretically possible to invert the mapping and go from one of our padded strings to its shorter representation, this is computationally intensive. So there is one other downside to this approach: if you give someone a path to a leaf node, but don't provide the leaf node or original key, it's hard to know which key this is proves membership of in the state trie. I don't believe this is a major issue, as it's more typical to prove to someone which value is stored under a key rather than prove that a value is stored for this key. If that's needed, you can just provide the original key along with the nodes along the longer padded lookup path.

------

### Introducing Extension Nodes

To handle the case of long shared prefixes in storage keys, we will introduce extension nodes which encode long partial lookup paths.

The first challenge to solve in introducing extension nodes is to add a third kind of node, beyond branches and leaves. Most trie implementations encode the type of node with a discriminant preceding the encoded value of the node, by using code that looks like this:

```rust
enum Node {
    #[index = "0"]
    Branch(left_child, right_child),
    #[index = "1"]
    Leaf(key, value),
    //...
}
```

Preston's proposal implements this differently: the type of node is encoded by its hash rather than the value. One bit is taken from the beginning of the hash - if it's a `0`, the node referenced by the hash is a branch. If it's a `1`, the node referenced by the hash is a leaf. This is referred to as a "domain separation".

Since the hash function we'd use in the trie is cryptographic, taking 1 bit from a 256-bit hash for domain separation doesn't meaningfully impact security. To domain-separate out 3 (or 4) kinds of nodes, we'd need to take 2 bits from the hash function output. This also does not impact security meaningfully.

So to add a 3rd kind of node, we extend the domain separation to 2 bits with the following schema:
  1. If the hash starts with `00` the node is a branch.
  2. If the hash starts with `01` the node is a leaf.
  3. If the hash starts with `10` the node is an extension.
  4. The hash cannot legally start with `11`.

But what is an extension node? Its logical structure will be this:

```rust
struct Extension {
    partial_path: [u8; 64],
    child_1: [u8; 32],
    child_2: [u8; 32],
}
```

The partial path encodes some part of the lookup path which is shared by all of its descendant nodes. It is laid out like this, bitwise:
```
000000101|01101000000...
```
The first 9 bits encode a length from 0 to 511. The next 503 bits encode the partial key, right-padded with 0s. Partial key lengths of 0, 1, or any value above 503 are disallowed for obvious reasons: 0 is impossible, 1 should just be a branch, and we don't have space for anything more than 503 bits.

Extensions are logically branches and have two children. It would make no sense for an extension to be followed by a single leaf. Such a structure would be more efficiently represented as a single leaf node. Therefore extensions must implicitly be branches. As a side note: this is actually a major difference between the Merkle Patricia Trie in Ethereum versus the one used in Polkadot-SDK: in Ethereum extension node format allows it to be followed by a leaf, though it is never in practice. In Polkadot-SDK it is illegal.

The next problem to solve is how to store the data of an extension node in our page structure. The child hashes of normal branches are stored directly "below" the branch node itself - either in the next layer of the current page or in the first layer of the next page, if the branch node is at the bottom of its page. We can adopt a similar strategy: the child hashes of the extension node will be stored in the page, and the position of the page, implied by the path to the child nodes. As a result, we may have empty pages between the extension node and its children. Importantly, the storage of descendents of the extension node will never be affected by modifications to or the removal of the extension node.

We still need to store that 64-byte partial path somewhere. It turns out we can store it directly under the extension node. Because extensions with 1 bit are illegal, and the children of the extension are stored in the page implied by their partial key, the slots for storing the nodes directly beneath the extension will be empty. We can use these two 32-byte slots to store the 64-byte partial key. When the extension node itself is at the bottom layer of the page, the partial key will be stored on the next page. There are possible fancier schemes to avoid this, such as finding empty locations within the current page to squirrel away the partial key - spaces under leaves and empty-subtrie markers can be overwritten. But that adds loads of complexity for little real gain: when the difference in the amount of pages we need to traverse is 1, what matters is whether we can pre-fetch the required pages from the SSD more so than how many pages we load. Modern SSDs are good at parallel reading.

TODO: diagram (storing extension nodes, partial key, and children)

The last issue is related: the fact that extensions can introduce gaps in the necessary pages to load could be a huge problem. An extension node located in a page of depth 2, which encodes a partial path of length 60, would land the child nodes of the extension squarely in page 12. We can no longer just pre-fetch the first N pages as computed from the key's lookup path and expect to find a terminal there. This is a general issue which would be a showstopper if not for the practical workload that the Polkadot-SDK imposes on the trie: there are relatively few shared prefixes, so we can just cache the "page paths" for all of them.

A Polkadot-SDK runtime might have 50 pallets (modules), which each have 10 different storage maps or values, for a total of ~550 common shared prefixes. When there are only a few hundred or even a few thousand shared storage prefixes, it's pretty trivial to keep an in-memory cache which tells us which pages you need to load to traverse to the end of some common prefix. Caching becomes intractable somewhere in the millions of shared prefixes. Then you can also run a simple pre-processsing on every queried key to see which shared prefixes match, if any. For the Polkadot-SDK workload, this will keep our SSD page-loads perfectly predictable just from the key. For a smart contract workload it would be better to incorporate a "child trie" approach, where each smart contract has its own state trie.