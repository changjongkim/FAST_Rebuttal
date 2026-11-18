#### Q1-Ans)

For transport, we ran CASCADE with MPI over TCP, the transport of the peer baselines. In the setup and TTFT of Figure 14 with 64K prefixes, CASCADE over TCP achieved 8.6× and 3.7× speedups over Mooncake and LMCache (P2P) on 8 nodes, close to 7.7× and 3.3× over RDMA. Its TTFT remained 1.09s to 1.12s on 2 to 8 nodes, while Mooncake rose from 1.37s to 9.31s. The peers locate each block through centralized coordination, whereas CASCADE resolves locations in its local Global Shard Index replica and caches retrieved blocks in its Shadow Copy Buffer (Section 3.3). Although CASCADE used 16-token pages against the 256-token chunks of the peers, all three systems reused about 99% of prompt tokens. In the trace-driven evaluation of Figure 7, CASCADE retrieved 160MB blocks in 63ms to 74ms over TCP and 38ms to 49ms over RDMA, against 211ms to 218ms for LMCache (Disk) and PDC. Thus, the advantage of CASCADE persists without RDMA, and RDMA further shortens each retrieval.

For compression, disabling INT8 lowered retrieval latency at every block size and node count, from 47ms to 19ms for 160MB blocks on 4 nodes. As a result, the trace-driven gains do not depend on compression, which reduces the DRAM footprint of each block copy to half of FP16 so that more prefixes remain in DRAM (Section 3.3). Without compression, retrieval latency grew in proportion to block size, from 5ms for 40MB blocks to 38ms for 320MB blocks.

#### Q2-Ans)

CASCADE compresses only the block copies outside GPU HBM, so decoding reads decompressed FP16 blocks (Section 3.3). INT8 uses one scale and zero point per layer for keys and for values, which bounds the error of each value to 1/510 of its range. On 280 questions from six English LongBench question answering tasks over 2 to 8 nodes, CASCADE scored an F1 of 40.0 with INT8 and 39.5 without compression, against 40.2 for vLLM GPU. With one scale per block, the F1 score dropped to 38.9, and 28% of the answers differed from vLLM GPU against 7% with per-layer INT8 and 8% between repeated runs of vLLM GPU. For token generation with four concurrent requests per node, 64K prefixes, and 128 output tokens, CASCADE took 48ms to 50ms per token against 53ms for vLLM GPU. Since CASCADE retrieved 99.9% of the prompt tokens, its TTFT was 14s to 17s against 55s to 57s, and the full request time was 20s to 22s against 62s to 64s.

#### Q3-Ans)

CASCADE reuses a prefix up to the last complete block before divergence, since each Block ID hashes the tokens of a block with the preceding Block ID. The trace-driven evaluation uses blocks of 512 and 1024 KV pairs, but our vLLM integration uses one 16-token page per block and recomputes at most 15 shared tokens at a divergence.

On Lustre, Eviction Manager invalidates the Global Shard Index entry of each purged block. CASCADE can delete an aggregate file once all its blocks are invalidated and Metadata Sync has propagated the invalidations, which rewrites no valid block. The current prototype does not count valid blocks per file.

#### Q4-Ans)

CASCADE targets LLM inference on HPC allocations with fixed nodes, so our paper did not consider node failures or dynamic scaling, but Global Shard Index can support both. If the MPI runtime supports communicator recovery, the remaining nodes can rebuild the communicator and RDMA Windows after a node failure and invalidate the entries of the failed node, whose blocks CASCADE then recomputes. For dynamic scaling, a new node receives the current Global Shard Index and joins Metadata Sync. Since Location Metadata records the location of each block, a new mapping of Block IDs to shards rebuilds only the index and moves no stored block.

#### Q5-Ans)

Yes, salting is optional and set per request. A salt in the first block changes every chained Block ID of the request, so requests with different salts share no cached prefix. With one shared salt and one output token, 64K runs on 2 to 8 nodes reused 99.9% of prompt tokens at a TTFT of 7.5s to 8.9s. Requests with another tenant's salt reused no prefix block at a TTFT of 48.4s to 49.7s, no faster than recomputation, which removes the timing channel. Our vLLM integration sets one salt per tenant through the cache salt of vLLM.

---

#### Q6-1-Ans) Global Shard Index (RA, RB, RD)

Every node stores a full replica of Global Shard Index in host DRAM and resolves block locations locally. The node that stores a block writes its entry, and Metadata Sync copies the entry to every replica. Shards are logical partitions of each replica keyed by GPU ID, so lookups and Index Merger on different shards proceed in parallel. Dedup Map records only whether any copy of a block exists, which changes at creation and full eviction, whereas a Location Metadata entry changes at every block movement. As a result, the PUT check of Dedup Coordinator remains valid during block movement. Dedup Map adds about 0.3µs per merged delta to Metadata Sync and avoids 57 to 66% of block writes (Table 3).

#### Q6-2-Ans) Consistency and metadata scaling (RA, RB)

Concurrent PUTs of the same Block ID produce equivalent copies, so Metadata Sync needs no synchronous conflict resolution. A node publishes a location only after the block is stored in Shadow Copy Buffer (Algorithm 1). After an RDMA READ, the reader matches the Block ID stored with the block against the requested one and the payload against the checksum recorded for that Block ID (Section 3.3). A read of a replaced slot, even a partially overwritten one, fails validation, and CASCADE recomputes the block. Thus, stale metadata costs cache hits but not correctness. In a stress test with 4KB blocks on 4 nodes, Metadata Sync took at most 41ms and 569ms with 10K and 100K new blocks per node per interval. At 100K blocks per second per node, these correspond to intervals of 100ms and 1000ms, so each sync finished within its interval. Metadata Sync took about 1µs per merged delta on 2, 4, and 8 nodes, mostly in local processing. Reads before each sync missed, and all 72,000 reads after it hit with 0 Block ID mismatches. Our 4-node end-to-end runs created fewer than 100 blocks per node per interval, far below the stress test.

#### Q6-3-Ans) Workload and statistics (RA, RB, RD)

The end-to-end evaluation routes each request to a prefix cached on another node to isolate cross-node reuse. Over three or more runs, the TTFT of CASCADE and Mooncake had a coefficient of variation of at most 11%, small relative to their gap.
