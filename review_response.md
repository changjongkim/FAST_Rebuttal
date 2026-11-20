- **Q1.** Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?
- **Q2.** Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?
- **Q3.** How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
- **Q4.** How does cross-machine coordination handle server failures or dynamic scaling?
- **Q5.** Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment
- **Q6.** Please respond to other questions posted in the reviews.

---#### Q1-Ans)

**Networking.** RDMA accelerates retrieval, but CASCADE retains its TTFT advantage over TCP peers. We test CASCADE over MPI/TCP at 2–32 nodes (8–128 GPUs) with Figure 14(c)'s 64K-prefix weak scaling. CASCADE's TTFT is 1.23/1.58/2.02/1.94/2.18s versus Mooncake's 1.37/4.89/9.31/18.24/25.94s. The 32-node speedups are 11.9×/6.6× over Mooncake/LMCache (P2P). CASCADE resolves locations locally via Global Shard Index and reads Shadow Copy Buffers. In Figure 7, uncompressed 160MB retrieval takes 63–71ms over TCP or 19–22ms over RDMA, versus 211–218ms for LMCache (Disk)/PDC.

**Compression.** INT8 trades retrieval latency for capacity rather than accelerating individual reads. CASCADE decompresses DRAM shadows into FP16 for GPU HBM. Disabling INT8 lowers RDMA retrieval latency across sizes and nodes, from 100ms to 38ms for 320MB blocks on 32 nodes. INT8 doubles DRAM block capacity before Lustre eviction.

**Block size.** Replacing 16-token vLLM pages with 256-token blocks reduces CASCADE-TCP TTFT from 1.23–2.18s to 0.97–0.99s on 2–32 nodes. A 64K prefix then spans 16× fewer blocks. At 8 nodes with matched TCP, 256-token blocks, compression, cache capacity, and routing, CASCADE achieves 9.5×/4.1× speedups over Mooncake/LMCache (P2P). All three reuse about 99% of prompt tokens. Separately, uncompressed retrieval rises from 5ms at 40MB to 38ms at 320MB.

#### Q2-Ans)

CASCADE maintains measured QA quality with per-layer INT8 offload. Using one scale and zero point per layer for keys and values, F1 across 280 answers to 160 unique questions from six English LongBench v1 QA tasks is 40.0 with INT8, 39.5 uncompressed, and 40.2 with vLLM GPU. All GPU workers use INT8. In pass two, CASCADE retrieves 99.9% of prompt tokens. Per-layer INT8 changes 7% of vLLM GPU answers, against 8% between its repeated runs. Per-block scaling instead yields F1 38.9 and 28% answer disagreement.

Decoding reads FP16 blocks in HBM, as compression applies only outside HBM. With four concurrent requests/node, 64K prefixes, and 128 output tokens, CASCADE takes 48–50ms/token against 53ms for vLLM GPU. With 99.9% prompt reuse, CASCADE's TTFT is 14–17s against 55–57s and full request time is 20–22s against 62–64s.

#### Q3-Ans)

Each Block ID hashes its tokens with the preceding Block ID, allowing reuse until the last complete block before divergence. With 16-token blocks, at most 15 shared tokens are recomputed.

An aggregate Lustre file can be deleted once Metadata Sync propagates invalidations of all its blocks. For a partially valid file, Eviction Manager can be extended to count valid blocks and invalidate remaining blocks once the count falls below a threshold. This extension would reclaim the file without copying its remaining valid blocks, at the cost of possible recomputation on later requests.

#### Q4-Ans)

CASCADE targets fixed HPC allocations. Live recovery and dynamic scaling remain extensions. With fault-tolerant MPI, surviving nodes could retain KV blocks and index replicas, rebuild communicators and RDMA Windows, invalidate failed-node locations, and recompute missing blocks. A new node could receive Global Shard Index and join Metadata Sync after reconfiguration without moving stored blocks.

#### Q5-Ans)

Salting is optional and assigned per request by a trusted serving layer. Our single-tenant performance runs use no salt. A secret salt per tenant changes the first Block ID and every chained one, preventing cross-tenant prefix-cache hits and their TTFT timing channel. On 2–16 nodes, we run four concurrent requests/node with 64K prefixes and one output token. Requests with a shared salt reuse 99.9% of prompt tokens at a TTFT of 7.5–9.9s. Distinct tenant salts prevent reuse, yielding 48.4–51.2s TTFT, comparable to recomputation.

#### Q6-1-Ans) Global Shard Index (RA, RB, RD)

**Replicas and ownership.** Every node holds a full, nonpersistent Global Shard Index replica in host DRAM covering GPU HBM, DRAM, and Lustre. Metadata Sync replicates each storing node's location updates. In Figure 4, Node 2 resolves a remote block through its local replica.

**Shards.** A Block ID hash selects a GPU and then its shard. Shard-local read/write locks allow concurrent lookups and confine index merges to individual shards without a global lock.

**Deduplication.** Dedup Map tracks existence across tier moves and sets flags False on full eviction. Its boolean flag takes one byte per block, against 80 bytes for a location entry. Global deduplication avoids 57–66% of E2E writes (Table 3). Block IDs also hash model, LoRA adapter, KV dtype/layout, tensor-parallel rank/size, and block size.

#### Q6-2-Ans) Consistency and metadata scaling (RA, RB)

**Consistency.** CASCADE treats KV blocks as recomputable, prioritizing correctness over cache availability. Concurrent PUTs of one Block ID yield equivalent copies, so Metadata Sync may retain either location without consensus. A storing node publishes its location only after writing the block and invalidates it on eviction. Since delayed sync can expose stale locations, readers validate the requested Block ID and compressed-payload checksum after RDMA retrieval. A mismatch triggers a miss and recomputation rather than incorrect reuse.

**Metadata scaling.** On 2–32 nodes, Metadata Sync takes 0.7–1.6µs per merged delta, including 0.1–0.8µs for Dedup Map. In 4-node stress tests, one sync merges 10K/100K new 4KB blocks per node in at most 41/569ms. At an equivalent 100K blocks/s/node, these batches represent 100/1000ms of input. E2E runs create fewer than 100 blocks per node per interval. New remote blocks miss before sync. All 72,000 post-sync reads hit without Block ID mismatches.

#### Q6-3-Ans) Workload and statistics (RA, RB, RD)

E2E requests target prefixes cached on other nodes, so node-local vLLM CPU offload misses in Figure 16. In Figure 12b, shared prefixes are 10% of a 10.2TB footprint, and CASCADE retains 99.4% of prefix blocks against 45.5–85.8% for the baselines. Across at least three E2E runs, CASCADE and Mooncake TTFT coefficients of variation stay within 11%.
