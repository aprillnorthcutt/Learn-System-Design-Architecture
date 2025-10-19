
# 🌍 i18n System Design — Final Architecture (Meta Interview Ready)

This document captures the **final, interview-grade architecture** for the internationalization (i18n) system design question, as asked at Meta and similar companies.  
It separates **user-facing reads (🟢)** from **backend update propagation (🔵)** to ensure global scalability, consistency, and sub-50ms latency.

---

## 🧭 System Overview

```mermaid
graph LR
  %% ==== READ PATH (Client-facing, GREEN) ====
  A[Client]
  B[CDN Edge]
  C[API Gateway]
  D[Redis Cache]
  E[Postgres DB]

  %% Read flow (only green)
  A -->|GET /i18n| B
  B -->|cache miss| C
  C -->|MGET locale_keys| D
  D -->|cache miss| E
  E -->|query result| D
  D -->|values and version| C
  C -->|bundle| B
  B -->|localized response| A

  %% ==== UPDATE PATH (Backend-only, BLUE) ====
  T[Translator UI]
  F[CDC Debezium]
  G[Kafka translation_events]
  H[Redis Updater]
  X[CDN Control]

  %% Update flow - never touches Client or CDN Edge
  T -->|POST /translations| C
  C -->|write translation| E
  E -->|emit change| F
  F -->|publish event| G
  G -->|update cache| H
  H -->|refresh Redis| D
  G -->|purge request| X

  %% ==== LINK COLORS ====
  %% Read Path - green (edges 0..7)
  linkStyle 0 stroke:#228B22,stroke-width:2px;
  linkStyle 1 stroke:#228B22,stroke-width:2px;
  linkStyle 2 stroke:#228B22,stroke-width:2px;
  linkStyle 3 stroke:#228B22,stroke-width:2px;
  linkStyle 4 stroke:#228B22,stroke-width:2px;
  linkStyle 5 stroke:#228B22,stroke-width:2px;
  linkStyle 6 stroke:#228B22,stroke-width:2px;
  linkStyle 7 stroke:#228B22,stroke-width:2px;

  %% Update Path - blue (edges 8..14)
  linkStyle 8 stroke:#1E90FF,stroke-width:2px;
  linkStyle 9 stroke:#1E90FF,stroke-width:2px;
  linkStyle 10 stroke:#1E90FF,stroke-width:2px;
  linkStyle 11 stroke:#1E90FF,stroke-width:2px;
  linkStyle 12 stroke:#1E90FF,stroke-width:2px;
  linkStyle 13 stroke:#1E90FF,stroke-width:2px;
  linkStyle 14 stroke:#1E90FF,stroke-width:2px;
```

---

## 🟢 Read Path — User-Facing Flow

**Purpose:** Deliver localized UI strings in <50ms.

1. Client requests `/i18n?locale=fr&surface=home`.
2. CDN Edge returns cached bundle if present.
3. On miss → API Gateway queries Redis.
4. Redis returns hot keys; if cache miss → Postgres lookup.
5. Postgres result rehydrates Redis and returns to client via CDN.
6. Clients see atomic, versioned bundles — never mixed locales.

**Tech stack:** CDN (Cloudflare/Akamai), Redis Cluster, Postgres, API Gateway.

---

## 🔵 Update Path — Backend Propagation

**Purpose:** Refresh caches and CDN bundles after translation edits.

1. Translator UI posts a new translation via `/translations`.
2. API writes to Postgres (versioned translation table).
3. CDC (Debezium) detects change → sends to Kafka.
4. Kafka publishes `translation_changed` events.
5. Redis Updater consumes event → refreshes keys.
6. CDN Control API invalidates stale edge bundles.

**Important:** This path **never touches the client** — it prepares data for future reads.

---

## 🧩 Key Interview Insights

| Principle | Explanation |
|------------|-------------|
| **Separation of concerns** | Green path = reads; Blue path = backend-only updates. |
| **Low latency** | CDN + Redis = sub-50ms p95 response. |
| **Event-driven freshness** | CDC + Kafka keeps caches and CDN updated in near-real-time. |
| **Atomic consistency** | Versioned bundles prevent mixed-language UI. |
| **No client fan-out** | Users only get updated data on their next request. |

---

## 🗣️ How to Explain This at Meta

> “We decouple translation delivery from translation updates.  
> The read path serves localized bundles via CDN and Redis for <50ms p95 latency.  
> The update path runs asynchronously — when translators update text, CDC and Kafka propagate changes to Redis and CDN, ensuring global consistency without touching clients directly.  
> This design scales linearly, prevents fan-out, and guarantees atomic locale bundles per surface.”

---

🟢 **Green = Serves user requests**  
🔵 **Blue = Propagates updates (no client contact)**

---

**Author:** April Northcutt  
**Prepared for:** Meta / FAANG System Design Interview Practice  
**Filename:** `README_i18n_MetaReady.md`

---

🧠 1. What the interviewer is asking

When they say:

“Design an i18n system.”

They mean:
Design the backend platform that delivers translated system UI strings (like buttons, menus, labels, notifications) to users all over the world — fast, consistently, and safely.

It’s not a language algorithm problem.
It’s a distributed systems and scalability problem.


---


💡 2. Why this problem exists

Facebook, Instagram, WhatsApp, etc. serve billions of users.
Every button and label (“Home,” “Share,” “Create Post”) must appear instantly in the right language — without deploying new code.

So your system needs to:

Serve localized UI content at <50ms p95 latency
> - Handle hundreds of languages
- Allow non-engineers (translators) to make updates
- Propagate changes globally in real time
- Avoid stale, mixed-language screens

---


🧩 3. What they want you to design (core components)
| Layer                          | Function                                                     | Tech Examples      |
| ------------------------------ | ------------------------------------------------------------ | ------------------ |
| **Client**                     | Fetches localized strings by locale (e.g., `en_US`, `fr_FR`) | HTTP API           |
| **CDN Edge**                   | Caches bundles per locale and surface                        | Akamai, Cloudflare |
| **Redis Cache**                | Regional cache for hot translations                          | Redis cluster      |
| **Database (Source of Truth)** | Stores all translations + versions                           | PostgreSQL         |
| **CDC / Kafka**                | Streams translation changes to invalidate caches             | Debezium + Kafka   |
| **Admin Tool**                 | Used by translators to edit text                             | Internal UI        |

---


⚙️ 4. What “design” means here

You must explain how to:
- Separate content from code — avoid hardcoding strings.
- Model data — key, locale, value, version.
- Design APIs — /i18n/{locale}/{surface} and /i18n/translations.
- Build caching layers — CDN → Redis → DB.
- Handle updates — CDC → Kafka → Redis → CDN.
- Support fallbacks — fr_CA → fr → en.
- Scale globally — replication, versioning, event propagation.

---


🧠 5. What interviewers are really testing
| Skill                     | What they’re evaluating                                                          |
| ------------------------- | -------------------------------------------------------------------------------- |
| **System decomposition**  | Can you identify and separate translation, caching, delivery, and update layers? |
| **Scalability & latency** | Do you use multi-tier caching (Redis + CDN)?                                     |
| **Consistency model**     | How do you keep translations fresh but not stale?                                |
| **Resilience**            | What happens if Redis or CDN fails?                                              |
| **Versioning strategy**   | How do you avoid half-new/half-old screens?                                      |
| **Real-world awareness**  | Do you consider non-engineer workflows (translators)?                            |


---


🧭 6. The standard design flow in an interview

> 1.	Clarify requirements.
Ask: “System UI only? Dynamic messages? Real-time updates?”
2.	Define functional & non-functional goals.
3.	Propose architecture.
Sketch Postgres → Redis → CDN → Client.
4.	Explain read path (hot path).
5.	Explain write/update path.
6.	Discuss propagation (CDC + events).
7.	Cover fallbacks, pluralization, ICU.
8.	Add metrics & versioning.
9.	Address trade-offs.


---


🧱 7. Example interview summary

“We separate content from code and store translations in a versioned DB. <br>
At runtime, clients fetch localized bundles from CDN with Redis and DB fallbacks. <br>
Updates propagate via CDC → Kafka to invalidate caches globally. <br>
We maintain p95 under 50ms with layered caching and atomic bundle versioning, ensuring consistent UI in any language.” <br>
