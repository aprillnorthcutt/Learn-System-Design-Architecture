# ⚙️ Kafka Complete Guide

*A visual, beginner-friendly guide to how Kafka handles data flow, reliability, and recovery — designed for ADHD brains.*

---

## 📑 Table of Contents

1. [🧩 Core Concepts & Architecture](#c-ore-concepts--architecture)#-core-concepts--architecture
2. [🎛️ Control Knobs & Configuration](#-control-knobs--configuration)
3. [🔒 Messaging Guarantees](#-messaging-guarantees)
4. [💾 Transactions & Storage Layout](#-transactions--storage-layout)
5. [🚨 Failures, Retries & DLQ Handling](#-failures-retries--dlq-handling)
6. [🧠 Operational Tips & Monitoring](#-operational-tips--monitoring)
7. [📘 Quick Reference Cheat Sheet](#-quick-reference-cheat-sheet)

---

## 🧩 Core Concepts & Architecture

> 🎯 *Mental Model:* Kafka is like a durable postal system for data — producers write messages, brokers deliver them, and consumers pick them up when ready.

![Kafka Architecture](/images/kafka-architecture.png)

```mermaid
flowchart LR
  subgraph Cluster
    P1["Partition 1<br>(Leader)"]
    P2["Partition 2<br>(Follower)"]
    P3["Partition 3<br>(Follower)"]
  end

  PRD["Producer"] -->|write| P1
  P1 -->|replicate| P2
  P1 -->|replicate| P3
  C1["Consumer 1"] -->|read| P1
  C2["Consumer 2"] -->|read| P2
```

| Concept            | Quick Definition  | Why It Matters                  |
| ------------------ | ----------------- | ------------------------------- |
| **Topic**          | Named data stream | Producers write, consumers read |
| **Partition**      | Slice of a topic  | Enables scaling                 |
| **Broker**         | Kafka server      | Stores partitions               |
| **Consumer Group** | Set of consumers  | Ensures balanced consumption    |
| **Offset**         | Message position  | Enables replay/resume           |
| **Replication**    | Data copies       | Survives failure                |

> 💡 *Tip:* Kafka = “Google Drive for events.” Upload (produce) → Store → Download (consume).

[⬆️ Back to Top](#-kafka-complete-guide)

---

## 🎛️ Control Knobs & Configuration

> ⚙️ *Goal:* Tune Kafka between durability 🔒, performance ⚡, and availability 🌍.

![Kafka Control Knobs](../images/kafka-replication.png)

```mermaid
flowchart LR
  P1["Producer 1"] --> T1["Topic A<br>(Partition 0–2)"]
  P2["Producer 2"] --> T1
  T1 -->|"replicated"| B1["Broker 1"]
  T1 -->|"replicated"| B2["Broker 2"]
  T1 -->|"replicated"| B3["Broker 3"]
  B1 -->|"deliver"| C1["Consumer 1"]
  B2 -->|"deliver"| C2["Consumer 2"]
  B3 -->|"deliver"| C3["Consumer 3"]
```

| Knob                      | Safer Setting          | Pros                   | Cons                |
| ------------------------- | ---------------------- | ---------------------- | ------------------- |
| **Replication Factor ↑**  | 3+                     | Survives broker loss   | More disk & network |
| **min.insync.replicas ↑** | 2+                     | Guarantees consistency | May reject writes   |
| **acks=all**              | Waits for all replicas | Strong durability      | Higher latency      |

> 💡 *Tip:* Use `acks=all` + `min.insync.replicas=2` for production safety.
> ⚠️ *Watch out:* Don’t over-tighten these in dev — you’ll slow yourself down.

[⬆️ Back to Top](#-kafka-complete-guide)

---

## 🔒 Messaging Guarantees

> 🧠 *Question:* What’s safer — sending fast or never losing data? Kafka lets you choose.

![Kafka Guarantees](../images/kafka-guarantees.png)

| Guarantee         | Behavior               | Use When           |
| ----------------- | ---------------------- | ------------------ |
| **At-most-once**  | Message may be lost    | Telemetry, metrics |
| **At-least-once** | Message delivered ≥1   | Notifications      |
| **Exactly-once**  | Message once, no dupes | Payments, orders   |

```mermaid
sequenceDiagram
  participant P as Producer
  participant K as Kafka
  participant C as Consumer
  P->>K: Send message
  K-->>C: Deliver message
  C->>K: Commit offset after processing
```

> 💡 *Tip:* Start with *at-least-once* and dedupe by key.
> ⚠️ *Watch out:* Exactly-once needs `enable.idempotence=true` and transactions.

[⬆️ Back to Top](#-kafka-complete-guide)

---

## 💾 Transactions & Storage Layout

> 🧭 *Goal:* Understand how Kafka achieves atomicity — all or nothing.

![Kafka Transactions](../images/kafka-transactions.png)

```mermaid
sequenceDiagram
  participant P as Producer (txn-enabled)
  participant CO as Txn Coordinator
  participant TP as Topic Partition
  participant C as Consumer (read_committed)
  P->>CO: BeginTransaction
  P->>TP: Write records
  TP-->>P: Acknowledge
  P->>CO: CommitTransaction
  CO->>TP: Append COMMIT marker
  Note over TP,C: LSO advances — committed data visible
  TP-->>C: Fetch records ≤ LSO
```

| Term              | Definition                         | Analogy             |
| ----------------- | ---------------------------------- | ------------------- |
| **LSO**           | Last Stable Offset                 | End of visible data |
| **HW / LEO**      | High Watermark / Replication point | Durability fence    |
| **Commit Marker** | Control record for visibility      | Checkmark at end    |

> 💡 *Tip:* HW ≠ visible — visibility controlled by LSO.
> ⚠️ *Watch out:* Long-running transactions block consumers until commit.

[⬆️ Back to Top](#-kafka-complete-guide)

---

## 🚨 Failures, Retries & DLQ Handling

> 💥 *Goal:* Handle retries safely and isolate bad data.

![Kafka Retries](../images/kafka-retries.png)

```mermaid
flowchart LR
  IN["Main Topic"]
  W["Worker"]
  OK["Success<br>Commit Offset"]
  R1["Retry 5s"]
  R2["Retry 1m"]
  R3["Retry 10m"]
  DLQ["Dead Letter Queue"]

  IN --> W
  W -->|success| OK
  W -->|fail| R1
  R1 --> W
  W -->|fail again| R2
  R2 --> W
  W -->|fail again| R3
  R3 --> W
  W -->|fail max| DLQ
```

> ⚠️ *Watch out:* Poison messages can block partitions — isolate with tiered retries.
> 💡 *Tip:* Add headers like `errorType`, `attempt`, `stacktrace` for DLQ analytics.

[⬆️ Back to Top](#-kafka-complete-guide)

---

## 🧠 Operational Tips & Monitoring

> 📊 *Goal:* Keep Kafka healthy and predictable.

| Area         | Metric                        | Why It Matters             |
| ------------ | ----------------------------- | -------------------------- |
| Producer     | `record-error-rate`           | Detect overload            |
| Consumer     | `lag`, `rebalance-count`      | Find bottlenecks           |
| Broker       | `under-replicated-partitions` | Detect instability         |
| Transactions | `txn-abort-rate`              | Reveal coordination issues |

> 💡 *Tip:* Monitor **Consumer Lag vs LSO** — if it widens, consumers are behind commits.
> ⚙️ *Pro Move:* Auto-heal stuck consumers by rebalancing groups on lag threshold.

[⬆️ Back to Top](#-kafka-complete-guide)

---

## 📘 Quick Reference Cheat Sheet

> 🧾 *Use this as your “first 10 seconds before panic” guide.*

| Goal           | Key Settings                        | Notes                               |
| -------------- | ----------------------------------- | ----------------------------------- |
| Safe writes    | `acks=all`, `min.insync.replicas=2` | Most reliable baseline              |
| Deduplication  | `enable.idempotence=true`           | Avoid duplicate sends               |
| Exactly-once   | + `transactional.id`                | Enables atomic offset + sink commit |
| Debugging lag  | Check consumer offsets              | Look for rebalances                 |
| Retry strategy | Tiered topics                       | Avoid partition blocking            |

> 💡 *Tip:* Kafka doesn’t lose data — you just have to tell it how patient to be.
> 🧩 *Mnemonic:* “Acks, Replicas, Transactions = ART of durability.”

[⬆️ Back to Top](#-kafka-complete-guide)
