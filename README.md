# SeatWatch 🪑

**Never miss a seat opening again.**

Alerts UBC students the instant a full course section opens up — a Java/Spring Boot API paired with a concurrent, rate-limited C++ polling worker.

![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1-brightgreen?logo=spring&logoColor=white)
![C++](https://img.shields.io/badge/C%2B%2B-17-blue?logo=cplusplus&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

> **Status: in progress.** This README describes the full intended system. The [Build Status](#-build-status) table below says exactly what's actually working today — nothing here is overstated.

---

## 📑 Table of Contents

[🎯 Mission](#-mission) · [✨ Key Features](#-key-features) · [🛠 Tech Stack](#-tech-stack) · [🏗 Architecture](#-architecture) · [🚦 Build Status](#-build-status) · [🚀 Getting Started](#-getting-started) · [🔌 API Contract](#-api-contract) · [📜 License](#-license)

## 🎯 Mission

**Stop refreshing. Start watching.**

Every registration period, UBC students sit there hitting refresh on a full section, hoping to catch the exact moment a seat frees up. That's two problems at once:

| ❌ Problem | 💸 Impact |
|---|---|
| Manual refreshing | Students waste hours babysitting a browser tab instead of studying |
| Narrow windows | A seat opens and fills again before anyone notices |

SeatWatch answers one question automatically: **"Did a seat just open up in the section I need?"** — and tells every student watching it, once, by email.

## ✨ Key Features

| Feature | What it does | Status |
|---|---|---|
| 📝 Subscribe by section code | Students watch any `SUBJECT COURSE SECTION` for a term via a simple web form | 🚧 Planned |
| 🔁 Idempotent opening detection | Row-locked, transactional check so the same opening can never double-notify, even under concurrent updates | 🚧 Planned |
| 📬 Outbox-pattern email delivery | Notifications are queued, retried on failure, and sent exactly once per opening | 🚧 Planned |
| ⚙️ Concurrent polling worker | C++ thread pool + token-bucket rate limiter checks many sections without hammering the source | 🚧 Planned |
| 🔄 Retry with jittered backoff | Survives a flaky seat source without false "section is full" reports | 🚧 Planned |
| 📊 Metrics & latency tracking | Micrometer counters and a detection-to-email latency report | 🚧 Planned |
| 🐳 One-command local demo | `docker compose up` runs Postgres, the API, the worker, and a fake SMTP inbox together | 🚧 Planned |
| ✅ Health check | `/actuator/health` confirms the API is alive | **Done** |

## 🛠 Tech Stack

**Languages**
- **Java 21** — API business logic, persistence, scheduling
- **C++17** — concurrent, rate-limited polling worker
- **SQL** — PostgreSQL schema, versioned with Flyway

**Frameworks & Libraries**

| API (Java) | Worker (C++) |
|---|---|
| Spring Boot 4, Spring Web, Spring Data JPA | CMake ≥ 3.20 |
| Flyway (migrations) | libcurl (HTTP) |
| Spring Mail | nlohmann/json |
| JUnit 5, MockMvc, Testcontainers | GoogleTest (via FetchContent) |

**Ops:** Docker (multi-stage builds), docker-compose, GitHub Actions

## 🏗 Architecture

```
                 ┌──────────────────────────────┐
  student ──────▶│  api/  (Java 21, Spring Boot) │──── email (SMTP)
  (web form)     │  REST API + static web page   │
                 │  PostgreSQL via Flyway        │
                 └──────────────┬───────────────┘
              GET watched list  │  ▲  POST seat updates (batch)
              (X-Worker-Token)  ▼  │
                 ┌──────────────────────────────┐
                 │ worker/ (C++17)               │
                 │ thread pool + rate limiter    │──── seat source
                 │ retry/backoff, change detect  │     (mock feed now,
                 └──────────────────────────────┘      real source later)
```

**Why two languages:** Java owns the business logic and database — what Java is good at, and what co-op postings ask for. C++ owns the concurrent, rate-limited polling loop — threads, RAII, memory-safe resource handling. They talk over a small HTTP contract, which is how real services split work. Full writeup in [`docs/architecture.md`](docs/architecture.md).

## 🚦 Build Status

Built one commit at a time from [`PLAN.md`](PLAN.md), each reviewed and understood before it's pushed.

| # | Commit | Status |
|---|---|---|
| 0 | Repo skeleton & architecture docs | ✅ Done |
| 1 | Spring Boot scaffold with health check | ✅ Done |
| 2 | PostgreSQL schema + Flyway + Testcontainers | 🚧 Planned |
| 3 | Subscribe / unsubscribe | 🚧 Planned |
| 4 | Internal endpoints for the worker | 🚧 Planned |
| 5 | Opening detection & notification queue | 🚧 Planned |
| 6 | Outbox email dispatcher | 🚧 Planned |
| 7 | Mock seat feed (dev profile) | 🚧 Planned |
| 8 | Signup web page | 🚧 Planned |
| 9–13 | C++ worker: scaffold, HTTP client, polling, retries | 🚧 Planned |
| 14 | Logging & metrics | 🚧 Planned |
| 15–16 | Docker, end-to-end demo, CI | 🚧 Planned |
| 17 | Real data source decision | 🚧 Planned |
| 18–19 | Deploy & final README | 🚧 Planned |

## 🚀 Getting Started

### Prerequisites

| Requirement | Version | Check |
|---|---|---|
| Java | 21+ | `java -version` |
| Maven | via wrapper, nothing to install | — |

### Run the API

```bash
git clone https://github.com/shamss11/SeatWatch.git
cd SeatWatch/api
./mvnw spring-boot:run
```

```bash
curl http://localhost:8080/actuator/health
# {"status":"UP"}
```

### Run the tests

```bash
cd api
./mvnw verify
```

The `worker/` C++ service, Docker compose stack, and demo script land in later commits — see [Build Status](#-build-status).

## 🔌 API Contract

The worker and API talk over a small internal HTTP contract — full spec in [`docs/api-contract.md`](docs/api-contract.md):

```json
GET /internal/sections/watched
[{ "id": 12, "term": "2026W2", "subject": "CPSC", "course": "221", "section": "101", "seatsAvailable": 0 }]
```

```json
POST /internal/seat-updates
{ "checkedAt": "2026-10-09T04:00:00Z", "updates": [{ "sectionId": 12, "seatsAvailable": 3 }] }
→ { "processed": 1, "notificationsQueued": 4 }
```

## 📜 License

MIT — see [`LICENSE`](LICENSE).

---

Built with 🪑 by Shervin Shams, using Claude Code as a pair-programming tool, one commit at a time — see [`PLAN.md`](PLAN.md) for the plan and the reasoning behind each step.
