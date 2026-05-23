# System Design Deep Dive: Enterprise Issue Tracker (Jira Scale)

**Level:** Staff/Principle
**Topic:** Multi-Tenant Architecture, State Management, CQRS, Concurrency

## 1. Problem Statement & Clarifications
Design a B2B Enterprise Issue Tracking system (similar to Jira). The system allows thousands of companies to manage millions of tasks, define custom workflow state machines, and collaborate in real-time. 

The primary engineering challenges are handling concurrent data mutations safely, isolating massive datasets by tenant, and separating strict ACID transactional state from complex, low-latency full-text search capabilities.

## 2. Requirements

### Functional Requirements (FRs)
* **Ticket Management:** Users can create, update, comment on, and transition tickets through workflow states.
* **Custom Workflows:** Each project can define its own state transitions (e.g., To-Do -> QA -> Done).
* **Advanced Search (JQL):** Users can run complex, nested text searches across millions of tickets.
* **Real-time Updates:** Ticket status changes reflect instantly on the UI for anyone viewing the board.
* **Notifications:** The system must reliably send email and push notifications for ticket assignments and mentions.

### Non-Functional Requirements (NFRs)
* **Multi-Tenant Isolation:** B2B SaaS architecture; data from Company A must be strictly isolated from Company B. Over 100M issues must be stored without performance degradation.
* **High Availability & Consistency:** The system is read-heavy, but core writes must be strictly consistent (ACID).
* **Concurrency Control:** The system must gracefully handle simultaneous edits to the same ticket without data loss.
* **Low Latency Search:** Complex JQL queries must return in < 200ms.

## 3. Back-of-the-Envelope (BoTE) Calculations
* **Scale:** 10,000 Enterprise Tenants, 50 Million Daily Active Users (DAU).
* **Throughput:**
  * Writes (Creates/Updates): 2,000 TPS.
  * Reads (Views/Searches): 200,000 TPS (100:1 Read/Write ratio).
* **Storage:** 50M tickets/day x 5 KB = 250 GB/day (~90 TB/year).
* **Conclusion:** The massive read/search volume compared to writes dictates a decoupling of the search engine from the primary transactional database using the CQRS pattern.

## 4. Core Entities & APIs

### Entities
* `Issue`: `id`, `tenant_id`, `project_id`, `status`, `assignee_id`, `description`, `version` (integer for locking).
* `Workflow`: `project_id`, `transitions_json` (A Directed Acyclic Graph defining allowed states).
* `Comment`: `id`, `issue_id`, `user_id`, `body`, `created_at`.

### Core APIs
* `PATCH /v1/issues/{id}` -> Updates a ticket. Payload must include the current `version` integer for concurrency control.
* `POST /v1/search` -> Executes a complex JQL query (e.g., `project=ENG AND status=DONE`).

## 5. High-Level Design (HLD)

<img width="2005" height="743" alt="image" src="https://github.com/user-attachments/assets/ad6ebe2e-0904-4039-bfd8-e2cc96440216" />

## 6. Detailed Data Flow

This section outlines the step-by-step data execution for the three most critical paths in the system: mutating state, executing search, and fan-out notifications.

### 6.1 Path A: The Write Path (Updating a Ticket)
1. **Client Request:** The user submits a `PATCH /v1/issues/123` request containing the updated fields and the current `version` integer.
2. **Authorization (AuthZ):** The API Gateway forwards the request to the Authorization Service (Zanzibar-based) to verify if the user has `WRITE` access to the specific project.
3. **Workflow Validation:** The Ticket Service fetches the project's custom Workflow DAG from the Redis cache to ensure the requested state transition is legally allowed.
4. **Optimistic Locking Execution:** The service executes the database update: `UPDATE issues SET status='Done', version=version+1 WHERE id=123 AND version=<client_version>`. 
5. **Conflict Resolution:** If the database returns 0 affected rows, the service throws a `409 Conflict`, returning the latest database state to the client for frontend merging.
6. **Change Data Capture (CDC):** If successful, Debezium reads the PostgreSQL Write-Ahead Log (WAL) at the disk level and publishes an `issue_updated` event to the primary Kafka cluster.

### 6.2 Path B: The Search Path (Executing JQL)
1. **Index Synchronization (Background):** A dedicated consumer reads the `issue_updated` Kafka topic and updates the Elasticsearch Inverted Index. This ensures eventual consistency (typically within sub-milliseconds).
2. **Query Submission:** The user submits a JQL string (e.g., `project = ENG AND text ~ "latency"`).
3. **AST Parsing:** The Search Service parses the JQL string into an Abstract Syntax Tree (AST) to validate syntax and prevent injection attacks.
4. **Elasticsearch Translation:** The AST is translated into a native Elasticsearch JSON query.
5. **Data Retrieval:** Elasticsearch executes the query, returning the matching `issue_ids` and hydrated ticket metadata to the client in under 200ms.

### 6.3 Path C: The Real-Time & Notification Path (Fan-Out)
1. **Event Consumption:** Both the WebSocket Manager and Notification Service independently consume the `issue_updated` Kafka topic.
2. **Live UI Updates:** The WebSocket Manager queries Redis for `Key: Issue_123_Viewers` to retrieve the connection IDs of users currently viewing the ticket. It pushes the delta payload directly to those TCP sockets.
3. **Watcher Fan-Out:** The Notification Worker queries the primary database to find all users subscribed to the ticket (the "Watchers").
4. **Task Enqueueing:** The worker splits the notification task into individual payloads and drops them into an Amazon SQS queue.
5. **Idempotent Delivery:** Fleet workers pull from SQS. Before sending an email via SendGrid, the worker sets an expiring hash in Redis to ensure duplicate emails are not sent if a Kafka partition rebalances.

---

## 7. Deep Dives & Trade-offs

At the enterprise scale, naive approaches fail under the weight of concurrent users and massive datasets. Below is the architectural evolution from standard Senior-level patterns to Staff-level optimizations.

### 7.1 Database Architecture & Isolation
* **Basic Approach:** A monolithic relational database or a single NoSQL table.
* **The Problem:** 100 million issues create massive B-Tree index overhead. NoSQL fails to support the strict relational integrity and complex joins required for project management tools.
* **Optimal Pattern:** **PostgreSQL with Tenant-Based Sharding**. 
* **Trade-off Analysis:** By sharding physically based on `tenant_id`, we guarantee that Company A's data never resides on the same disk as Company B's data. This allows infinite horizontal scaling. The trade-off is operational complexity; we must deploy a routing proxy (like Vitess or Citus) to direct incoming queries to the correct physical database shard based on the user's JWT token.

### 7.2 Concurrency Control (The Lost Update Problem)
* **Basic Approach:** Pessimistic Locking (`SELECT FOR UPDATE`).
* **The Problem:** Locking the database row blocks all other readers and writers, freezing the UI for collaborators and creating massive database connection pool bottlenecks.
* **Optimal Pattern:** **Optimistic Concurrency Control (OCC) + Client-Side Merging**.
* **Trade-off Analysis:** We shift the burden of conflict resolution from the database to the application layer. The database simply enforces an atomic integer check (`version`). The trade-off requires building complex frontend logic (using Redux or similar state management) to cache the user's unsaved input, catch the `409 Conflict`, and render a visual diff/merge tool so no human effort is lost.

### 7.3 Search & Querying 
* **Basic Approach:** Running SQL `LIKE '%term%'` queries against the primary relational database.
* **The Problem:** Full table scans will lock up the primary database, causing total system outages.
* **Optimal Pattern:** **Command Query Responsibility Segregation (CQRS) via CDC**.
* **Trade-off Analysis:** We separate write concerns (PostgreSQL) from read/search concerns (Elasticsearch). Using Debezium for CDC prevents distributed transaction failures. The trade-off is giving up strict read-after-write consistency for search. Search results are *eventually consistent*, meaning a user might update a ticket and not see it in search results for ~50 milliseconds.

### 7.4 Workflow State Machines
* **Basic Approach:** Hardcoding business logic (`if status == 'To-Do' then allow 'In-Progress'`).
* **The Problem:** Enterprise clients require customized workflows. Hardcoding logic prevents multi-tenant customization.
* **Optimal Pattern:** **Directed Acyclic Graphs (DAGs) as Data**.
* **Trade-off Analysis:** Workflows are modeled as JSON-based adjacency lists stored in the database and aggressively cached in Redis. The backend uses a finite state machine evaluator to process transitions dynamically. This provides maximum flexibility but requires robust cyclic dependency checks when a user attempts to save a new workflow configuration.

### 7.5 Access Control & Team-Level Security
* **Basic Approach:** Role-Based Access Control (RBAC) checking simple user roles.
* **The Problem:** At Amazon scale, access is not just based on roles, but relationships (e.g., User A is part of Team B, which owns Project C, which contains Ticket D).
* **Optimal Pattern:** **Relation-Based Access Control (ReBAC) / Google Zanzibar architecture**.
* **Trade-off Analysis:** Instead of complex SQL joins to verify permissions on every request, we offload access checks to a dedicated, globally distributed AuthZ service. The AuthZ service evaluates graph relationships in micro-seconds. The trade-off is the engineering overhead of synchronizing domain data (like project ownership changes) to the AuthZ system in real-time.

### 7.6 Disaster Recovery & Point-In-Time Recovery (PITR)
* **Basic Approach:** Nightly database snapshots.
* **The Problem:** If a database is corrupted at 4:00 PM, an entire day of enterprise work is permanently lost.
* **Optimal Pattern:** **Continuous WAL Archiving (pgBackRest / WAL-G)**.
* **Trade-off Analysis:** We stream the PostgreSQL Write-Ahead Log directly to AWS S3. If data is corrupted, infrastructure teams can restore the base backup and replay the S3 logs to the exact second before the corruption occurred (PITR). This guarantees zero data loss but incurs higher S3 storage and network transfer costs.

