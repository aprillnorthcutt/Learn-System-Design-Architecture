# 🧭 System Design Patterns Overview

> **Architecture determines reliability, scalability, cost, and latency.**

This document summarizes the **core concepts and foundational patterns** of system design.  
Each section includes a short explanation, visual diagram, and key trade-offs.

---

## 📦 Storage

Choosing the right data store for performance, scalability, and cost.

![Storage Diagram](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/storage.png?raw=true) <br>
*Figure 1 – Data storage strategies and types (SQL, NoSQL, Blob).*

**Key ideas**
- Use **Relational DBs** (PostgreSQL, MySQL) for ACID transactions.  
- Use **NoSQL** (Cassandra, DynamoDB) for horizontal scalability.  
- Use **Object storage** (S3, Blob) for media and static assets.

---

## 🧮 Partitioning & Sharding

Splitting data logically or physically to improve scalability and fault isolation.

![Partitioning & Sharding](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/partitioning-sharding.png?raw=true) <br>
*Figure 2 – Logical vs physical data distribution.*

**Key ideas**
- Partitioning divides data within a DB (by region, user ID, etc.).  
- Sharding distributes partitions across multiple servers.  
- Helps scale horizontally and contain failures.

---

## ♻️ Redundancy & Replication

Ensuring availability and durability through duplication.

![Redundancy & Replication](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/redundancy-replication.png?raw=true) <br>
*Figure 3 – Replication and failover patterns.*

**Key ideas**
- Replicate from primary → read replicas for load distribution.  
- Add redundant servers to eliminate single points of failure.  
- Combine both for fault-tolerant systems.

---

## ⚖️ Load Balancing

Distribute user requests evenly across multiple backend servers.

![Load Balancing](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/load-balancing.png?raw=true) <br>
*Figure 4 – Round Robin, Least Connections, and IP Hash strategies.*

**Key ideas**
- Use **Round Robin** for simplicity.  
- Use **Least Connections** for dynamic workloads.  
- DNS-level balancing for global routing.

---

## 🧊 Caching

Store frequently accessed data close to users to reduce latency.

![Caching](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/caching.png?raw=true) <br>
*Figure 5 – Caching layers and eviction strategies.*

**Key ideas**
- Place caches between app ↔ DB.  
- Strategies: LRU, LFU, TTL.  
- Balance freshness vs performance.

---

## 🌍 Content Delivery Network (CDN)

Serve static content from edge locations near users.

![CDN](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/cdn.png?raw=true) <br>
*Figure 6 – Edge delivery and origin fallback.*

**Key ideas**
- Use CDNs for media, JS/CSS, and images.  
- Reduces latency and server load.  
- Improves availability during spikes.

---

## 🚦 Rate Limiting & Throttling

Protect systems from abuse and traffic spikes.

![Rate Limiting](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/rate-limiting.png?raw=true) <br>
*Figure 7 – Token bucket algorithm controlling request flow.*

**Key ideas**
- Smooths bursts while maintaining throughput.  
- Common algorithms: Token Bucket, Leaky Bucket.  
- Implement per API key, IP, or user.

---

## ⚙️ Asynchronous Processing

Handle background work off the main thread for better responsiveness.

![Async Processing](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/async-processing.png?raw=true) <br>
*Figure 8 – Service → Queue → Workers flow.*

**Key ideas**
- Decouple slow tasks via queues (Kafka, SQS).  
- Enable retries and DLQs.  
- Improves throughput and resilience.

---

## 🧩 CAP Theorem

A distributed system cannot simultaneously guarantee all three:  
**Consistency, Availability, Partition tolerance.**

![CAP Theorem](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/cap-theorem.png?raw=true) <br>
*Figure 9 – CAP trade-offs (CP vs AP systems).*

**Key ideas**
- **CP:** prefer accuracy, may reject requests.  
- **AP:** prefer uptime, may return stale data.  
- Partition tolerance is mandatory.

---

## 🧠 PACELC Theorem

Extends CAP — when no partition occurs, systems trade **Latency** vs **Consistency**.

![PACELC Theorem](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/pacelc.png?raw=true) <br>
*Figure 10 – Latency vs Consistency under normal conditions.*

**Key ideas**
- During partition → choose between C and A.  
- Else → trade between L and C.  
- Guides modern distributed DB design.

---

## 📊 Functional vs Non-Functional Requirements

![Functional vs Non-Functional Requirements](https://github.com/aprillnorthcutt/Learn-System-Design-Architecture/blob/main/images/func-nonfunc.png?raw=true) <br>
*Figure 10 – Latency vs Consistency under normal conditions.*

| Type | Examples |
|------|-----------|
| **Functional** | Upload photos, create accounts, generate invoices |
| **Non-Functional** | Latency < 200 ms, 99.9 % uptime, secure authentication |

---

## 💡 Takeaways

- Architecture choices drive reliability, scalability, and latency.  
- Every decision is a trade-off — there’s no silver bullet.  
- Scale horizontally early; optimize vertically later.  
- Pick consistency, availability, and latency trade-offs consciously.  

---

> 🧩 *Part of April Northcutt’s System Design Series — visual, educational patterns for scalable architecture.*
