- **Q1.** Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?
- **Q2.** Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?
- **Q3.** How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
- **Q4.** How does cross-machine coordination handle server failures or dynamic scaling?
- **Q5.** Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment
- **Q6.** Please respond to other questions posted in the reviews.

---

#### Q1-Ans)

For transport, we ran CASCADE with MPI over TCP, the transport of the peers, in the weak-scaling setup of Figure 14 with 64K prefixes. On 8 nodes, CASCADE over TCP achieved 8.6× and 3.7× TTFT speedups over Mooncake and LMCache (P2P). Its TTFT remained 1.09s to 1.12s on 2 to 8 nodes, while the TTFT of Mooncake increased from 1.37s to 9.31s. The peers locate each block through centralized coordination. In contrast, CASCADE resolves locations in its local Global Shard Index replica and reuses retrieved blocks from Shadow Copy Buffer (Section 3.3). The E2E runs stored each 16-token vLLM page as one CASCADE block, against the 256-token chunks of the peers. However, all three systems reused about 99% of prompt tokens, so the finer blocks did not increase the reuse rate. The trace-driven evaluation of Figure 7 uses 160MB blocks of 512 tokens. CASCADE over TCP retrieved these blocks in 63ms to 74ms, against 211ms to 218ms for LMCache (Disk) and PDC. Over RDMA, the same retrieval took 38ms to 49ms. Thus, CASCADE outperforms the peers without RDMA.

For compression, disabling INT8 lowered retrieval latency at every block size and node count, from 47ms to 19ms for 160MB blocks on 4 nodes. As a result, the trace-driven gains do not depend on compression. The benefit of INT8 is capacity, since an INT8 copy uses half of the DRAM of FP16 (Section 3.3). Without compression, retrieval latency increased in proportion to block size, from 5ms for 40MB blocks to 38ms for 320MB blocks.


#### Q2-Ans)

CASCADE compresses only block copies outside GPU HBM, so decoding reads FP16 blocks in GPU HBM (Section 3.3). Each INT8 copy uses one scale and zero point per layer for keys and for values. With this quantization, CASCADE scored an F1 of 40.0 on 280 questions from six English question answering tasks of LongBench. Without compression, CASCADE scored 39.5, and vLLM GPU scored 40.2. In contrast, one scale per block lowered the F1 score to 38.9 and changed 28% of the answers of vLLM GPU. Per-layer INT8 changed 7% of these answers, against 8% between repeated runs of vLLM GPU. For token generation with four concurrent requests per node, 64K prefixes, and 128 output tokens, CASCADE took 48ms to 50ms per token against 53ms for vLLM GPU. Since CASCADE retrieved 99.9% of the prompt tokens, its TTFT was 14s to 17s against 55s to 57s. As a result, its full request time was 20s to 22s against 62s to 64s.


#### Q3-Ans)

CASCADE reuses a prefix up to the last complete block before divergence, since each Block ID hashes its tokens with the preceding Block ID. With 16-token blocks, CASCADE recomputes at most 15 shared tokens at a divergence.

To reclaim obsolete data on Lustre, CASCADE can delete an aggregate file once Metadata Sync has propagated the invalidation of all its blocks. For a partially valid file, Eviction Manager can be extended to count its valid blocks. Once this count decreases below a threshold, Aggregated File Writer can append these blocks to a new aggregate file and delete the old one. Thus, reclaiming a partially valid file copies fewer blocks than the threshold.

#### Q4-Ans)

Since CASCADE targets HPC allocations with fixed nodes, our paper did not consider node failures or dynamic scaling. However, its decentralized Global Shard Index can support both. After a node failure, the remaining nodes can rebuild the communicator and RDMA Windows with a fault-tolerant MPI runtime. These nodes can then invalidate the entries of the failed node, whose blocks CASCADE recomputes. For dynamic scaling, a new node can receive the current Global Shard Index and join Metadata Sync without moving any stored block.

#### Q5-Ans)

Salting is optional in CASCADE and set per request by the serving layer, which can assign each tenant a secret salt. A salt in the first block changes every chained Block ID of the request, so requests with different salts share no cached prefix. We ran four concurrent requests per node with 64K prefixes and one output token on 2 to 8 nodes. Requests with one shared salt reused 99.9% of prompt tokens at a TTFT of 7.5s to 8.9s. In contrast, requests with another tenant's salt reused no prefix block at a TTFT of 48.4s to 49.7s. Since this TTFT was no faster than recomputation, it did not reveal whether another tenant had used the same prompt.

---

#### Q6-1-Ans) Global Shard Index (RA, RB, RD)

Every node resolves block locations in its full replica of Global Shard Index in host DRAM. This replica records every tier, including GPU HBM and Lustre, and is not persistent. The node that stores a block writes its entry, and Metadata Sync copies the entry to every replica. Within each replica, shards are logical partitions keyed by GPU ID, so lookups and Index Merger on different shards proceed in parallel. Since block movement does not change whether a copy exists, the PUT check on Dedup Map remains valid. Global deduplication avoids 57 to 66% of block writes (Table 3). Each Block ID also hashes the model, LoRA adapter, KV data type and layout, tensor-parallel rank and size, and block size.

#### Q6-2-Ans) Consistency and metadata scaling (RA, RB)

Since concurrent PUTs of the same Block ID produce equivalent copies, Metadata Sync can merge their entries in any order. A node publishes a location only after storing the block in Shadow Copy Buffer (Algorithm 1). Each invalidation removes an entry only while the entry names the invalidating node. After an RDMA READ, the reader validates the stored Block ID and payload checksum against the requested Block ID and its recorded checksum (Section 3.3). A stale read either returns the requested block or fails validation after any overwrite, and CASCADE then recomputes the block. Thus, stale metadata lowers the cache hit rate, but every request uses a validated or recomputed block. For metadata scaling, Metadata Sync took at most 41ms and 569ms in a 4-node stress test with 10K and 100K new 4KB blocks per node per interval. At 100K blocks per second per node, these loads correspond to intervals of 100ms and 1000ms. Across 2, 4, and 8 nodes, Metadata Sync took about 1µs per merged delta, including 0.3µs for Dedup Map. Reads before each sync missed, and all 72,000 reads after it hit with 0 Block ID mismatches. Our 4-node E2E runs created fewer than 100 blocks per node per interval, under 1% of the 10K blocks of the stress test.


#### Q6-3-Ans) Workload and statistics (RA, RB, RD)

The E2E runs route each request to a prefix cached on another node. Over three or more runs, the TTFT of CASCADE and Mooncake had a coefficient of variation of at most 11%.
