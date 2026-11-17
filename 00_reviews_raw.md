# FAST '27 Fall, Paper #165 — Reviews (verbatim)

> Source: HotCRP page for #165, printed 2026-11-17 21:23 KST (`fast_review_cascade.pdf`).
> Title: *CASCADE: Content-Addressed Distributed KV Cache Storage for LLM Inference on Tiered Data Architectures*
> Submitted: Sep 15, 2026, 9:51:59 PM AoE (efd19275).
> Rebuttal response deadline: **72 hours from the snapshot**, i.e. about 2026-11-20 21:20 KST. Check the exact time on HotCRP.
> Transcription note: the PDF text extraction lists section headings before their contents. The order below follows the rendered page.
> Text is verbatim, including reviewer typos. Links are kept as written.

| Review | Overall merit | Reviewer expertise |
|---|---|---|
| #165A | 2. Weak reject | 3. Knowledgeable |
| #165B | 2. Weak reject | 4. Expert |
| #165C | 3. Weak accept (accept if room) | 3. Knowledgeable |
| #165D | 4. Accept | 2. Some familiarity |
| #165E | 3. Weak accept (accept if room) | 2. Some familiarity |

Plus one comment from Reviewer D (@A1), reproduced at the end. It sets the priority order for the response.

---

## Review #165A

**Overall merit:** 2. Weak reject  
**Reviewer expertise:** 3. Knowledgeable

### Paper summary
CASCADE coordinates GPU HBM, local and remote DRAM, and Lustre for distributed LLM KV caching on HPC platforms, targeting cross-node prefix reuse and data-access and storage-metadata overhead. Its control layer identifies blocks using chained token hashes and periodically replicates location updates through MPI_Allgatherv, allowing local lookup for cross-node prefix reuse. The data layer reads remote INT8 DRAM copies via MPI RMA and pipelines retrieval, reconstruction, and GPU transfer. Prefix-aware eviction retains reusable blocks in DRAM while cold blocks are written to aggregated Lustre files, reducing file metadata operations. Evaluation covers 64-node retrieval and 32-node serving with vLLM and SGLang.

### Strengths
- The system addresses a concrete deployment problem on Slingshot/Lustre HPC platforms by integrating MPI RMA, memory-resident lookup, and aggregated storage.
- Evaluation covers multi-node retrieval, tail latency, memory pressure, and end-to-end serving with both vLLM and SGLang.
- The paper transparently reports baseline transport limitations and negative DeepCAM results.

### Weaknesses
- Mooncake and LMCache (P2P) use TCP while CASCADE uses native MPI/CXI RMA. Without transport-controlled comparisons, the headline end-to-end gains cannot be separated into transport and cache-management contributions.
- The protocols for concurrent metadata updates and RDMA-buffer reclamation are underspecified.
- The task-quality impact of INT8 KV quantization is not evaluated directly.
- Predominantly high-sharing workloads leave performance under lower sharing and changing hotspots unclear.

### Comments for authors
Overall, the HPC-oriented integration and multi-node results are valuable. I currently lean weak reject because several key design guarantees and sources of improvement remain insufficiently supported.

1. Transport confounding (Sections 4.1, 4.6, 4.7).

Mooncake and LMCache (P2P) use TCP on the evaluated platform, whereas CASCADE uses native MPI/CXI RMA. The headline 9.39-16.75x TTFT and 7.17-12.42x throughput improvements therefore reflect the combined effects of transport and cache management. The authors explain the CXI compatibility constraints, but the contribution of each factor remains unclear.

Please provide a transport-controlled comparison, such as CASCADE over TCP with other settings fixed if feasible, or peer comparisons where all systems use native RDMA. Compression, capacities, workloads, and routing should also be controlled. Ablations for compression, prefix protection, and aggregated I/O would provide additional evidence. Figure 16's latency breakdown still reflects different transports, and transfer costs can amplify queueing. If further controls are unavailable, the architectural claims should be limited to the evaluated platform and configurations.

2. Design clarity and motivation (Sections 3.1, 3.2):

the paper describes a sharded Global Shard Index but also states that every node holds all shards. Please distinguish logical partitioning, update ownership, and replica placement, and explain the benefit of sharding under full replication. A multi-node PUT/GET example would clarify where entries are created, updated, and queried.

The motivation for a separate Dedup Map is also insufficiently explained. The paper compares a one-byte existence value with an 80-byte location entry, yet retains the complete location index, which already supports existence queries. Please explain the extra structure's benefit, its consistency with the location index, and its incremental cost relative to using the location index alone.

3. Concurrency (Sections 3.2, 3.3): the paper provides fallback recomputation, but the protocol for simultaneous duplicate allocations, metadata conflict resolution, and replica reclamation remains unclear. How are publication, checksum validation, and buffer reuse coordinated with outstanding RDMA reads?

4. KV quality (Section 3.3): FP16-to-INT8 conversion is lossy and changes both transfer volume and effective capacity. Please specify quantization granularity/scales, measure task-quality changes on the evaluated LongBench v2 requests, and report unquantized performance. The cited work should be supplemented with quality measurements for this implementation.

5. Applicability (Sections 4.1, 4.5): the evaluation primarily covers 80-95% sharing workloads and protected prefixes. Lower-sharing and changing-hotspot workloads would help establish the scope of the results. In Figure 12, the roughly 1 TB shared prefix fits distributed memory while most other data are disposable. Please report actual tier capacities, hit ratios, and PFS traffic to clarify cold-tier access under oversubscription.

6. Metadata scaling (Sections 3.2, 4.2): full index replicas still require collective update dissemination. Please report synchronization duration, CPU/network usage, propagation delay, and fallback rates as update rates and synchronization intervals change. Local lookup latency alone does not measure these costs.

Further clarifications: scope zero-copy/CPU-bypass claims to the relevant transfer segment; specify model/configuration namespaces for block IDs, tensor-parallel block-size accounting, and partial-prefix handling. Distinguish avoided PUT allocations from avoided model computation, and logical footprint from compressed physical bytes. Report software versions, warm-up/routing, run duration, output length, repetitions, and completion rates. Fixed-capacity Redis OOM and negative DeepCAM results should retain their stated configuration/workload scope.

### Questions for authors' response
Check my detailed comments.

---

## Review #165B

**Overall merit:** 2. Weak reject  
**Reviewer expertise:** 4. Expert

### Paper summary
The paper presents a full KV caching ecosystem spanning GPU HBM, CPU DRAM and Disk. There are a multitude of techniques employed in this solution:

1) A distributed global index maintained on the various GPUs using content based sharding, which is periodically synced with in memory replicas of the global index across all nodes
2) a multi-path RDMA implementation for moving data across DRAM tiers
3) a lossy compression is used on all non-GPU tiers
4) Maintaining a global deduplication of KV data and integrating this with eviction policies for the tiers.

There is an extensive set of tests, comparing the retrieval time of their solution vs. a number of existing open source solutions, and a small set of e2e inference tests. This all show very promising results vs. alternatives.

### Strengths
- some of the techniques (especially the indexing part) seem very interesting.
- strong results on many tests
- very decent scale of tests (upto 64 nodes, albeit relatively old GPUs)

### Weaknesses
- Paper is hard to follow. Lots of design details make it hard to understand the high level picture. Also it is hard to understand where the main benefits come from.
- Using lossy compression and comparing it to other methods that don't is problematic. It creates an unfair comparison and makes it hard to understand the actual source of improvement. You should also have a version of you test that does not do compression.
- Unclear if the data path improvements are novel, since other methods already use RDMA.
- I couldn't understand if the index is persisted and how it covers the 3 tiers

### Comments for authors
Some key questions:

- The lossy compression may effect both inter-node transfer speeds but more importantly fit more KV data in the same space. I would like to see how your system performs without it.
- Also not that some models will already be quantized and then your compression is not relevant. Bottom line need an apples to apples comparison.
- How are requests routed in your tests? Do I understand correctly that this is random routing and must rely on the transfer mechanisms for KV Cache hits? This would explain why in Figure 16 it seems that there are almost no CPU cache hits at all (in the CPU offload). Probably no GPU hits either...
- The index design seems nice But I couldn't understand if it is sharded by GPU, is the GPU HBM involved (or all is in CPU DRAM)? Does this also map data on Lustre? is it persistent?
- Figure 16 is very good - it finally helps understand where the improvements lie. It would also be helpful to understand what the cache hit rates are for each method.
- Both LMCache P2P and Moonckae use RDMA (or more precisely use a transfer engine that can support RDMA). I would love to see their transfer latency compared to yours, yet you only test them in the E2E tests. From figure 16 they seem in the same ball park with the difference potentially only the compression...
- Did you run LMCache and Mooncake with the same block size as you? Their default is often 256 while you use 1024 tokens. I'd like to see the same size for all.
- I'd like to understand the impact of your choice of using an eventually consistent index on cache hits. This is especially true when running a workload that frequently updates the index. It is unclear that you ran such tests.

Smaller comments:

- Table 1 is a bit of an eclectic collection of works that don't really compare with your work.

Page 3:
- Saying "KV Cache blocks must be offloaded to disaggregated shared storage" is a bit too strong. You can consider local storage and cache-aware routing, for example.
- paragraph starting with "Data Layer:" is very confusing: a) you discuss mainly Metadata in it so why call it data layer? b) you introduce a bunch of stuff like block_id, GPU_id shard which you don't explain. All of this finally makes sense in the next page (section 3.2). You should either make it a high level explaination (but readable), flip the order between the paragraphs, or drop this confusing paragraph.

Page 7:
- would be helpful to explain the exact workload used for weak and strong scalability tests. I was confused...

Page 8:
- in Figure 9 you should explain that "Hot" doesn't help much because in your tests there are hardly any GPU hits.
- The term "Variable-sized blocks" is confusing, I thought you were doing a sensitivity test to block size... Perhaps use "mixed block-size workload"?

Page 9:
- in section 4.4 about bursts should consider also a workload that updates metadata...

### Questions for authors' response
Mainly understand where the improvements come from. How much do compression and block size play a role.

---

## Review #165C

**Overall merit:** 3. Weak accept (accept if room)  
**Reviewer expertise:** 3. Knowledgeable

### Paper summary
As inference workloads grow to process long-context documents and multi-turn dialogues, managing KV cache across thousands of GPUs demands coordinated distributed caching beyond GPU HBM capacity. This paper presents CASCADE, a distributed KV cache management system designed to overcome the storage and metadata bottlenecks of LLM inference. CASCADE introduces a unified memory hierarchy that integrates GPU HBM, DRAM, and parallel file systems. With this design, CASCADE achieves scalable crossnode sharing through decentralized block management with relaxed-consistency synchronization, transfers data across tiers via multi-path zero-copy data movement without CPU serialization, and preserves shared prefixes in fast memory tiers through semantic-aware eviction. Our evaluation on a leadership-scale HPC system shows 5.7-28.5× lower KV cache retrieval latency than baselines across 64 nodes and 9.39-16.75× lower P99 TTFT and 7.17-12.42× higher prompt throughput than SOTA baselines in 32-node end-to-end LLM inference.

### Strengths
The paper is well written and addresses an important topic of reusing KV Cache across LLM inference prompts.

CASCADE is evaluated on a real Top20 supercomputer and has been integrated with vLLM and SGLang.

### Weaknesses
The paper builds a separate, PFS/Lustre-targeted KV Cache offloading backend to vLLM and SGLang instead of augmenting the widely used offloaders from the industry that presently mostly leverage NFS for centralized storage offloading (LMCache, HiCache, Mooncake). This makes the work HPC-centric and thus limiting in scope. Aspects like supporting CXI and Lustre cross-node deduplication support could be PR'ed into the LMCache/HiCache/Mooncake repos.

The recent models [https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/main/DeepSeek_V41_Tech_Report.pdf] have drastically reduced their KV Cache storage needs, so PFS may be an overkill going forward.

Your evaluation uses datasets like ShareGPT and LongBench which, although realistic, are quite old and belong more to the "chat" era of LLM inference as opposed to the current "agentic" era. Agents reuse much more of incremental context on each turn, so need to use modern agentic datasets like https://huggingface.co/semianalysisai/datasets

### Questions for authors' response
"In multi-tenant LLM serving, shared system prompts and conversation prefixes create redundant KV cache blocks across nodes. To eliminate this overhead, CASCADE employs content-addressed deduplication using Dedup Map, a distributed index tracking block existence."

Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment [https://docs.vllm.ai/en/stable/usage/security/#prefix-cache-timing-side-channel-mitigation-cache-salting]?

---

## Review #165D

**Overall merit:** 4. Accept  
**Reviewer expertise:** 2. Some familiarity

### Paper summary
CASCADE presents an architecture for scale-out KV caches used in LLM inference. They present a tiered design that includes GPU, DRAM, and parallel file systems. With a variety of techniques such as decentralized lock management, a global index for deduplication, and zero-copy data movement across nodes, they achieve lower latency and higher throughput in experiments up to 64 nodes.

### Strengths
This was an immense amount of work and the results are positive.

### Weaknesses
The writing is hard to follow throughout the paper. There are many technical details presented without explanation of their importance.

Experiments are missing statistical significance.

Many figures need improvement.

### Comments for authors
**1 Introduction**

LLM inference involves KV caches that grow larger than GPU high bandwidth memory or DRAM can support, so they propose an architecture including remote parallel file systems. Bandwidth and latency issues result. They point out the functionality gaps of multiple previous techniques.

CASCADE is available on github.

**2 Background**

2.1

The sequence of LLM token processing is reviewed including prefill memory usage and decoding, and both involve KV cache block accesses.

2.2

They argue for a coordinated block placement across GPU, HBM, DRAM, and parallel file systems.

Storage metadata requires random small accesses that reduces throughput.

KV cache blocks are often stored as individual files, so the metadata overhead in Lustre is high. They focus in CASCADE on reducing metadata operations with a unified hierarchy and aggregating writes into larger files.

**3 Cascade Design**

3.1

This is dense to follow.

3.2

This description is also dense but somewhat better.

KV pairs are grouped together, 320 MB units for distribution.

A SHA-256 Block ID is generated. It includes the preceding Block ID, so context is maintained. Hash-based routing selects the target GPU and shard. Deduplication is checked at that location.

Modified blocks are tracked, and only deltas are transferred.

"This map stores a boolean per Block ID, requiring 1 byte per block versus 80-byte Location Metadata entries in Global Shard Index." What is this 1 byte boolean used for? The next paragraph indicates that it indicates the status, and False indicates the block is evicted. But, why keep it in the index if it is False?

3.3

Local path: Why is it important to compress from FP16 to INT8 and then decompress? This is a local path situation, so is this compression and decompression necessary?

Remote path: RDMA reads of compressed blocks. How is the KV block recomputed on a validation failure?

3.4

Algorithm 1: "Blocks protected by Semantic Prefix Registry or identified through Dedup Map and Global Shard Index as reusable across nodes receive lower eviction priority." I don't see this in the algorithm. Also, why do you keep a DRAM shadow of everything in the GPU? That seems wasteful.

Algorithm 2: lines 8, 17, and 25 "Memory Pool[GPU].promote(b, data)" What does Memory Pool [GPU] mean? On line 4, you use a variable g to iterate over the the 4 GPUs.

**4 Evaluation**

4.1

The evaluation setup is very dense and complicated. There is hardware, LLMs, and multiple baselines discussed. The baselines have to be modified to work in this environment. While this is probably all necessary for this type of experiment, it is hard for a non-expert to follow why these particular details are important.

4.2

Figure 7: You need to clarify the difference between your weak and strong scalability tests. What point are you trying to make, and what is the significant difference in these tests? You mention a fixed 128-request workload, but what does that mean? I wouldn't think that 128 requests would be a long enough test to be meaningful. What is the statistical significance of these results? Did you run the experiments repeatedly, and what is the mean and standard deviation of the results? The throughput yaxis needs more ytics for 7b and 7f. Overall, the results suggest that CASCADE has lower latency and higher throughput than LMCache, PDC, and HDFS.

4.3

Figure 8: Tail latency (P99.9) is significantly lower for CASCADE than other techniques.

Figure 9: You need to show more ytics in 9a. What is the brown bar in 9b the 64 node run?

Average latency is significantly lower for CASCADE.

Figure 10: The experiment is repeated with mixed-length KV cache blocks. CASCADE has lower latency and higher throughput again. I think you need to explain the implications of mixed-length KV cache blocks. What does that really mean, and what intuition should we have about the expected results?

4.4

When do you measure the recovery latency? Is it immediately or after a period of recovery? How long are you taking these measurements? I suggest reordering the columns in Table 2 so Degradation goes immediately to the right of Burst. That would move Recovery (99) to the right column.

Figure 10, Table 2, and Figure 11 are tightly packed, and the paper as a whole is very dense with text and figures.

4.5

With 80-95% prefix sharing, CASCADE has lower latency and higher throughput than other techniques. I suggest reminding readers why prefix sharing is important. Otherwise, this result is getting lost with the other results. Figure 11b needs another ytic.

Figure 12: CASCADE keeps shared prefixes in GPU and achieves low latency under a high memory pressure experiment. Other techniques fail to keep the common prefixes in GPU and have high latency.

Figure 13: Cross-node deduplication has a ~15X reduction in space requirements compared to LMCache. The figure itself needs improvements. I realize the numbers have a large range, but using logscale for the yaxis is hiding a lot of detail in the results. Consider different ways to present this experiment.

4.6

TTFT is not defined in this paper. You use it in the abstract and introduction and other places. Define it in the introduction if not the abstract.

Figure 14: Generally, this figure shows CASCADE has the best latency and throughput in an end-to-end test across number of nodes. The importance of the prefix size (8K vs 64K) needs to be better explained. Several subfigures need more ytics.

4.9

Figure 18 shows DeepCAM, which has few duplicates to remove. Standard CASCADE underperforms, but it can be adjusted to disable deduplication and locking to be competitive with other approaches. It is not clear the safety of the streaming version that bypasses locks and aggregation.

---

## Review #165E

**Overall merit:** 3. Weak accept (accept if room)  
**Reviewer expertise:** 2. Some familiarity

### Paper summary
This paper presents a content-addressed, distributed KV cache storage system designed to eliminate storage bandwidth and file-system metadata bottlenecks in long-context, multi-node LLM inference across disaggregated HPC architectures. Unifying GPU HBM, host DRAM, and parallel file systems into a three-tier memory hierarchy, CASCADE decouples metadata orchestration from data movement. On the data path, it eliminates CPU serialization and per-block file metadata contention. Evaluated on up to 64 nodes and integrated into vLLM and SGLang, CASCADE achieves 5.7–28.5× lower KV cache retrieval latency in trace-driven benchmarks than LMCache and Mooncake.

### Strengths
+ Cohesive end-to-end architecture across GPU HBM, host DRAM, and parallel file systems
+ Portable, CPU-bypassing data path with semantic-aware retention
+ Comprehensive leadership-scale evaluation across engines and workloads

### Weaknesses
- Asymmetric transport and compression comparison against SOTA baselines
- Unevaluated INT8 quantization accuracy loss and missing decoding metrics
- Coarse block granularity, rigid MPI world assumptions, and unaddressed Lustre garbage collection

### Comments for authors
Thank you for submitting this paper to FAST. Addressing data movement and storage coordination bottlenecks in large-scale LLM serving is a timely and well-motivated systems problem. The paper presents a unified architecture across device memory, host memory, and shared storage. Pairing global duplicate detection with smart cache retention keeps shared prompt content in fast memory under heavy load.

My primary concern is the fairness of the performance comparisons against competing systems. On the evaluated cluster, the baseline systems cannot use the native high-speed network and fall back to slower standard networking, while the proposed system uses both the native high-speed interconnect and data compression to cut transfer sizes. Because of this mismatch, it is difficult to tell how much of the reported speedup comes from the core storage design versus faster networking and smaller payloads. Including a controlled breakdown that isolates each design choice would make the performance claims much more convincing.

In addition, the system relies on lossy compression for cached data stored outside device memory, yet the paper does not measure whether this degrades output quality on long-document tasks, relying instead on an external study. Similarly, although the introduction highlights token generation bottlenecks as a major motivation, the end-to-end experiments focus only on initial response delay and input processing speed. Measuring output quality alongside generation speed and full request completion time would provide a more complete picture of end-to-end serving behavior.

Finally, a few practical design trade-offs deserve further discussion. Managing cached data in very large fixed chunks keeps tracking overhead low, but it prevents finer-grained reuse when conversations diverge early, so clarifying how partial chunks are handled would be helpful. Moreover, evicted data is appended to large shared files on persistent storage, yet the paper does not explain how obsolete entries inside partially valid files are cleaned up over time without creating extra storage traffic. Lastly, the cross-machine coordination design assumes a fixed, failure-free group of servers; discussing how the system handles machine failures or dynamic scaling would clarify its readiness for long-running production deployments.

Thanks again for your submission. Looking forward to the response.

### Questions for authors' response
1. Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression?
2. Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?
3. How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
4. How does cross-machine coordination handle server failures or dynamic scaling?

---

## Comment @A1 — Reviewer D (posted about 12 h before the snapshot)

Dear authors, Thank you for your submission to FAST. In your rebuttal, please focus on these topics and then address other questions from the reviews if you have space.

1. Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?
2. Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?
3. How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?
4. How does cross-machine coordination handle server failures or dynamic scaling?
5. Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment
6. Please respond to other questions posted in the reviews.
