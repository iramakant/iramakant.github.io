---
layout: post
title: Welcome to My Blog
date: 2026-04-04 01:53:47 +0000
---

```markdown
*(Note: The specific calculation that an average update operation costs roughly 10 RUs is drawn from outside of the provided sources, based on our previous conversation, and you may want to independently verify this specific metric. All other metrics, limits, and pricing details are directly from the provided sources.)*

# Architecting for Scale: Azure Cosmos DB Serverless vs. Provisioned Throughput

When designing data-intensive applications, choosing the right capacity model in Azure Cosmos DB is a critical architectural decision. Azure Cosmos DB offers two primary capacity modes: provisioned throughput (which reserves capacity) and serverless (which bills based on consumption). 

While Serverless sounds appealing for reducing idle costs, high-throughput write workloads reveal hidden architectural constraints. Let's look at how these two models handle **10,000, 5,000, and 1,000 record updates per second**.

### The Baseline Math: RUs and Item Size

In Azure Cosmos DB, the Request Unit (RU) is the fundamental currency that normalizes the physical resources required to perform database operations, including CPU, memory, and IOPS. A point-read of a 1 KB document by its unique identifier and partition key is the baseline operation and costs exactly 1 RU. 

Because an update operation inherently requires more effort, it averages about 10 RUs for a 1 KB document (as discussed previously). Applying this math, we can estimate the throughput required for our update workloads:
* **10k updates/sec** = ~100,000 RU/s
* **5k updates/sec** = ~50,000 RU/s
* **1k updates/sec** = ~10,000 RU/s

### The Hidden Trap: Physical Partition Limits

Both capacity models distribute your data across logical and physical partitions to achieve horizontal scale. However, their throughput densities differ significantly:
* **Serverless Mode:** Hard-capped at **5,000 RU/s per physical partition**. 
* **Provisioned Throughput:** Can handle up to **10,000 RU/s per physical partition**.

In serverless mode, your total capacity only scales linearly as your data size grows and forces physical partition splits. If your database has a small storage footprint (under 50 GB), it fits on a single physical partition and is physically incapable of bursting beyond 5,000 RU/s. 

Here is how this constraint impacts our three scale scenarios:

---

### Scenario 1: 10,000 updates/sec (~100,000 RU/s)

* **Serverless:** **Total Failure.** Your workload requires 100,000 RU/s, but a new serverless container maxes out at 5,000 RU/s. You will be severely rate-limited and overwhelmed with HTTP 429 throttling errors. To organically achieve 100,000 RU/s in Serverless, your dataset would need to be massive enough to span 20 physical partitions. 
* **Provisioned Throughput:** **The Only Choice.** By manually provisioning 100,000 RU/s (or using Autoscale), Azure Cosmos DB pre-allocates the necessary physical partitions behind the scenes. Furthermore, Provisioned Throughput guarantees **< 10-millisecond write latency at the 99th percentile (P99) under a strict SLA**, whereas Serverless only offers a looser < 30-millisecond Service Level Objective (SLO) for writes.

### Scenario 2: 5,000 updates/sec (~50,000 RU/s)

* **Serverless:** **Severe Throttling.** At 50,000 RU/s, you are still demanding 10x the maximum throughput of a single serverless partition. Unless you have hundreds of gigabytes of stored data to force partition splits, your application will crash during these bursts.
* **Provisioned Throughput:** **Perfect Fit.** If this traffic represents unpredictable spikes, **Autoscale Provisioned Throughput** is ideal. You can set a maximum limit, and the system will automatically and instantaneously scale the available RU/s between 10% of that maximum and the full value based on instantaneous demand. 

### Scenario 3: 1,000 updates/sec (~10,000 RU/s)

* **Serverless:** **Highly Risky.** Updating 1,000 records per second requires roughly 10,000 RU/s. Because a serverless container's single-partition burst limit is 5,000 RU/s, you will still hit the ceiling and experience 429 throttling errors. 
* **Provisioned Throughput:** **Safest and Most Efficient.** Provisioning exactly 10,000 RU/s ensures your workload runs smoothly, supported by dedicated resource allocation ensuring the database engine can process requests without contention. 

---

### The Economic Tipping Point (TCO)

The choice between the two models is often driven by a Total Cost of Ownership (TCO) analysis. 

Serverless is billed strictly by consumption at a rate of approximately **$0.25 per 1 million RUs**. Standard provisioned throughput bills at a fixed hourly rate of **$0.008 per 100 RU/s**. 

The generally accepted breakeven point where Provisioned Throughput becomes more cost-effective is between **100 million and 250 million RUs per month**. 

If your app pushes 1,000 to 10,000 updates per second on a sustained basis, you will easily consume billions of RUs per month, making **Provisioned Throughput significantly cheaper** than the pay-as-you-go serverless premium.

### Conclusion

Serverless mode is architecturally optimized for "spiky" workloads characterized by long idle times and unpredictable bursts of activity. However, **if your workload demands thousands of writes per second, Provisioned Throughput is an architectural necessity** to avoid the 5,000 RU/s serverless partition cap and to maintain strict low-latency SLAs.

![A placeholder image](/assets/images/placeholder.svg)