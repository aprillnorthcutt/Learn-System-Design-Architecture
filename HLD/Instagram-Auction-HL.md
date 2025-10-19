# Instagram Auctions — Beginner Friendly System Design

> Learn how an online auction feature (like Instagram Auctions) works from end-to-end — designed for clarity, fairness, and real-time updates.

---

## 📚 Table of Contents
- [🎯 Problem Statement](#-problem-statement)
- [🏗️ Architecture Overview](#️-architecture-overview)
- [⚙️ Place Bid Sequence](#️-place-bid-sequence)
- [⏲️ End Auction Sequence](#️-end-auction-sequence)
- [🧩 Bid Validation Flow](#-bid-validation-flow)
- [💾 Data Model Overview](#-data-model-overview)
- [🌐 API Surface](#-api-surface)
- [🧠 Beginner Takeaways](#-beginner-takeaways)

---

## 🎯 Problem Statement
Enable creators to run **time-boxed auctions on posts** where:
- Only **one valid highest bid** is accepted at any moment.
- Viewers see **instant updates** as bids come in.
- The auction **always closes correctly** even under high traffic.
- **Fairness, scalability, and durability** are guaranteed.

---

## 🏗️ Architecture Overview

<details>
<summary>Beginner System Overview</summary>

```mermaid
graph TD
  %% CLIENT
  subgraph Client
    APP["User App (iOS / Android / Web)"]
  end

  %% EDGE
  subgraph Edge
    GW["API Gateway"]
    RT["Realtime Gateway (SSE or WebSocket)"]
  end

  %% CORE SERVICES
  subgraph Core_Services["Core Services"]
    AS["Auction Service - creates and ends auctions"]
    BS["Bidding Service - validates bids and decides winner"]
    FO["Fanout Service - streams updates to viewers"]
    PAY["Payment Service - handles winner checkout"]
  end

  %% DATA & INFRA
  subgraph Data["Data and Infrastructure"]
    R["Redis - fast current state and atomic CAS"]
    PG["Postgres - source of truth ledger"]
    K["Kafka - ordered events and replay"]
    SCH["Scheduler - end timers and guard job"]
  end

  %% FLOWS
  APP -->|HTTP API| GW
  APP -->|Live updates| RT

  GW --> BS
  GW --> AS

  BS --> R
  BS --> PG
  BS -->|Publish event| K
  K --> FO
  FO --> RT

  AS --> SCH
  SCH --> AS
  AS --> PG
  AS -->|Winner declared| K
  K --> PAY
  PAY --> PG
```
</details>

How to read it:
•	The App talks to the API Gateway for requests and connects to Realtime Gateway for live data. <br>
•	Bidding Service is the only entry point for bids; it checks Redis for atomic fairness and logs to Postgres. <br>
•	Kafka streams all accepted bids for replay and fan-out to viewers. <br>
•	Auction Service schedules and seals auctions using a Scheduler so nothing is missed. <br>

---


⚙️ Place Bid Sequence
<details> <summary>Place Bid Sequence</summary>
  
```mermaid
  
sequenceDiagram
  participant C as Client App
  participant GW as API Gateway
  participant B as Bidding Service
  participant R as Redis atomic CAS
  participant DB as Postgres
  participant K as Kafka
  participant F as Fanout
  participant RT as Realtime Gateway

  C->>GW: POST /auctions/{id}/bids { amount, idempotencyKey }
  GW->>B: Forward request
  B->>R: Atomic compare and set on current price
  alt Accepted
    R-->>B: accepted with new price and version
    B->>DB: Insert bid and update auction row
    B->>K: Publish BidAccepted event
    K-->>F: Consume event
    F->>RT: Push update to viewers
    B-->>C: 200 accepted with latest state
  else Rejected
    R-->>B: rejected with reason and latest state
    B-->>C: 200 rejected with reason and latest state
  end
```
</details>

Takeaway:
•	Redis ensures only one winner per moment (atomic compare-and-set). <br>
•	Postgres records the durable truth. <br>
•	Kafka → Fanout → Realtime sends instant updates to watchers. <br>
