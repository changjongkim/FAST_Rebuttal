#### Q1-Ans)

For transport, we ran CASCADE with MPI over TCP, the transport of the peer baselines. In the setup and TTFT of Figure 14 with 64K prefixes, CASCADE over TCP achieved 8.6× and 3.7× speedups over Mooncake and LMCache (P2P) on 8 nodes. Its TTFT remained 1.09s to 1.12s on 2 to 8 nodes, while the TTFT of Mooncake increased from 1.37s to 9.31s. This gap occurs because the peers locate each block through centralized coordination, whereas CASCADE resolves locations in its local Global Shard Index replica and reuses retrieved blocks from its Shadow Copy Buffer (Section 3.3). The block size does not explain the gap, since all three systems reused about 99% of prompt tokens with 16-token pages in CASCADE and 256-token chunks in the peers. In the trace-driven evaluation of Figure 7, CASCADE over TCP also retrieved 160MB blocks in 63ms to 74ms, against 211ms to 218ms for LMCache (Disk) and PDC. Over RDMA, the same retrieval took 38ms to 49ms. Thus, CASCADE outperforms the peers without RDMA, and RDMA further reduces the retrieval latency.

For compression, disabling INT8 lowered retrieval latency at every block size and node count, from 47ms to 19ms for 160MB blocks on 4 nodes. As a result, the trace-driven gains do not depend on compression. The benefit of INT8 is capacity, since it reduces the DRAM footprint of each block copy to half of FP16 (Section 3.3). Without compression, retrieval latency increased in proportion to block size, from 5ms for 40MB blocks to 38ms for 320MB blocks.

#### Q2-Ans)

CASCADE compresses only the block copies outside GPU HBM, and decoding reads decompressed FP16 blocks in GPU HBM (Section 3.3). Each INT8 copy uses one scale and zero point per layer for keys and for values, which bounds the error of each value to 1/510 of its range. With this quantization, CASCADE scored an F1 of 40.0 on 280 questions from six English LongBench question answering tasks over 2 to 8 nodes, against 39.5 without compression and 40.2 for vLLM GPU. One scale per block, in contrast, lowered the F1 score to 38.9 and changed 28% of the answers of vLLM GPU, against 7% with per-layer INT8 and 8% between repeated runs of vLLM GPU. For token generation with four concurrent requests per node, 64K prefixes, and 128 output tokens, CASCADE took 48ms to 50ms per token against 53ms for vLLM GPU. Since CASCADE retrieved 99.9% of the prompt tokens, its TTFT was 14s to 17s against 55s to 57s, and the full request time was 20s to 22s against 62s to 64s.

#### Q3-Ans)

CASCADE reuses a prefix up to the last complete block before divergence, since each Block ID hashes the tokens of a block with the preceding Block ID. The block size sets this granularity, with 512 and 1024 KV pairs in the trace-driven evaluation and one 16-token page in our vLLM integration, which recomputes at most 15 shared tokens at a divergence.

For reclamation on Lustre, Eviction Manager invalidates the Global Shard Index entry of each purged block. Once all blocks of an aggregate file are invalidated and Metadata Sync has propagated the invalidations, CASCADE can delete the file without rewriting any valid block. The current prototype does not count valid blocks per file.

#### Q4-Ans)

Since CASCADE targets LLM inference on HPC allocations with fixed nodes, our paper did not consider node failures or dynamic scaling. Its decentralized Global Shard Index, however, can support both without a central metadata service. After a node failure, the remaining nodes can rebuild the communicator and RDMA Windows if the MPI runtime supports communicator recovery, and then invalidate the entries of the failed node. Lookups of those blocks become cache misses, and CASCADE recomputes them. For dynamic scaling, a new node receives the current Global Shard Index and joins Metadata Sync. Since Location Metadata records the location of each block, a new mapping of Block IDs to shards rebuilds only the index and moves no stored block.

#### Q5-Ans)

Salting is optional in CASCADE and set per request. A salt in the first block changes every chained Block ID of the request, so requests with different salts share no cached prefix. With one shared salt and one output token, 64K runs on 2 to 8 nodes reused 99.9% of prompt tokens at a TTFT of 7.5s to 8.9s. Requests with another tenant's salt reused no prefix block at a TTFT of 48.4s to 49.7s, no faster than recomputation, which removes the timing channel. Our vLLM integration sets one salt per tenant through the cache salt of vLLM.

---

#### Q6-1-Ans) Global Shard Index (RA, RB, RD)

Every node stores a full replica of Global Shard Index in host DRAM and resolves block locations locally. The node that stores a block writes its entry, and Metadata Sync copies the entry to every replica. Within each replica, shards are logical partitions keyed by GPU ID, so lookups and Index Merger on different shards proceed in parallel. Dedup Map records only whether any copy of a block exists, which changes at creation and full eviction, whereas a Location Metadata entry changes at every block movement. As a result, the PUT check of Dedup Coordinator remains valid during block movement. Dedup Map adds about 0.3µs per merged delta to Metadata Sync, while global deduplication avoids 57 to 66% of block writes (Table 3).

#### Q6-2-Ans) Consistency and metadata scaling (RA, RB)

Since concurrent PUTs of the same Block ID produce equivalent copies, Metadata Sync needs no synchronous conflict resolution. A node publishes a location only after the block is stored in Shadow Copy Buffer (Algorithm 1), and an invalidation removes an entry only while the entry names the invalidating node. After an RDMA READ, the reader matches the Block ID stored with the block against the requested one and the payload against the checksum recorded for that Block ID (Section 3.3). A stale read either returns the requested block or, after a full or partial overwrite, fails validation, and CASCADE then recomputes the block. Thus, stale metadata reduces cache hits without affecting correctness. For metadata scaling, Metadata Sync took at most 41ms and 569ms in a stress test with 10K and 100K new 4KB blocks per node per interval on 4 nodes. At 100K blocks per second per node, these loads correspond to intervals of 100ms and 1000ms, so each sync finished within its interval. Across 2, 4, and 8 nodes, Metadata Sync took about 1µs per merged delta, mostly in local processing. Reads before each sync missed, and all 72,000 reads after it hit with 0 Block ID mismatches. Our 4-node end-to-end runs created fewer than 100 blocks per node per interval, under 1% of the 10K blocks of the stress test.

#### Q6-3-Ans) Workload and statistics (RA, RB, RD)

The end-to-end evaluation routes each request to a prefix cached on another node to isolate cross-node reuse. Over three or more runs, the TTFT of CASCADE and Mooncake had a coefficient of variation of at most 11%, small relative to their gap.
