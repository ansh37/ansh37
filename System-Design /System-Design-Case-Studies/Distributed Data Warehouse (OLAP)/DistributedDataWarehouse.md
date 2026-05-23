# System Design Deep Dive: Distributed Data Warehouse (Teradata / OLAP)

**Level:** L5
**Topic:** Massively Parallel Processing (MPP), Columnar Storage, Distributed Joins, Data Skew

## 1. Problem Statement & Clarifications
Design a Distributed Data Warehouse (OLAP - Online Analytical Processing) capable of ingesting petabytes of data and executing complex analytical queries. 

Unlike OLTP (Online Transaction Processing) systems which prioritize sub-millisecond, single-row ACID transactions, an OLAP system optimizes for scanning billions of rows to calculate aggregations, execute massive joins, and power Business Intelligence (BI) dashboards. The core challenges are minimizing disk I/O, preventing network saturation, and managing distributed compute workloads.

## 2. Requirements

### Functional Requirements (FRs)
* **Data Ingestion:** System must support high-throughput bulk loading of structured and semi-structured data.
* **Complex Query Execution:** Support standard ANSI SQL for massive aggregations, group-bys, and multi-table joins.
* **Query Management:** Support asynchronous query execution, status polling, and result retrieval for long-running jobs.

### Non-Functional Requirements (NFRs)
* **Petabyte Scalability:** Must store and query 10+ PB of data without performance degradation.
* **High Throughput (over Latency):** Analytical queries may take minutes to hours; throughput and successful completion are prioritized over millisecond latency.
* **Fault Tolerance:** If a compute node crashes during a 4-hour query, the system must recover gracefully without restarting the entire job from scratch.
* **Linear Scalability:** Adding more compute nodes should linearly decrease query execution time.

## 3. Back-of-the-Envelope (BoTE) Calculations
* **Storage Scale:** 10 Petabytes (PB) of raw historical data.
* **Ingestion Rate:** 10 Terabytes (TB) of new data appended daily.
* **Compute Constraints:** A standard 10Gbps network link transfers ~1.25 GB/sec. Transferring 100 TB of data across the network to a single node for processing would take over 22 hours. 
* **Conclusion:** Moving data to compute is mathematically impossible at this scale. The architecture must push compute down to the data.

## 4. Core Entities & APIs

### Logical Entities
* `Partition / Micro-Partition`: The physical chunk of data stored on disk (typically 50MB - 500MB).
* `Execution Plan`: A Directed Acyclic Graph (DAG) representing the physical execution steps of a SQL query.

### Core APIs
* `POST /v1/queries` -> Submits a SQL query. Returns a `job_id`.
* `GET /v1/queries/{job_id}/status` -> Polls the status of the query (QUEUED, RUNNING, SUCCESS, FAILED).
* `GET /v1/queries/{job_id}/results` -> Retrieves the paginated result set.

## 5. High-Level Design (HLD)

<img width="1768" height="683" alt="image" src="https://github.com/user-attachments/assets/1b968865-d968-4bf2-9cd9-4a76f6294f3b" />


## 6. Detailed Data Flow

This section details the physical execution path of a massive analytical query, highlighting how the system minimizes network I/O and distributes compute across the cluster.

### 6.1 Path A: Query Parsing & Planning
1. **Query Submission:** A data analyst submits a complex aggregation query via a BI tool to the Master Node API: `SELECT country, SUM(revenue) FROM sales JOIN users ON sales.user_id = users.id GROUP BY country`.
2. **AST & Semantic Analysis:** The SQL Parser validates the syntax and converts it into an Abstract Syntax Tree (AST). It verifies that the user has permission to read the `sales` and `users` tables.
3. **Cost-Based Optimization (CBO):** The Optimizer queries the Metadata Catalog to fetch statistics (e.g., row counts, cardinality, disk locations). It evaluates thousands of potential execution paths and selects the one with the lowest CPU and I/O cost.
4. **Distributed Query Plan:** The query is compiled into a Directed Acyclic Graph (DAG) of physical execution tasks (Scans, Hashes, Shuffles, Aggregates).

### 6.2 Path B: Compute Pushdown (The Map Phase)
1. **Task Distribution:** The Master Node does not pull data over the network. Instead, it sends the executable query fragments (the "code") directly to the specific Compute Nodes that hold the relevant micro-partitions on their local SSDs or attached S3 storage.
2. **Columnar Read:** Each Compute Node scans the disk but only extracts the `revenue`, `user_id`, and `country` columns. All other columns (e.g., timestamps, user addresses) are completely ignored, saving massive disk I/O bandwidth.
3. **Local Hash & Filter:** Each node applies local filtering and builds an in-memory hash table for the upcoming join phase using only the data it locally possesses.

### 6.3 Path C: The Shuffle & Join Phase
1. **Broadcast or Shuffle Evaluation:** * If the `users` table is small (e.g., 10,000 rows), the Master Node broadcasts the entire table to all Compute Nodes. Joins happen instantly in local memory.
   * If both tables are massive, the cluster enters a **Shuffle Phase**.
2. **Network Redistribution:** Every Compute Node applies a hash function to the `user_id` column of its local rows. Based on the hash output, the nodes transmit their rows across the network to new target nodes.
3. **Deterministic Colocation:** By the end of the shuffle, the system guarantees that all `sales` records and `users` records sharing the same `user_id` physically reside on the exact same Compute Node.
4. **Local Join:** The Compute Nodes execute the join locally on the newly shuffled data.

### 6.4 Path D: Aggregation & Result Retrieval (The Reduce Phase)
1. **Partial Aggregation:** Each Compute Node calculates the `SUM(revenue)` grouped by `country` for its specific chunk of data.
2. **Global Aggregation:** The Compute Nodes transmit these small, partial aggregate numbers back to the Master Node.
3. **Final Merge:** The Master Node merges the partial sums into the final, globally accurate result set.
4. **Delivery:** The Master Node paginates the final output and returns it to the client's BI tool.

---

## 7. Deep Dives & Trade-offs

### 7.1 The Storage I/O Problem: Row vs. Columnar
* **Basic Approach:** Store data in standard B-Trees or Row-oriented formats (like PostgreSQL or MySQL).
* **The Problem:** Analytical queries rarely need all columns. If a table has 100 columns and the query asks for `SUM(revenue)`, a row-oriented database must read all 100 columns from the disk into RAM, wasting 99% of disk bandwidth.
* **Optimal Pattern:** **Columnar Storage (Apache Parquet / ORC)**.
* **Trade-off Analysis:** Data is stored on disk grouped by columns. The query reads *only* the specific disk sectors containing the requested columns. Because columns contain homogeneous data types (e.g., all integers), we apply aggressive **Run-Length Encoding and Dictionary Compression**, drastically reducing the physical storage footprint. The trade-off is that single-row inserts and updates (`UPDATE sales SET...`) are extremely expensive, which is why OLAP systems rely on immutable, append-only bulk ingestion.

### 7.2 The Network Bottleneck: Compute Pushdown
* **Basic Approach:** The Master Node requests all data from the storage nodes, pulls it over the network into a central server, and runs the calculation.
* **The Problem:** Moving 100 Terabytes of data across a network switch will cause total network saturation and take days to complete.
* **Optimal Pattern:** **The MapReduce Paradigm (Compute Pushdown)**.
* **Trade-off Analysis:** The compiled execution plan is just a few kilobytes of code. We move the code to the data, not the data to the code. Each node processes its local data and returns only a tiny aggregate number to the Master Node. This turns massive network data transfer into isolated CPU-bound tasks.

### 7.3 Data Skew: The "Out of Memory" Trap
* **Basic Approach:** Hash partition the data across servers using a logical key, such as `HASH(Country) % Node_Count`.
* **The Problem:** If partitioned by `Country`, Server 1 gets Vatican City (1,000 rows) and Server 2 gets India (1.4 Billion rows). Server 1 finishes its task in milliseconds, while Server 2 runs out of memory (OOM) and crashes, failing the entire global query.
* **Optimal Pattern:** **Salting and Composite Sharding Keys**.
* **Trade-off Analysis:** Instead of partitioning by `HASH(Country)`, we partition by a composite key: `HASH(Country + Random_Int(1_to_10))`. This "Salt" forces the massive India dataset to be mathematically sliced into 10 perfectly even chunks and distributed uniformly across 10 different servers. This entirely eliminates data skew and guarantees uniform CPU utilization. The trade-off is slightly higher query complexity, as the Master Node must aggregate the 10 separate "India" buckets at the end of the query.

### 7.4 Workload Management & Noisy Neighbors
* **Basic Approach:** First-In-First-Out (FIFO) query execution queue.
* **The Problem:** A junior data analyst submits a poorly written cross-join query that consumes 100% of the cluster's CPU for 6 hours, preventing the CEO's critical real-time financial dashboard from loading.
* **Optimal Pattern:** **Multi-Level Workload Management (WLM) & Resource Queues**.
* **Trade-off Analysis:** The Master Node assigns queries to specific resource pools based on the user's IAM role. The CEO's dashboard is routed to a "High Priority" queue guaranteed 40% of cluster CPU. The analyst's query is routed to a "Batch" queue that is throttled to 10% CPU. If the analyst's query exceeds memory limits, the system terminates it automatically to protect cluster health.

### 7.5 Architecture Evolution: Shared-Nothing vs. Storage/Compute Separation
* **Basic Approach:** Shared-Nothing Architecture (Classic Teradata/Hadoop). Each compute node physically owns its own hard drives.
* **The Problem:** Scaling compute and storage are tightly coupled. If you run out of disk space, you must purchase a new expensive server (CPU + RAM + Disk), even if your CPU utilization is currently at 5%.
* **Optimal Pattern:** **Separation of Compute and Storage (Snowflake / AWS Redshift RA3 model)**.
* **Trade-off Analysis:** Storage is completely offloaded to a cheap, infinitely scalable object store (AWS S3). Compute nodes are stateless and pull data from S3 dynamically over a high-speed network backbone. This allows the system to auto-scale compute clusters up during heavy query loads and spin them down to zero during off-hours to save costs, while the storage remains permanent. The primary trade-off is higher network latency when fetching cold data from S3, which is aggressively mitigated by caching frequently accessed micro-partitions on local NVMe SSDs attached to the compute nodes.
