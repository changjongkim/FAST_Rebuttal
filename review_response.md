- **Q1.** Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?
- **Q2.** Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?
- **Q3.** How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
- **Q4.** How does cross-machine coordination handle server failures or dynamic scaling?
- **Q5.** Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment
- **Q6.** Please respond to other questions posted in the reviews.

---
#### Q1-Ans)

**Networking.** RDMA accelerates CASCADE's retrieval, but CASCADE retains its TTFT advantage over TCP peers. We run CASCADE with MPI over TCP on 2–32 nodes (8–128 GPUs) using Figure 14(c)'s 64K-prefix weak scaling. CASCADE's TTFT is 1.23–2.18s versus Mooncake's 1.37–25.94s. At 32 nodes, CASCADE achieves 11.9×/6.6× speedups over Mooncake/LMCache (P2P). Unlike the peers' centralized coordination, CASCADE resolves locations locally through Global Shard Index and reuses blocks through Shadow Copy Buffer. In Figure 7's trace-driven workload, uncompressed 160MB retrieval takes 63–71ms over TCP or 19–22ms over RDMA, versus 211–218ms for LMCache (Disk)/PDC.

**Compression.** INT8 trades retrieval latency for capacity. On 32 nodes, RDMA retrieval of a 320MB block takes 100ms with INT8 versus 38ms without it. INT8 halves each block's DRAM footprint, retaining more prefixes before Lustre eviction.

**Block size.** Replacing 16-token vLLM pages with 256-token blocks reduces CASCADE-TCP TTFT from 1.23–2.18s to 0.97–0.99s on 2–32 nodes. A 64K prefix then spans 16× fewer blocks. At 8 nodes with 256-token blocks and matched compression settings, per-node cache capacity, and routing, CASCADE achieves 9.5×/4.1× speedups over Mooncake/LMCache (P2P). All three reuse about 99% of prompt tokens. Separately, uncompressed retrieval rises from 5ms at 40MB to 38ms at 320MB.

#### Q2-Ans)

Per-layer INT8, with one scale and zero point per layer, does not lower output quality. On 280 answers to 160 questions from six English LongBench QA tasks, F1 is 40.0 for CASCADE with INT8, 39.5 without compression, and 40.2 for vLLM GPU. Per-layer INT8 changes 7% of vLLM GPU answers, against 8% between its repeated runs. In contrast, one scale per block lowers the F1 score to 38.9 and changes 28% of the answers.

Decoding reads FP16 blocks in GPU HBM, since CASCADE compresses only copies outside HBM. With four concurrent requests per node, 64K prefixes, and 128 output tokens, CASCADE takes 48-50ms per token against 53ms for vLLM GPU. As CASCADE retrieves 99.9% of prompt tokens, its TTFT is 14-17s against 55-57s, and its full request time is 20-22s against 62-64s.

#### Q3-Ans)

Since each BlockID hashes its tokens with the preceding BlockID, CASCADE reuses a prefix up to the last complete block before divergence. With 16-token blocks, at most 15 shared tokens are recomputed.

CASCADE can delete a Lustre aggregate file once Metadata Sync propagates invalidations of all its blocks. For a partially valid file, Eviction Manager can be extended to count valid blocks and invalidate them once the count decreases below a threshold. Since invalidated blocks only become misses, file deletion copies no block, at the cost of later recomputation.

#### Q4-Ans)

CASCADE targets fixed HPC allocations, but its decentralized Global Shard Index can support node failures and dynamic scaling. With a fault-tolerant MPI runtime, the remaining nodes can rebuild the communicator and RDMA Windows and invalidate entries of the failed node, whose blocks CASCADE recomputes. A new node can receive Global Shard Index and join Metadata Sync without moving stored blocks.

#### Q5-Ans)

Salting is optional and set per request by a trusted serving layer, and our single-tenant performance runs use no salt. A secret salt per tenant changes the first Block ID and every chained one, so TTFT does not reveal whether another tenant used the same prompt. We run four concurrent requests per node with 64K prefixes and one output token on 2-16 nodes. Requests with a shared salt reuse 99.9% of prompt tokens at a TTFT of 7.5-9.9s. In contrast, requests with distinct tenant salts reuse no prefix at a TTFT of 48.4-51.2s, no faster than recomputation.


#### Q6-1-Ans) Global Shard Index (RA, RB, RD)

**Replicas and ownership.** Every node resolves block locations in its full, nonpersistent replica of Global Shard Index in host DRAM, which covers GPU HBM, DRAM, and Lustre. Only the node storing a block writes its entry, which Metadata Sync propagates to every replica. In Figure 4, Node 2 then resolves the location of a block stored on another node in its own replica.

**Shards.** A Block ID hash selects a GPU and then its shard. Independent shards allow concurrent lookups and index merges without a single global lock.

**Deduplication.** Dedup Map records whether a block exists from its PUT until its full eviction, independent of tier movement. Its boolean flag takes one byte per block, against 80 bytes for a location entry. In E2E serving, global deduplication avoids 57-66% of block writes in Table 3. Each Block ID also hashes the model, LoRA adapter, KV dtype/layout, tensor-parallel rank/size, and block size.
#### Q6-2-Ans) Consistency and metadata scaling (RA, RB)

**Consistency.** CASCADE treats KV blocks as recomputable, so it enforces correctness strictly and treats availability as best effort. Since concurrent PUTs of one Block ID produce equivalent copies, Metadata Sync accepts any entry without consensus. A node publishes a location only after storing the block and invalidates only entries that name it. Each RDMA READ is validated against the requested Block ID and its recorded checksum. A failed validation triggers recomputation, never the reuse of a wrong block.


**Metadata scaling.** On 2-32 nodes, Metadata Sync takes 0.7-1.6µs per merged delta, including 0.1-0.8µs for Dedup Map. In a 4-node stress test, batches of 10K and 100K new 4KB blocks per node take at most 41ms and 569ms. At 100K blocks per second per node, the batches equal 100ms and 1000ms intervals. E2E runs create fewer than 100 blocks per node per interval. Before each sync, reads of newly created remote blocks miss, and all 72,000 reads after it hit without Block ID mismatches.

#### Q6-3-Ans) Workload and statistics (RA, RB, RD)

We route each E2E request to a prefix cached on another node, so node-local vLLM CPU offload misses in Figure 16. In Figure 12b, shared prefixes are 10% of a 10.2TB footprint, and CASCADE retains 99.4% of prefix blocks against 45.5-85.8% for the baselines. Over three or more E2E runs, the TTFT of CASCADE and Mooncake has a coefficient of variation of at most 11%.
