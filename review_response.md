## 1. Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?

## 2. Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?

## 3. How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
Our paper does not describe reclamation on the PFS, and the CASCADE metadata already contains the information that reclamation requires. Aggregated File Writer appends evicted blocks to aggregate files, and Location Metadata records the tier and offset of each block. When Eviction Manager purges a block, CASCADE invalidates the entry of the block in Global Shard Index. From Global Shard Index, CASCADE can count the valid entries of each aggregate file and delete the file once the count drops to zero. A deletion is one metadata operation and moves no data. In contrast, compaction rewrites valid entries and adds PFS traffic. Thus, per-file deletion can reclaim obsolete data without extra storage traffic, and we will add per-file deletion in the revision.

## 4. How does cross-machine coordination handle server failures or dynamic scaling?

Our paper does not describe node failures or dynamic scaling, and the design of Global Shard Index can support both without a central service. Every node stores an eventually consistent replica of Global Shard Index, and each Location Metadata entry records the node that stores the block. After a node fails, the surviving nodes can shrink the MPI communicator and remove the entries of the failed node from their replicas. The removed blocks then become cache misses, which CASCADE can recompute through the same path as stale location metadata. Since KV cache is reconstructible, a node failure discards only cached blocks and requires no data recovery.
Dynamic scaling can follow the same metadata path. A new node can receive the current replica of Global Shard Index through one Metadata Sync and then join the periodic synchronization. Since Location Metadata records the location of each block explicitly, every node can recompute the shard mapping for the new GPU count locally without moving stored blocks. Before a node leaves, Eviction Manager can move the protected prefix blocks of the node to the PFS through Aggregated File Writer, and the other nodes can then read the blocks from the PFS. Thus, Global Shard Index can support node failures and dynamic scaling without a central service, and we will add both to CASCADE in the revision.

## 5. Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment

Yes, salting is optional and set per request. Since each Block ID chains from the first block of a request, a salt in the hash of the first block changes every later Block ID of the request. As a result, requests with different salts never share a block, and the TTFT of one tenant does not depend on the requests of other tenants. Within one salt, Global Shard Index and Dedup Map operate unchanged and deduplicate the shared prefixes of the tenant. A single-user allocation disables salting to maximize deduplication, and a multi-tenant service assigns one salt per tenant. Our vLLM integration already supports per-tenant salts, because the Block ID chain includes the per-request cache salt of vLLM.

---

## 6. Please respond to other questions posted in the reviews.

**Global Shard Index and Dedup Map.**

**Concurrency and consistency.**

**Workload and scope.**

**Statistics and presentation.**
