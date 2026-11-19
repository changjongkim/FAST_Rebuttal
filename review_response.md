- **Q1.** Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?
- **Q2.** Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?
- **Q3.** How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
- **Q4.** How does cross-machine coordination handle server failures or dynamic scaling?
- **Q5.** Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment
- **Q6.** Please respond to other questions posted in the reviews.

---

#### Q1-Ans)

**Networking.** CASCADE outperforms the peers without RDMA. We run CASCADE with MPI over TCP, the peers' transport, on 2-32 nodes (8-128 GPUs). With the 64K prefixes of Figure 14, its TTFT remains 1.23-2.18s, while that of Mooncake increases from 1.37s to 25.94s. On 32 nodes, CASCADE achieves 11.9× and 6.6× TTFT speedups over Mooncake and LMCache (P2P). CASCADE resolves locations in its local Global Shard Index replica, without the centralized coordination of the peers. In the trace-driven Figure 7, CASCADE over TCP without INT8 retrieves 160MB blocks in 63-71ms, against 211-218ms for LMCache (Disk) and PDC. Over RDMA, the same retrieval takes 19-22ms.

**Compression.** The trace-driven gains do not depend on compression. Over RDMA, disabling INT8 lowers retrieval latency at every block size and node count, from 100ms to 38ms for 320MB blocks on 32 nodes. The benefit of INT8 is capacity, since an INT8 copy uses half the DRAM of FP16.

**Block size.** Without compression, trace-driven retrieval latency increases in proportion to block size, from 5ms for 40MB to 38ms for 320MB blocks. With the peers' 256-token blocks in place of 16-token vLLM pages, the TTFT of CASCADE over TCP decreases from 1.23-2.18s to 0.97-0.99s on 2-32 nodes. CASCADE, Mooncake, and LMCache (P2P) all reuse about 99% of prompt tokens. On 8 nodes, CASCADE achieves 9.5× and 4.1× speedups over the two peers.


#### Q2-Ans)

Per-layer INT8, with one scale and zero point per layer, does not lower output quality. On 280 questions from six English LongBench QA tasks, the F1 score is 40.0 for CASCADE with INT8, 39.5 without compression, and 40.2 for vLLM GPU. Per-layer INT8 changes 7% of vLLM GPU answers, against 8% between its repeated runs. In contrast, one scale per block lowers the F1 score to 38.9 and changes 28% of the answers.

Decoding reads FP16 blocks in GPU HBM, since CASCADE compresses only copies outside HBM. With four concurrent requests per node, 64K prefixes, and 128 output tokens, CASCADE takes 48-50ms per token against 53ms for vLLM GPU. As CASCADE retrieves 99.9% of prompt tokens, its TTFT is 14-17s against 55-57s, and its full request time is 20-22s against 62-64s.


#### Q3-Ans)

Since each BlockID hashes its tokens with the preceding BlockID, CASCADE reuses a prefix up to the last complete block before divergence. With 16-token blocks, at most 15 shared tokens are recomputed.

CASCADE can delete a Lustre aggregate file once Metadata Sync propagates invalidations of all its blocks. For a partially valid file, Eviction Manager can be extended to count valid blocks and invalidate them once the count decreases below a threshold. Since invalidated blocks only become misses, file deletion copies no block, at the cost of later recomputation.

#### Q4-Ans)

CASCADE targets fixed HPC allocations, but its decentralized Global Shard Index can support node failures and dynamic scaling. With a fault-tolerant MPI runtime, the remaining nodes can rebuild the communicator and RDMA Windows and invalidate entries of the failed node, whose blocks CASCADE recomputes. A new node can receive Global Shard Index and join Metadata Sync without moving stored blocks.

#### Q5-Ans)

Salting is optional and set per request by the serving layer, which can assign each tenant a secret salt. With secret salts per tenant, TTFT does not reveal whether another tenant used the same prompt. A salt in the first block changes every chained BlockID, so requests with different salts share no cached prefix. We run four concurrent requests per node with 64K prefixes and one output token on 2-8 nodes. Requests with one shared salt reuse 99.9% of prompt tokens at a TTFT of 7.5-8.9s. In contrast, requests with another tenant's salt reuse no prefix at a TTFT of 48.4-49.7s, no faster than recomputation.



---

#### Q6-1-Ans) Global Shard Index (RA,RB,RD)

**Replica placement.** Every node resolves block locations in its full, non-persistent replica of Global Shard Index in host DRAM, which covers GPU HBM, DRAM, and Lustre.

**Update ownership.** Only the node storing a block writes its entry, which Metadata Sync copies to every replica.

**Logical partitioning.** Within each replica, a BlockID hash selects a GPUID and then its shard. Thus, under full replication, sharding replaces one global lock with per-shard locks, so Index Merger and lookups contend only within a shard.

**Deduplication.** Dedup Map records whether a block exists from its PUT until its full eviction, and Global Shard Index records one location per block. Thus, tier movement updates only Global Shard Index, and the PUT check covers unfinished stores. Each BlockID also hashes the model, LoRA adapter, KV dtype/layout, tensor-parallel rank/size, and block size.

#### Q6-2-Ans) Consistency and metadata scaling (RA,RB)

Since inference can recompute any KV block, CASCADE enforces correctness strictly and treats availability as best effort, and a lost or overwritten block becomes a miss. Concurrent PUTs of one BlockID produce equivalent copies, so Metadata Sync accepts any entry without consensus. A node publishes a location only after storing the block and invalidates only entries that name it. Each RDMA READ is validated against the requested BlockID and its recorded checksum. Thus, a node reuses a buffer slot without waiting for outstanding reads. Any read during or after the reuse returns the requested block or fails validation and triggers recomputation.

On 2-32 nodes, Metadata Sync takes 0.7-1.6µs per merged delta, including 0.1-0.8µs for Dedup Map. In a 4-node stress test, it takes at most 41ms and 569ms for 10K and 100K new 4KB blocks per node. At 100K blocks per second per node, the loads equal 100ms and 1000ms intervals. E2E runs create fewer than 100 blocks per node per interval. Reads before each sync miss, and all 72,000 reads after it hit without BlockID mismatches.



#### Q6-3-Ans) Workload and statistics (RA,RB,RD)

E2E requests use prefixes cached on other nodes, so vLLM CPU offload misses in Figure 16. With 10% sharing in Figure 12b, CASCADE retains 99.4% of prefix blocks, compared with 45.5- 85.8% for baselines. Over three or more runs, the TTFT coefficient of variation for CASCADE and Mooncake is at most 11%.
