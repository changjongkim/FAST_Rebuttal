## 1. Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?

## 2. Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?

## 3. How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
CASCADE reuses a prefix up to the last complete block before divergence. Each Block ID is a SHA-256 hash of the tokens of a block and the preceding Block ID. As a result, only blocks with identical preceding contexts share a Block ID, and the tokens after the last shared block are recomputed. The reuse granularity follows the block size. A smaller block size provides finer reuse at the cost of more metadata entries, and each Location Metadata entry requires only 80 bytes.

CASCADE already invalidates the Global Shard Index entry of each block that Eviction Manager purges, and these invalidations can support per-file reclamation on the PFS. For each block on the PFS, CASCADE also records the aggregate file and the offset. From the valid entries, CASCADE can count the valid blocks of each aggregate file and delete a file once the count drops to zero. File deletion requires no rewriting of live blocks, in contrast to compaction. Space in a partially valid file remains until the last valid block of the file is invalidated, and only compaction reclaims that space earlier. The current prototype does not maintain per-file counts, and we will add per-file deletion and clarify the distinction in the revision.

## 4. How does cross-machine coordination handle server failures or dynamic scaling?

Our paper targets batch-scheduled HPC systems, where a job runs on the fixed node set of an allocation. On such systems, the scheduler terminates the whole job by default when one of the allocated nodes fails, and a different node count requires a new allocation. Thus, our paper did not consider node failures or dynamic scaling within a job. However, the decentralized Global Shard Index of CASCADE can support both without a central metadata service.

Each Location Metadata entry identifies the node that stores a block. After communicator recovery, the surviving nodes can invalidate the entries that reference a failed node. Later lookups of the invalidated blocks become cache misses, and CASCADE recomputes the KV blocks from their token sequences. With reconstructed MPI RMA windows, the surviving nodes can then continue inference with the remaining cache.

For dynamic scaling, a joining node can receive the current Global Shard Index and then take part in metadata synchronization and shard remapping. Since Location Metadata records block locations explicitly, a change of shard ownership moves no stored block. A departing node can migrate its reusable blocks to the PFS before the departure, and the other nodes can then read the blocks from the PFS. Both paths additionally require a reconfiguration of the MPI communicator and the RMA windows. Thus, CASCADE can apply both mechanisms through Global Shard Index, and we will describe the design in the revision.

## 5. Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment

Yes, salting is optional and set per request. Since each Block ID chains from the first block of a request, a salt in the hash of the first block changes every later Block ID of the request. As a result, requests with different salts never share a block, and the TTFT of one tenant does not depend on the requests of other tenants. Within one salt, Global Shard Index and Dedup Map operate unchanged and deduplicate the shared prefixes of the tenant. A single-user allocation disables salting to maximize deduplication, and a multi-tenant service assigns one salt per tenant. Our vLLM integration already supports per-tenant salts, because the Block ID chain includes the per-request cache salt of vLLM.

---

## 6. Please respond to other questions posted in the reviews.

**Global Shard Index and Dedup Map.**

**Concurrency and consistency.**

**Workload and scope.**

**Statistics and presentation.**
