# SeatWatch — UBC Course Seat Tracker

Implementation plan, one section per commit. Built to be handed to Claude Code one commit at a time.

---

## How to use this file (for Shervin)

1. Put this file in the root of your repo as `PLAN.md` and commit it.
2. In Claude Code, start each step with:

   > Read PLAN.md. Follow the "Rules for Claude Code" section. Implement **Commit N** only. When tests pass, make the commit with the message given, then stop and give me the explanation and interview questions.

3. Read the explanation, try to answer the interview questions yourself, then `git push`.
4. Move on to the next commit.

Don't skip step 3. Recruiters will ask you about this project, and "Claude wrote it" won't survive a technical interview. If something in a commit doesn't make sense, ask Claude Code to explain it before moving on.

---

## Rules for Claude Code

- Implement **only** the commit requested. Do not start the next one.
- Keep the repo layout below. Don't rename folders or add frameworks not listed here without asking.
- Every commit must build and its tests must pass before committing:
  - API: `cd api && ./mvnw verify`
  - Worker: `cd worker && cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build && ctest --test-dir build --output-on-failure`
- Use the exact commit message given. Author is the user; no co-author lines unless the user asks.
- No secrets in the repo. Config comes from environment variables; document every new variable in `.env.example`.
- After committing, reply with:
  1. A plain-English explanation of what changed and why (under 200 words).
  2. Two or three interview questions a co-op interviewer could ask about this commit, without answers.
  3. Anything the user must do by hand (install something, set an env var).

---

## What we're building

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

**Why two languages:** Java owns the business logic and database (what Java is good at, and what co-op postings ask for). C++ owns the concurrent, rate-limited polling loop (shows threads, RAII, memory-safe resource handling). They talk over a small HTTP contract, which is how real services split work.

### Repo layout

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

### Tech choices (fixed)

| Area | Choice |
|---|---|
| API | Java 21, Spring Boot 3.x, Spring Web, Spring Data JPA, Validation, Actuator, Spring Mail |
| DB | PostgreSQL 16, Flyway migrations |
| API tests | JUnit 5, Spring Boot Test, MockMvc, Testcontainers (PostgreSQL) |
| Worker | C++17, CMake ≥ 3.20, libcurl, nlohmann/json, GoogleTest (via FetchContent) |
| Ops | Docker (multi-stage builds), docker-compose, GitHub Actions |

### The data-source question (decide before Commit 17)

UBC course registration runs on Workday, and seat counts there sit behind a student login. Don't build anything that logs in as you or scrapes behind authentication — that likely breaks UBC's terms of use and could put your account at risk. The worker talks to seat data through a `SeatSource` interface, so the whole project works and is demoable with the **mock feed** (Commit 7). Commit 17 covers what to do about real data.

---

## API contract (worker ⇄ API)

All `/internal/**` requests need header `X-Worker-Token: <WORKER_TOKEN>`; otherwise 401.

`GET /internal/sections/watched` → sections with at least one ACTIVE subscription:
```json
[{ "id": 12, "term": "2026W2", "subject": "CPSC", "course": "221", "section": "101", "seatsAvailable": 0 }]
```

`POST /internal/seat-updates`:
```json
{ "checkedAt": "2026-10-09T04:00:00Z",
  "updates": [{ "sectionId": 12, "seatsAvailable": 3 }] }
```
→ `200 {"processed": 1, "notificationsQueued": 4}`

Write this into `docs/api-contract.md` in Commit 0 and keep it current.

---

## Commit 0 — Repo skeleton

**Goal:** empty but organised repo.

- `README.md` with project name, one-paragraph pitch, "status: in progress".
- `.gitignore` for Java/Maven, CMake build dirs, IDE files (`.idea/`, `.vscode/` except `extensions.json`), `.env`.
- `LICENSE` (MIT, author Shervin Shams).
- `docs/architecture.md` with the diagram and "why two languages" paragraph above.
- `docs/api-contract.md` with the contract above.
- `.env.example` (empty section headers for now).
- Empty `api/` and `worker/` folders with a `.gitkeep`.

**Check:** `git status` clean after commit.
**Message:** `chore: initial repo structure and architecture docs`

---

## Commit 1 — Spring Boot scaffold

**Goal:** an API that starts and has a health check.

- Generate a Maven project in `api/` (group `dev.shervin`, artifact `seatwatch-api`, package `dev.shervin.seatwatch`), Java 21, with the Maven wrapper (`mvnw`).
- Dependencies: web, validation, actuator, data-jpa, postgresql, flyway-core, flyway-database-postgresql, mail, test, testcontainers (junit-jupiter, postgresql).
- Expose only `/actuator/health`.
- For this commit, exclude DataSource autoconfig in a test so the context test runs without a DB, *or* wait to add JPA/Flyway deps until Commit 2 — Claude Code's choice, but explain it.
- Test: `HealthEndpointTest` using MockMvc expects 200 and `"status":"UP"`.

**Message:** `feat(api): scaffold Spring Boot service with health check`

---

## Commit 2 — Database schema and local Postgres

**Goal:** real database with versioned schema.

- `docker-compose.yml` at repo root with only a `postgres:16` service (db `seatwatch`, user/pass from `.env`), named volume, healthcheck.
- `application.yml` reading `DB_URL`, `DB_USER`, `DB_PASSWORD` from env with local defaults.
- `src/main/resources/db/migration/V1__init.sql`:

```sql
CREATE TABLE sections (
  id               BIGSERIAL PRIMARY KEY,
  term             VARCHAR(10)  NOT NULL,   -- e.g. 2026W2
  subject          VARCHAR(8)   NOT NULL,   -- CPSC
  course           VARCHAR(8)   NOT NULL,   -- 221
  section          VARCHAR(8)   NOT NULL,   -- 101
  seats_available  INTEGER,                 -- NULL = never checked
  last_checked_at  TIMESTAMPTZ,
  UNIQUE (term, subject, course, section)
);

CREATE TABLE subscriptions (
  id                 BIGSERIAL PRIMARY KEY,
  email              VARCHAR(254) NOT NULL,
  section_id         BIGINT NOT NULL REFERENCES sections(id),
  status             VARCHAR(16) NOT NULL,  -- ACTIVE, NOTIFIED, CANCELLED
  unsubscribe_token  UUID NOT NULL UNIQUE,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  notified_at        TIMESTAMPTZ
);
CREATE UNIQUE INDEX one_active_sub_per_email_section
  ON subscriptions (email, section_id) WHERE status = 'ACTIVE';

CREATE TABLE notifications (
  id               BIGSERIAL PRIMARY KEY,
  subscription_id  BIGINT NOT NULL REFERENCES subscriptions(id),
  seats_seen       INTEGER NOT NULL,
  status           VARCHAR(16) NOT NULL,    -- PENDING, SENT, FAILED
  attempts         INTEGER NOT NULL DEFAULT 0,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  sent_at          TIMESTAMPTZ,
  last_error       TEXT
);
CREATE INDEX notifications_pending ON notifications (status) WHERE status = 'PENDING';
```

- Tests use Testcontainers Postgres through a shared `AbstractIntegrationTest` base class (`@ServiceConnection`).
- Test: context loads and Flyway applied V1.

**Message:** `feat(api): add PostgreSQL schema with Flyway and Testcontainers`

---

## Commit 3 — Subscribe / unsubscribe

**Goal:** students can watch a section.

- JPA entities `Section`, `Subscription` (enum `SubscriptionStatus`), repositories.
- `SectionCode` value object that parses input like `CPSC 221 101` or `cpsc221-101` into subject/course/section; reject anything else with a clear message. Term must match `^\d{4}W[12]$|^\d{4}S[12]?$`.
- `SubscriptionService`:
  - find-or-create the section, create ACTIVE subscription with a random `unsubscribe_token`.
  - max 10 ACTIVE subscriptions per email (409 otherwise).
  - duplicate ACTIVE (same email + section) returns the existing one, not an error.
- Endpoints:
  - `POST /api/subscriptions` `{email, term, sectionCode}` → 201 with id, section, status.
  - `GET /api/unsubscribe/{token}` → sets CANCELLED, returns a tiny confirmation (used from email links).
- `@RestControllerAdvice` that returns `{"error": "...", "field": "..."}` for validation errors.
- Tests: `SectionCodeTest` (unit, many formats incl. bad ones), `SubscriptionControllerIT` (create, duplicate, limit of 10, bad email, unsubscribe).

**Message:** `feat(api): subscribe and unsubscribe to course sections`

---

## Commit 4 — Internal endpoints for the worker

**Goal:** the worker can ask what to check and report results.

- `WorkerTokenFilter` (`OncePerRequestFilter`) guarding `/internal/**`, constant-time compare (`MessageDigest.isEqual`) against env `WORKER_TOKEN`. Fail startup if `WORKER_TOKEN` is blank outside the `test` profile.
- `GET /internal/sections/watched` (query with a JOIN, not N+1).
- `POST /internal/seat-updates`: validate, then for each update save `seats_available` and `last_checked_at`. No notifications yet.
- Tests: 401 without/with wrong token; watched list excludes sections with only CANCELLED/NOTIFIED subs; updates persist.

**Message:** `feat(api): token-protected internal endpoints for the worker`

---

## Commit 5 — Opening detection and notification queue

**Goal:** the core logic, done correctly under concurrency.

- In `SeatUpdateService`, inside one transaction per update:
  - Lock the section row (`@Lock(PESSIMISTIC_WRITE)` or `SELECT ... FOR UPDATE`).
  - An **opening** is `previous seats is 0 or NULL` and `new seats > 0`. (NULL→>0 counts, since the student subscribed because it was full; document this choice in `docs/decisions/001-opening-rule.md`.)
  - On opening: for every ACTIVE subscription of that section, insert a PENDING `notifications` row and set the subscription to NOTIFIED with `notified_at`.
  - Return count queued.
- Because subscriptions flip to NOTIFIED in the same transaction, the same opening can't alert anyone twice even if the worker sends the update twice. That's the idempotency story — explain it in the decision doc.
- Tests:
  - 0→3 queues one per active sub; 3→5 queues none; 3→0→2 queues again only for subs still ACTIVE.
  - Sending the same update twice queues once.
  - Two threads posting the same opening at once (use `ExecutorService` + `CountDownLatch`) still produce one notification per subscription.

**Message:** `feat(api): detect seat openings and queue notifications idempotently`

---

## Commit 6 — Sending email (outbox pattern)

**Goal:** turn PENDING rows into real emails, reliably.

- `NotificationSender` interface; `SmtpNotificationSender` (Spring `JavaMailSender`) and `LoggingNotificationSender` (default when `MAIL_HOST` is unset, logs the email instead).
- `NotificationDispatcher` `@Scheduled(fixedDelay = 5s)`: claim up to 50 PENDING rows with `FOR UPDATE SKIP LOCKED`, send, mark SENT or increment `attempts` and store `last_error`; after 5 failed attempts mark FAILED.
- Email: subject `Seat open: CPSC 221 101 (2026W2)`, body with seat count, a reminder that seats go fast, and the unsubscribe link built from `PUBLIC_BASE_URL`.
- Add `MAIL_HOST`, `MAIL_PORT`, `MAIL_USER`, `MAIL_PASSWORD`, `MAIL_FROM`, `PUBLIC_BASE_URL` to `.env.example`.
- Tests: with a fake sender — success marks SENT; exceptions retry then FAILED; two dispatchers in parallel never send the same row twice.

**Message:** `feat(api): outbox dispatcher for email notifications with retries`

---

## Commit 7 — Mock seat feed (dev profile)

**Goal:** something for the worker to poll, so the whole system runs without UBC data.

- Under `@Profile("dev")` only: `MockSeatFeedController`.
  - `GET /mock/seats?term=2026W2&subject=CPSC&course=221&section=101` → `{"seatsAvailable": 0}` (in-memory map, default 0).
  - `PUT /mock/seats` with the same fields + `seatsAvailable` to change it (this is how you "open" a seat in a demo).
  - Optional `chaos` setting: return 503 or a slow response a configurable % of the time, so the worker's retry logic has something to deal with.
- Test: endpoints absent without `dev` profile; present with it.

**Message:** `feat(api): dev-only mock seat feed with failure injection`

---

## Commit 8 — Web page

**Goal:** a real person can sign up without curl.

- `src/main/resources/static/index.html` + `app.js` + `styles.css`, no framework. Form: email, term (dropdown), section code; shows success or the API's error message; short "how it works" and privacy note (what's stored, how to unsubscribe).
- Accessible: labels, focus states, works on a phone.
- Simple rate limiting on `POST /api/subscriptions` per IP (e.g. Bucket4j or a small in-memory token bucket) so the form can't be spammed. Test it.

**Message:** `feat(api): signup page and per-IP rate limiting`

---

## Commit 9 — C++ worker scaffold

**Goal:** a C++ project that builds and runs tests.

- `worker/CMakeLists.txt`: C++17, warnings as errors (`-Wall -Wextra -Wpedantic -Werror`), targets `seatwatch_core` (library), `seatwatch_worker` (executable), `seatwatch_tests`.
- GoogleTest via `FetchContent`; libcurl via `find_package(CURL REQUIRED)`; nlohmann/json via `FetchContent` (so CI doesn't need extra packages).
- Options for AddressSanitizer/UBSan (`-DSANITIZE=ON`).
- `src/config.{h,cpp}`: reads `API_BASE_URL`, `WORKER_TOKEN`, `SEAT_SOURCE_URL`, `POLL_INTERVAL_SECONDS` (default 60), `MAX_REQUESTS_PER_SECOND` (default 5), `THREADS` (default 4). Missing required → clear error, exit code 2.
- `main.cpp` prints parsed config and exits.
- Tests: config parsing (defaults, bad numbers, missing token).
- `.clang-format` (Google style, 100 cols).

**Message:** `feat(worker): scaffold C++17 worker with CMake, GoogleTest and config`

---

## Commit 10 — HTTP client and JSON models

**Goal:** worker can talk to the API.

- `HttpClient` interface (`get`, `post`) returning `HttpResponse{status, body}`; `CurlHttpClient` implementation with RAII wrapper for `CURL*` and `curl_slist*` (`std::unique_ptr` with custom deleters), timeouts (connect 3s, total 10s).
- `models.h`: `Section`, `SeatUpdate`, with `from_json`/`to_json`.
- `ApiClient`: `fetchWatched()` and `postUpdates(...)`, adds `X-Worker-Token`; non-2xx throws `HttpError` with status.
- Tests using a `FakeHttpClient`: parsing a real payload, malformed JSON throws, token header sent, error status throws.

**Message:** `feat(worker): libcurl HTTP client and API models`

---

## Commit 11 — Seat source and change detection

**Goal:** know what changed since last poll.

- `SeatSource` interface: `int seatsFor(const Section&)`.
- `MockFeedSeatSource` calls `GET {SEAT_SOURCE_URL}/mock/seats?...` (URL-encode params with curl).
- `ChangeTracker`: keeps last known seats per section id; `diff(results)` returns only sections whose count changed; seeded from `seatsAvailable` in the watched list so restarts don't cause false openings. Drops sections no longer watched.
- Tests: first poll seeded, unchanged → empty, change → reported, removed section forgotten.

**Message:** `feat(worker): pluggable seat source and change tracking`

---

## Commit 12 — Thread pool and rate limiter

**Goal:** check many sections at once without hammering the source.

- `ThreadPool` (fixed size, `std::condition_variable` task queue, `submit` returns `std::future`, joins on destruction).
- `TokenBucket` rate limiter (thread-safe, `acquire()` blocks) with an injectable `Clock` so tests don't sleep.
- `PollCycle::run()`: fetch watched → for each section submit a task that acquires a token then calls the seat source → gather → diff → post only changes in one batch.
- Tests: pool runs N tasks and returns results; bucket with fake clock allows burst then spaces requests; a cycle with fake source posts only changed sections.

**Message:** `feat(worker): concurrent poll cycle with thread pool and token bucket`

---

## Commit 13 — Retries, backoff, graceful shutdown

**Goal:** survive a flaky source and stop cleanly.

- `retry(fn, maxAttempts=4)` with exponential backoff + full jitter (base 200 ms, cap 5 s); retries on network errors and 5xx/429, not on 4xx.
- A failed section is skipped this cycle and logged; never reported as 0 seats (a false "full" could cause a false opening later — explain this in a comment).
- SIGINT/SIGTERM set an atomic flag; the main loop finishes the current cycle and exits 0. Sleep between cycles is interruptible.
- Tests: retry counts, no retry on 404, jitter stays within bounds (seeded RNG), failed section excluded from updates.

**Message:** `feat(worker): retry with jittered backoff and graceful shutdown`

---

## Commit 14 — Logging and metrics

**Goal:** numbers you can measure and put on your resume honestly.

- Worker: one JSON log line per cycle: `sections`, `checked`, `failed`, `changes`, `durationMs`, `retries`.
- API: Micrometer counters `seatwatch.openings`, `seatwatch.notifications.sent`, `seatwatch.notifications.failed`, timer for seat-update handling; expose `/actuator/prometheus`.
- API: record `detected_at` on notifications (migration `V2__notification_latency.sql`) so you can measure time from detection to email sent.
- `scripts/latency_report.sql`: p50/p95 of `sent_at - created_at`.

**Message:** `feat: structured cycle logs and Micrometer metrics`

---

## Commit 15 — Docker and end-to-end demo

**Goal:** `docker compose up` runs everything.

- `api/Dockerfile`: multi-stage (Maven build → `eclipse-temurin:21-jre`), non-root user.
- `worker/Dockerfile`: multi-stage (build with cmake + libcurl dev → slim runtime with libcurl only), non-root.
- `docker-compose.yml`: postgres, api (profile `dev`), worker, and `mailpit` (fake SMTP inbox at http://localhost:8025) so you can see emails locally.
- `scripts/demo.sh`: subscribe two emails to `CPSC 221 101`, set mock seats to 0, wait one cycle, set seats to 3, wait, then print the Mailpit inbox count. Exit non-zero if no email arrived.

**Message:** `build: dockerize api and worker with end-to-end demo`

---

## Commit 16 — GitHub Actions CI

**Goal:** green checkmark on every push.

- `.github/workflows/ci.yml` with jobs:
  - `api`: Temurin 21, Maven cache, `./mvnw verify` (Testcontainers works on `ubuntu-latest`).
  - `worker`: install `libcurl4-openssl-dev`, configure with `-DSANITIZE=ON`, build, `ctest`.
  - `e2e` (needs both): `docker compose up -d --build`, run `scripts/demo.sh`, `docker compose down`.
- README badge for CI.

**Message:** `ci: build, test and end-to-end check on every push`

---

## Commit 17 — Real data source (decision first)

**Stop and decide with the user before writing code.** Options, in order of preference:

1. **Ask.** Email UBC IT / Enrolment Services or the AMS asking whether a public seat-availability feed exists or whether a student tool may poll one. Record the answer in `docs/decisions/002-data-source.md`.
2. **A public, unauthenticated source** UBC explicitly allows. Implement it as a new `SeatSource` with a conservative rate limit (≤1 request/sec total) and a clear `User-Agent` with a contact email.
3. **No permitted source:** keep the mock feed, and make the project about the engineering. Say so plainly in the README. That's still a strong project; don't fake usage numbers.

Never: log in with your UBC credentials, store other students' credentials, or bypass rate limits.

**Message (if implemented):** `feat(worker): <source name> seat source`

---

## Commit 18 — Deploy

**Goal:** a public URL.

- Pick one: Render (easiest — web service for the API, background worker for the C++ worker, managed Postgres) or AWS (EC2 + RDS, more resume value, more work).
- `docs/deploy.md` with exact steps and every env var.
- Real SMTP: a transactional email provider's free tier; set SPF/DKIM so alerts don't land in spam.
- Only deploy publicly once Commit 17 has a permitted real source; otherwise deploy the demo with the mock feed and label it "demo".

**Message:** `docs: deployment guide`

---

## Commit 19 — README polish

- What it does (2 sentences), screenshot of the signup page and a sample email, architecture diagram, how to run locally (`docker compose up` + demo script), design decisions with links to `docs/decisions/`, measured numbers from Commit 14, what you'd do next.
- Remove "in progress".

**Message:** `docs: complete README with architecture and results`

---

## Resume bullets (fill in only with real numbers)

- Built a Java/Spring Boot and C++ service that monitors UBC course sections and emails students within **[measured p95]** of a seat opening.
- Designed idempotent opening detection with row-level locking and a transactional outbox (`FOR UPDATE SKIP LOCKED`), preventing duplicate alerts under concurrent updates.
- Wrote a multithreaded C++17 poller with a token-bucket rate limiter and jittered exponential backoff; **[N]** JUnit and GoogleTest tests run in GitHub Actions with sanitizers and a Docker end-to-end check.
