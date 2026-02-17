# Wise Transfer Quote Service

A small, production-minded backend service that creates **transfer quotes** (rate, fee, target amount) with **idempotency guarantees**, **structured JSON logging**, and **health checks** — designed to be built and extended in short, interview-style sessions.

This repo is intentionally scoped to a realistic “payments-style” API surface: safe retries, deterministic outputs, persistence, and operational basics.

---

## What this service does

### Endpoints

- `POST /v1/quotes`  
  Creates a quote for converting money from a source currency to a target currency.

  **Key behavior:** requires `Idempotency-Key` header
  - Same key + same payload → returns the **same quote** (safe retry)
  - Same key + different payload → returns `409` conflict

- `GET /healthz`  
  Returns `{ "status": "ok" }` and HTTP 200.

### Core features

- **Idempotent quote creation** via `Idempotency-Key`
- **SQLite persistence** (simple + reliable for local dev)
- **Structured JSON logs** (requestId, latency, status, etc.)
- **Dockerized** for consistent run environments
- **Automated tests** covering correctness and idempotency semantics

---

## Tech stack

- **Node.js + TypeScript**
- **Fastify** (HTTP server)
- **SQLite** (local persistence)
- **Zod** (request validation)
- **Vitest** + **supertest** (tests)

---

## Project structure

├── src
│ ├── server.ts # app bootstrap + plugins/hooks
│ ├── logger.ts # JSON logger + requestId plumbing
│ ├── db.ts # SQLite init + queries
│ ├── idempotency.ts # payload hashing + idempotency rules
│ └── routes
│ ├── healthz.ts # GET /healthz
│ └── quotes.ts # POST /v1/quotes
├── tests
│ ├── healthz.test.ts
│ └── quotes.test.ts
├── Dockerfile
├── package.json
├── tsconfig.json
└── README.md

---

## Getting started (local)

### Prerequisites

- Node.js **18+** (recommended: 20 LTS)
- npm (comes with Node)

### Install dependencies

```bash
npm install
```
Run the service (dev)
```bash
npm run dev
```


Service starts on:

`http://localhost:3000`

Healthcheck
```bash
curl http://localhost:3000/healthz
```

Expected:
```bash
{ "status": "ok" }
```

Using the API
1) Create a quote
```bash
curl -i \
  -X POST "http://localhost:3000/v1/quotes" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: demo-key-123" \
  -d '{
    "sourceCurrency": "SGD",
    "targetCurrency": "EUR",
    "sourceAmount": 100.00
  }'
```

Example response (shape will match your implementation):

```bash
{
  "id": "c4a2d0c0-1e6f-4c98-ae33-1b5e2f5a9f11",
  "sourceCurrency": "SGD",
  "targetCurrency": "EUR",
  "sourceAmount": "100.00",
  "rate": "0.68",
  "fee": "1.50",
  "targetAmount": "67.32",
  "createdAt": "2026-02-17T11:40:10.123Z"
}
```

2) Retry safely (idempotent)

Run the same request again with the same header key and body:

- You should get the same id, and amounts should match exactly.

3) Conflict example (same key, different body)
```bash
curl -i \
  -X POST "http://localhost:3000/v1/quotes" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: demo-key-123" \
  -d '{
    "sourceCurrency": "SGD",
    "targetCurrency": "EUR",
    "sourceAmount": 120.00
  }'
```

Expected:

HTTP `409`

JSON:
```bash
{ "error": "IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_PAYLOAD" }
```

Configuration

Environment variables (optional):

Variable	Default	Purpose
PORT	3000	Port to run the API
DB_PATH	./data/dev.sqlite	SQLite file location
IDEMPOTENCY_TTL_HOURS	24	TTL window for reusing same idempotency key

| Variable           | Default | Purpose     |
------------------ | ------ | -------- |
| `PORT`       | `3000`   | Port to run the API   |
| `DB_PATH` | `./data/dev.sqlite`   | SQLite file location   |
| `IDEMPOTENCY_TTL_HOURS`       | `24`    | TTL window for reusing same idempotency key      |

Example:
```bash
PORT=3000 DB_PATH=./data/dev.sqlite npm run dev
```
Logging

The service emits one JSON log line per request, e.g.
```bash
{
  "level": "info",
  "requestId": "req_01HT....",
  "method": "POST",
  "path": "/v1/quotes",
  "status": 201,
  "latencyMs": 12,
  "idempotencyKey": "demo-key-123"
}
```

This is meant to be:

machine-parseable (log pipelines)

safe (avoid PII)

useful (requestId + latency + outcome)

Running tests
```bash
npm test
```

Or watch mode:
```bash
npm run test:watch
```
What tests cover

- Quote creation returns required fields
- Retry returns the same quote for same key + same payload
- Conflict for same key + different payload
- Healthcheck returns OK

Running with Docker
Build image
```bash
docker build -t wise-transfer-quote-service .
```

Run container
```bash
docker run --rm -p 3000:3000 wise-transfer-quote-service
```

Healthcheck
```bash
curl http://localhost:3000/healthz
```

Note: If your SQLite DB is stored inside the container filesystem, it will be ephemeral.
To persist locally, bind mount a host directory (optional):

```bash
docker run --rm \
  -p 3000:3000 \
  -e DB_PATH=/app/data/dev.sqlite \
  -v "$(pwd)/data:/app/data" \
  wise-transfer-quote-service
```

How idempotency works (implementation notes)

On POST /v1/quotes:

1. Read Idempotency-Key header (required)
2. Hash the request payload (e.g., stable JSON stringify + SHA-256)
3. Look up existing record by idempotency key:
    - If not found → compute quote, store, return 201
    - If found and payloadHash matches → return stored quote (safe retry)
    - If found and payloadHash differs → return 409 conflict

This mirrors patterns used in payment/transfer systems where retries must not double-create state.

License

MIT (or your preferred license).
