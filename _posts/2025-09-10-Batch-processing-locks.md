---
layout: post
title: Eliminating Deadlocks in Batch Processing
tags: [Performace, Database]
---

Over the past few weeks, we significantly scaled our overnight batch imports. These imports were designed to pull daily records from multiple source systems and aggregate them into fewer consolidated records — a common requirement for analytics, reporting, and downstream processing.

At the beginning of the year, the initial solution was built using Spring Batch with JPA-based readers and writers. This helped us move quickly, as most required components were readily available. We rapidly deployed the solution and gradually increased the load every month. Initially, the volumes were small (10–20K records/day), and the batch job processed them without issue. As we started scaling, processing time naturally increased, but it was still acceptable because the job ran overnight.

However, as the data volume crossed 200K records/day, the processing time exceeded 10 hours, meaning the data was not ready when business users started their day. The long-term target was to process around 5 million records, and while the original design worked well for early assessments, it was no longer suitable for production-scale workloads.

## Debugging Performance Bottlenecks  

Our first instinct was the classic one: increase concurrency. More threads, bigger pool, more compute. But as we increased the thread pool, the job actually became slower. A deeper analysis of logs and job metrics revealed frequent deadlocks during record aggregation. These deadlocks caused write failures and triggered retries because the exception was part of the retry policy.

```
org.springframework.dao.CannotAcquireLockException: could not execute statement [ERROR: deadlock detected
Detail: Process 32210 waits for ShareLock on transaction 50823300; blocked by process 32213.
Process 32213 waits for ShareLock on transaction 50823301; blocked by process 32210.
Hint: See server log for query details.]
```

I’ve seen similar patterns in other domains, especially logistics systems handling high-frequency state transitions. The safe approach there was to offload the write logic into Oracle stored procedures. In the current scenario, we wanted to avoid stored procedures and look for alternatives that preserved throughput without introducing tight DB coupling.

## Alternative Design

Our goals were clear to maintain high throughput and ensure data integrity. The application layer can enforce consistency using locks, but this would drastically reduce throughput. The database layer is the ideal place to enforce integrity via constraints — however, naïvely using optimistic locking and unique constraints led to excessive row and index locking, causing retries and deadlocks.

We initially explored using the `ON CONFLICT` clause to ensure updates were applied correctly. However, under high concurrency, this triggered index-level lock contention on unique constraints. To address this, we moved to a more scalable two-phase design.

1. Use a staging table for raw imports. The staging table contains no constraints, allowing maximum ingestion throughput.

```sql
INSERT INTO staging_data (name, type, address, status, is_enabled, source)
VALUES (:name, :type, :address, :status, :isEnabled, :source)
```

2. Keep constraints in the final target table. The target table holds the required unique constraints (e.g., UNIQUE(name, type)), ensuring data integrity. Conflict resolution should be handled using ON CONFLICT.

3. Move data from staging to target in a single SQL statement. This ensures that index updates and conflict resolution occur in one transaction, minimizing locking and eliminating concurrent index modifications.

```sql
INSERT INTO target_analysis (name, type, address, status, is_enabled)
SELECT name, type, address, status, is_enabled
FROM staging_data WHERE source = :source
ON CONFLICT (name, type)
DO UPDATE SET 
    status = EXCLUDED.status,
    updated_at = NOW(),
    ref_count = target_analysis.ref_count + 1;
```

This pattern eliminates interleaved writes from multiple threads and consolidates all index updates into a single operation.

With the new design, we significantly improved the throughput of our batch process. Using 30 threads, we were able to import 1 million records in under an hour, with zero deadlocks.

