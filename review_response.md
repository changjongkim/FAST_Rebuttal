## 1. Controlled breakdown of the gains and block size

> Can you provide a controlled breakdown separating the gains of your storage design from faster networking and data compression? How does block size affect improvements?

## 2. Output quality, token generation, and full request time

> Does lossy compression affect output quality on long-document tasks, and how does the system impact token generation speed and full request time?

## 3. Partial chunk reuse and reclamation of persistent files

> How does the system reuse partial chunks on early conversation divergence and reclaim obsolete data in shared persistent files?

## 4. Server failures and dynamic scaling

> How does cross-machine coordination handle server failures or dynamic scaling?

## 5. Cross-tenant sharing and salting

> Sharing KV Cache blocks across tenants may introduce an information channel in between tenants, so information can be leaked (for example, fast TTFT for a never-used before prompt suggests to the attacker that a co-tenant used the same prompt before). Is salting optional in your HPC environment

---

## 6. Other questions

**Global Shard Index and Dedup Map.**

**Concurrency and consistency.**

**Workload and scope.**

**Statistics and presentation.**
