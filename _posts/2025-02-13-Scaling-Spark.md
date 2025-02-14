---
layout: post
title: Notes on Scaling Spark Jobs 
#subtitle: Each post also has a subtitle
tags: [Spark, Scaling]
---

Beginning in 2025, we set the goal of onboarding a large number of customers to our product. The product had been in development over the past year and was being demoed to prospective clients. The sales team started covering ground, and the first few customers were rolled out quickly. These were enterprise customers who pushed large volumes of their data.  

We had used Spark as the foundation of our analytical architecture. However, coming from a web and mobile development background, I realized that several best practices I was familiar with were quite different when it came to scaling Spark. Here are some key lessons we learned:  

### 1. **Avoid API-First Approach**  
In application development, we heavily rely on APIs for all operations. However, using the same approach for data analytics can become a bottleneck. API calls introduce unnecessary overhead and network latency, slowing down data processing. Instead, it's better to pull all data at once using:  
- **Direct database queries** for structured data  
- **Data files (Parquet, ORC, Avro, CSV, etc.)** stored in distributed storage (HDFS, S3, etc.)  
- **Bulk APIs** where necessary but optimized for batch retrieval  

### 2. **Leverage UDFs for Scalable Computations**  
User-Defined Functions (UDFs) allow us to perform complex transformations on data efficiently. Since UDFs operate in a distributed manner, they can scale with the underlying hardware capacity. However, a few best practices are essential:  
- Use **Scala or Java UDFs** instead of Python (PySpark) UDFs for better performance.  
- Consider **pandas UDFs** (vectorized UDFs) in PySpark for better speed.  
- Keep UDF logic simple and avoid excessive branching.  
- Handle errors well to avoid unexpected behaviours

### 3. **Optimize Data Partitioning**  
Partitioning is crucial in Spark because it determines how data is distributed across nodes. Poor partitioning leads to data skew, causing some executors to do more work than others. To optimize partitioning:  
- Choose an appropriate partition key to balance data distribution.  
- Use **repartition()** or **coalesce()** wisely—repartition() increases parallelism but shuffles data, while coalesce() minimizes shuffling.  

### **4. Avoid Unnecessary `count()` Operations**  
The `count()` operation in Spark triggers a full scan of the dataset, which can be extremely expensive, especially on large datasets. Instead of using `count()` to check if a DataFrame is empty or to estimate size:  
- Use **take(1)** or **isEmpty()** if you just need to check for data presence.  
- Use **approximate count techniques**, such as **df.rdd.countApprox(timeout)** for faster estimations.  
- If an exact count is required, try **optimizing queries** by using indexes, partitions, or pre-aggregated summaries.  

### 5. **Broadcasting Small DataFrames for Efficient Joins**  
One of the biggest performance bottlenecks in Spark is shuffling large datasets during joins. If one of the tables is small, use **broadcast joins** to speed up execution.  
- Use **broadcast(df)** to distribute small DataFrames to all nodes, avoiding expensive shuffle operations.  
- Ensure the DataFrame is genuinely small (typically <100MB) to prevent excessive memory usage.  

### 6. **Caching and Checkpointing for Iterative Workloads**  
If your Spark job performs multiple transformations on the same dataset, **caching** or **checkpointing** can significantly improve performance:  
- Use **df.cache()** for frequently accessed DataFrames to keep them in memory.  
- Use **checkpointing** for long lineage RDDs to avoid recomputation.  
- Be mindful of memory constraints—only cache datasets that fit within available executor memory.  

By applying these strategies, we were able to scale our Spark jobs efficiently and handle large volumes of data without major performance issues. 

