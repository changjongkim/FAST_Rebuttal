## 1. Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?

## 2. Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?

## 3. How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?

## 4. How does cross-machine coordination handle server failures or dynamic scaling?

## 5. Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment

---

## 6. Please respond to other questions posted in the reviews.

**Global Shard Index and Dedup Map.**

**Concurrency and consistency.**

**Workload and scope.**

**Statistics and presentation.**
