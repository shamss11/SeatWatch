# Architecture

Students pick full UBC course sections they want. A C++ worker checks seat counts on a schedule. When a section goes from 0 seats to at least 1, the Java API emails every student watching it, once.

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

## Why two languages

Java owns the business logic and database (what Java is good at, and what co-op postings ask for). C++ owns the concurrent, rate-limited polling loop (shows threads, RAII, memory-safe resource handling). They talk over a small HTTP contract, which is how real services split work.

## Repo layout

```
seatwatch/
├── api/                 Spring Boot app (Maven wrapper)
├── worker/              C++ worker (CMake)
├── docs/                architecture.md, api-contract.md, decisions/
├── scripts/             demo.sh, load-test helpers
├── docker-compose.yml
├── .env.example
├── .github/workflows/ci.yml
└── README.md
```

## Tech choices

| Area | Choice |
|---|---|
| API | Java 21, Spring Boot 3.x, Spring Web, Spring Data JPA, Validation, Actuator, Spring Mail |
| DB | PostgreSQL 16, Flyway migrations |
| API tests | JUnit 5, Spring Boot Test, MockMvc, Testcontainers (PostgreSQL) |
| Worker | C++17, CMake ≥ 3.20, libcurl, nlohmann/json, GoogleTest (via FetchContent) |
| Ops | Docker (multi-stage builds), docker-compose, GitHub Actions |

## Data source

UBC course registration runs on Workday, and seat counts there sit behind a student login. This project never logs in as a user or scrapes behind authentication. The worker talks to seat data through a `SeatSource` interface, so the whole project works and is demoable with a mock feed. See `docs/decisions/` for the real-data-source decision once it's made.
