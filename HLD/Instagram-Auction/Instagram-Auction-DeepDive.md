# 🧠 Instagram Auctions — Deep Dive System Design (Meta Interview Grade)

> Scope: **Auction feature only** — create auctions, place bids fairly, realtime updates, and reliable end-of-auction. (Payments and post-auction flows are out of scope.)

---

## 1️⃣ Functional Requirements

**Creator**
- Create a time-boxed auction on a post with: `startAt`, `endAt`, `startPrice`, `minIncrement`.
- Start, pause (optional v2), and end auctions; view current state and history.

**Bidder**
- Place a bid ≥ `currentPrice + minIncrement` while auction is active.
- Get immediate accept/reject with latest state; retries must be **idempotent**.

**Viewer**
- Subscribe to live updates (highest bid, winner, countdown).
- Read auction details and historical bids.

**System**
- Automatically end auctions at `endAt` and seal further bids.
- Expose APIs to fetch auction state and bid history.

---

## 2️⃣ Non‑Functional Requirements

- **Fairness & Consistency:** Exactly one highest bid at any instant per auction (atomic decision).
- **Low Latency:** P50 < 150 ms, P99 < 500 ms for bid acknowledgement under spike.
- **Scalability:** Thousands of simultaneous auctions; hot auctions with 100k+ viewers.
- **Durability:** No loss of accepted bids; auditable history.
- **Reliability:** No missed end events (timer guarantees); safe retries (idempotency).
- **Observability:** Metrics for latency, CAS conflicts, Kafka lag, end-event SLA.
- **Security:** Authn at gateway, authz by role (creator, bidder, viewer).

---

## 3️⃣ Core Entities & Relationships

```mermaid
erDiagram
  USER ||--o{ AUCTION : creates
  USER ||--o{ BID : places

  AUCTION ||--o{ BID : has

  USER {
    bigint id PK
    text handle
  }

  AUCTION {
    bigint id PK
    bigint post_id
    bigint creator_id FK
    timestamptz start_at
    timestamptz end_at
    numeric start_price
    numeric min_increment
    text status  // scheduled|active|ended
    numeric current_price
    bigint current_winner_id
    bigint version
    timestamptz created_at
  }

  BID {
    bigint id PK
    bigint auction_id FK
    bigint bidder_id FK
    numeric amount
    timestamptz created_at
    text status  // accepted|rejected|late|invalid
    text reason
    bigint version
    text idempotency_key
  }
```

**Notes**
- `version` increments on every accepted bid; clients apply **last-version-wins**.
- `idempotency_key` dedupes client retries per `(auction_id, bidder_id)`.

---

## 4️⃣ API Design (Auction Feature Only)

### Create Auction
```
POST /auctions
Body: {
  postId: string,
  startAt: ISO8601,
  endAt: ISO8601,
  startPrice: number,
  minIncrement: number
}
Resp: { auctionId: string }
Owner: Auction Service
```

### Get Auction
```
GET /auctions/{id}
Resp: {
  id, postId, creatorId,
  status, startAt, endAt,
  currentPrice, currentWinnerId, minIncrement, version
}
Owner: Auction Service
```

### Place Bid (Idempotent)
```
POST /auctions/{id}/bids
Headers: { Idempotency-Key: uuid }
Body: { amount: number }
Resp (accepted): { status:"accepted", currentPrice, currentWinnerId, version }
Resp (rejected): { status:"rejected", reason, currentPrice, currentWinnerId, version }
Owner: Bidding Service
```

### Stream Updates (Realtime)
```
GET /auctions/{id}/stream     // Server-Sent Events or WebSocket
data: {
  type: "bid"|"tick"|"ended",
  auctionId, currentPrice, winnerId, version, remainingSeconds?
}
Owner: Realtime Gateway
```

---

## 5️⃣ High‑Level Design & Rationale

```mermaid
graph TD
  U[User App] --> GW[API Gateway]
  U --> RT[Realtime Gateway]
  GW --> BS[Bidding Service]
  GW --> AS[Auction Service]
  BS --> R[Redis (atomic CAS)]
  BS --> PG[Postgres (ledger)]
  BS --> K[Kafka (events)]
  K --> FO[Fanout Service]
  FO --> RT
  AS --> SCH[Scheduler]
  SCH --> AS
  AS --> R
  AS --> PG
  AS --> K
```

**Why these components?**
- **Redis (Lua CAS):** atomic winner decision under contention; sub‑ms path.
- **Postgres:** authoritative ledger for auctions and bids (ACID, audit).
- **Kafka:** ordered, replayable events driving fanout and recovery.
- **Scheduler:** reliable end-of-auction triggers with a DB guard job.
- **Fanout + Realtime:** decouple compute from delivery; push deltas to clients.

---

## 6️⃣ End‑to‑End Flow (Bid & End)

**Bid Flow**
1) App → **API Gateway** → **Bidding Service** with `Idempotency-Key`.
2) **Bidding** reads hot state from **Redis**; validates time + amount.
3) **Lua CAS**: if `amount >= current + increment` → set new price & winner; bump `version`. Else reject with latest state.
4) If accepted: write **Bid** row and update **Auction** in **Postgres**; publish `BidAccepted` to **Kafka**.
5) **Fanout** consumes and pushes compact delta to **Realtime**; clients update by `version`.

**End Flow**
1) **Auction Service** schedules end on **Scheduler** at `endAt`.
2) Scheduler triggers `end-auction` → **Auction Service** seals **Redis** (no more CAS accepts).
3) Finalize in **Postgres** (`status=ended`, final `winner_id`, `current_price`); publish `WinnerDeclared`.
4) **Fanout** notifies clients (`type="ended"`).

---

## 7️⃣ Consistency, Idempotency, Ordering

- **Per‑auction consistency:** Redis key `auction:{id}` is the single writer via CAS.
- **Idempotency:** `(auction_id, bidder_id, idempotency_key)` unique index returns the first result on retries.
- **Ordering:** Kafka **keyed by `auction_id`** preserves event order to Fanout.
- **Client updates:** monotonic `version` → drop older frames, render latest only.

---

## 8️⃣ Scaling & Reliability (What interviewers probe)

**Scaling**
- Redis and Kafka **sharded by `auction_id`** to isolate hot auctions.
- Fanout workers scale per channel; Realtime coalesces updates to keep latency low.
- DB partitioning by time or creator to maintain write throughput.

**Reliability**
- **Dual timers:** delay queue + DB guard prevent missed ends.
- **Reconciler:** repairs Postgres vs Redis drift using Kafka tail.
- **Redis rebuild:** if cache loss, reconstruct from Postgres + latest events.
- **At‑least‑once fanout:** safe via version monotonicity.

---

## 9️⃣ Key Trade‑offs (Why this over alternatives)

| Area | Choice | Why | Trade‑off |
|------|--------|-----|-----------|
| Bid coordination | Redis Lua CAS | Fast atomicity under spikes | Needs reconcilers for drift |
| Source of truth | Postgres | ACID, auditability, simple queries | Higher write latency than KV |
| Realtime transport | SSE (start), WS (hot rooms) | Simpler, CDN friendly; upgrade as needed | SSE is one‑way |
| Fanout backbone | Kafka | Ordered, replayable, decouples compute | Operational overhead |
| Timer strategy | Scheduler + DB guard | Never miss end events | Extra moving parts |

---

## 🔟 Interview TL;DR (30 seconds)

“**Bids** go Gateway → Bidding → **Redis CAS** (atomic winner) → Postgres → Kafka → Fanout → Realtime → clients.  
**Ends** are triggered by Scheduler; Auction Service seals Redis and finalizes DB.  
Redis = fairness & speed; Postgres = truth; Kafka = order & replay; Scheduler = reliability.”
