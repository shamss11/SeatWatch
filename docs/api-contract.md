# API contract (worker ⇄ API)

All `/internal/**` requests need header `X-Worker-Token: <WORKER_TOKEN>`; otherwise 401.

## GET /internal/sections/watched

Returns sections with at least one ACTIVE subscription.

```json
[{ "id": 12, "term": "2026W2", "subject": "CPSC", "course": "221", "section": "101", "seatsAvailable": 0 }]
```

## POST /internal/seat-updates

```json
{ "checkedAt": "2026-10-09T04:00:00Z",
  "updates": [{ "sectionId": 12, "seatsAvailable": 3 }] }
```

Response:

```json
{ "processed": 1, "notificationsQueued": 4 }
```

This file is the source of truth for the worker/API HTTP boundary. Keep it current as the contract changes.
