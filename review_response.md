- **Q1.** Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?
- **Q2.** Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?
- **Q3.** How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
- **Q4.** How does cross-machine coordination handle server failures or dynamic scaling?
- **Q5.** Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment
- **Q6.** Please respond to other questions posted in the reviews.

---

#### Q1-Ans)

**Networking.** CASCADE outperforms the peers without RDMA. We ran CASCADE with MPI over TCP, the transport of the peers, in the 64K-prefix weak scaling of Figure 14. On 8 nodes, CASCADE over TCP achieved 8.6× and 3.7× TTFT speedups over Mooncake and LMCache (P2P). Its TTFT remained 1.09-1.12s on 2-8 nodes, while that of Mooncake increased from 1.37s to 9.31s. The peers locate each block through centralized coordination. In contrast, CASCADE resolves locations in its local Global Shard Index replica. For the trace-driven evaluation of Figure 7, CASCADE over TCP without INT8 retrieved 160MB blocks in 63-71ms, against 211-218ms for LMCache (Disk) and PDC.

**Compression.** The trace-driven gains do not depend on compression. Over RDMA, disabling INT8 lowered retrieval latency at every block size and node count, from 47ms to 19ms for 160MB blocks on 4 nodes. The benefit of INT8 is capacity, since an INT8 copy uses half of the DRAM of FP16 (Section 3.3).

**Block size.** Without compression, trace-driven retrieval latency increased in proportion to block size, from 5ms for 40MB blocks to 38ms for 320MB blocks of 1024 tokens. With the 256-token blocks of the peers in place of 16-token vLLM pages, the 8-node speedups of CASCADE over TCP increased to 9.5× and 4.1×. All three systems reused about 99% of prompt tokens.


#### Q2-Ans)

**Quality.** Per-layer INT8, with one scale and zero point per layer for keys and for values, did not lower output quality. On 280 questions from six English LongBench question answering tasks, the F1 score was 40.0 for CASCADE with INT8, 39.5 without compression, and 40.2 for vLLM GPU. Per-layer INT8 changed 7% of the answers of vLLM GPU, against 8% between repeated runs of vLLM GPU. In contrast, one scale per block lowered the F1 score to 38.9 and changed 28% of the answers.

**Generation.** Since CASCADE compresses only block copies outside GPU HBM, decoding reads FP16 blocks in GPU HBM (Section 3.3). With four concurrent requests per node, 64K prefixes, and 128 output tokens, CASCADE took 48-50ms per token against 53ms for vLLM GPU. As CASCADE retrieved 99.9% of the prompt tokens, its full request time was 20-22s against 62-64s.


#### Q3-Ans)

**Partial chunks.** CASCADE reuses a prefix up to the last complete block before divergence, since each Block ID hashes its tokens with the preceding Block ID. With 16-token blocks, CASCADE recomputes at most 15 shared tokens at a divergence.

**Obsolete data.** CASCADE can delete an aggregate file on Lustre once Metadata Sync has propagated the invalidation of all its blocks. For a partially valid file, Eviction Manager can be extended to count the valid blocks and to invalidate them once the count decreases below a threshold. Since invalidated blocks only become misses, deleting the file adds no storage traffic.

#### Q4-Ans)

While our paper targets fixed HPC allocations, the decentralized Global Shard Index of CASCADE can support node failures and dynamic scaling.

**Node failures.** With a fault-tolerant MPI runtime, the remaining nodes can rebuild the communicator and RDMA Windows and invalidate the entries of the failed node, whose blocks CASCADE recomputes. In large-scale inference, durable copies would avoid the recomputation but require coordination that increases with node count.

**Dynamic scaling.** A new node can receive the current Global Shard Index and join Metadata Sync without moving any stored block.

#### Q5-Ans)

Salting is optional in CASCADE and set per request by the serving layer, which can assign each tenant a secret salt. With secret salts per tenant, TTFT does not reveal the prompts of another tenant. A salt in the first block changes every chained Block ID, so requests with different salts share no cached prefix. We ran four concurrent requests per node with 64K prefixes and one output token on 2-8 nodes. Requests with one shared salt reused 99.9% of prompt tokens at a TTFT of 7.5-8.9s. In contrast, requests with another tenant's salt reused no prefix at a TTFT of 48.4-49.7s, no faster than recomputation.

---

#### Q6-1-Ans) Global Shard Index (RA, RB, RD)

**Replica placement.** Every node resolves block locations in its full replica of Global Shard Index in host DRAM. The replica records every tier, including GPU HBM and Lustre, and is not persistent.

**Update ownership.** Only the node that stores a block writes its entry, and Metadata Sync copies the entry to every replica.

**Logical partitioning.** Within each replica, the hash of a Block ID selects a GPU ID and then its shard, one per GPU. Thus, under full replication, sharding replaces one global lock with one lock per shard, so Index Merger and lookups contend only within a shard.

**Deduplication.** Dedup Map records whether a block exists, from its PUT until its eviction from all nodes, while Global Shard Index records one location per block. As a result, tier movement updates only Global Shard Index, and the PUT check on Dedup Map includes blocks whose store is in progress. Each Block ID also hashes the model, LoRA adapter, KV data type and layout, tensor-parallel rank and size, and block size.

#### Q6-2-Ans) Consistency and metadata scaling (RA, RB)

Since inference can recompute any KV block from its tokens and model, CASCADE enforces correctness strictly and treats availability as best effort. A lost, stale, or overwritten block becomes a miss. Concurrent PUTs of the same Block ID produce equivalent copies, so Metadata Sync can accept any one of their entries without coordination. A node publishes a location only after storing the block in Shadow Copy Buffer (Algorithm 1) and invalidates only the entries that name it. After an RDMA READ, the reader validates the block against the requested Block ID and its recorded checksum (Section 3.3). Thus, a node reuses a buffer slot without waiting for outstanding RDMA READs. A read during or after the reuse either returns the requested block or fails validation and triggers recomputation.

Across 2, 4, and 8 nodes, Metadata Sync took about 1µs per merged delta, including 0.3µs for Dedup Map. In a 4-node stress test with 10K and 100K new 4KB blocks per node per interval, it took at most 41ms and 569ms. At 100K new blocks per second per node, the two loads correspond to intervals of 100ms and 1000ms. Reads before each sync missed, and all 72,000 reads after it hit with 0 Block ID mismatches. Our 4-node E2E runs created fewer than 100 blocks per node per interval, under 1% of the smaller stress load.


#### Q6-3-Ans) Workload and statistics (RA, RB, RD)

The E2E runs route each request to a prefix cached on another node, so the CPU cache of vLLM CPU offload misses the requested prefixes in Figure 16. Over three or more runs, the TTFT of CASCADE and Mooncake had a coefficient of variation of at most 11%.
