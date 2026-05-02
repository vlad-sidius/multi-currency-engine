# multi-currency-engine

> Technical Specification & Iterative Development Roadmap  
> A production-grade fintech service demonstrating high-load architecture, financial accuracy, async resilience, and operational tooling.

---

## 📖 1. Goal & Context
Develop a modular service for automating multi-currency settlements, mutual obligation netting, batch generation, and double-entry bookkeeping. The system must guarantee financial accuracy, strict idempotency, full auditability, and resilience during partial failures of external FX rate providers or message brokers.

---

## 🧱 2. Architecture & Components

| Component | Purpose | Stack |
|-----------|---------|-------|
| **HTTP Gateway** | Payment intake, admin operations, health/metrics, UI serving | `gin`, `cors`, `promhttp` |
| **gRPC Internal Services** | Low-latency intra-cluster calls: FX converter, ledger core | `grpc-go`, `protoc-gen-go` |
| **Async Engine** | Async payout processing, netting triggers, retry handling | `amqp091-go` (RabbitMQ), DLX, retry+jitter, circuit breaker |
| **Data Layer** | Double-entry ledger, FX rates, batches, audit, complex analytics | `pgx/v5`, `pgxpool`, `goose`/`golang-migrate`, `shopspring/decimal` |
| **Admin UI** | Back-office for rates, batches, manual adjustments, reporting | React 18+, TS, Vite, TanStack Query, Zustand, React Table |
| **CLI / TUI** | Engineer tooling: dry-run, balance checks, export, diagnostics | `cobra`, `bubbletea`, `charm/bubbles` |
| **Observability** | Logs, metrics, traces, graceful shutdown | `slog`, Prometheus, OpenTelemetry (optional), `pprof` |

---

## ✅ 3. Functional Requirements
- Intake payout requests with `Idempotency-Key` and strict payload validation
- Currency conversion using live FX rates with caching and fallback to last known valid rate
- Bilateral/multilateral netting over configurable periods (hourly/daily)
- Settlement batch generation with unique `batch_id` assignment
- Double-entry ledger (`debit`/`credit`) for all financial movements
- Admin panel: rate management, batch review/approval, discrepancy resolution, audit history
- CLI: calculation simulation, ledger integrity checks, CSV/JSON export, queue lag inspection
- Manual batch reprocessing trigger for partner rejections
- Strict idempotency across all monetary operations

---

## ⚙️ 4. Non-Functional Requirements
- **Precision:** All monetary fields use `DECIMAL(19,4)` or `shopspring/decimal`. `float64` strictly prohibited for financial math.
- **Transaction Isolation:** `REPEATABLE READ` for netting calculations, `SERIALIZABLE` for final batch settlement.
- **Resilience:** Circuit breaker on external FX APIs, retry with exponential backoff + jitter, DLQ for failed messages.
- **Performance:** `pgxpool` tuned (`MaxConns`, `MinConns`, `MaxConnLifetime`, `MaxConnIdleTime`). Concurrent processing via `errgroup` + `context.WithTimeout`.
- **Testing:** Table-driven unit tests, `testcontainers-go` for PG+RabbitMQ, `-race` in CI, coverage `>75%`.
- **Deployment:** Self-contained Docker image, `docker-compose` for local env, `k8s/` manifests with probes and resource limits.
- **Observability:** Structured JSON logs, custom metrics (latency histograms, mismatch counters), `/health`, `/ready`, `/metrics`.

---

## 🗄️ 5. Key Data Models (Simplified)

| Table | Purpose | Key Features |
|-------|---------|--------------|
| `currencies` | Currency registry | `iso_code` PK, `is_active` |
| `exchange_rates` | FX conversion rates | `versioned`, `valid_from/to`, composite index `(base, quote, valid_from)` |
| `transactions` | Double-entry ledger | `id`, `batch_id`, `account_id`, `direction`, `amount`, `currency`, `idempotency_key` (unique) |
| `netting_batches` | Settlement batches | `id`, `period_start`, `period_end`, `status`, `total_settled` |
| `settlement_items` | Netting line items | `batch_id`, `partner_id`, `currency`, `net_amount`, `fx_rate_snapshot` |
| `audit_log` | Change journal | `event_type`, `payload JSONB`, `created_at`, monthly partitioning |

---

## 🔌 6. Integrations & Protocols
- **REST (Gin):** `/api/v1/transactions`, `/api/v1/batches`, `/api/v1/rates`, `/api/v1/admin/*` (JWT)
- **gRPC:** `fx.v1.RateProvider.GetRate`, `ledger.v1.Core.PostDoubleEntry`
- **RabbitMQ:** 
  - `settlement.inbox` → incoming payout processing
  - `settlement.netting.trigger` → aggregator kickoff
  - `settlement.dlq` → retry/manual intervention
- **External Sources:** Mock FX API (dev), real providers via HTTP/gRPC (prod)
- **Admin UI:** REST + SSE (`/api/v1/stream/batches`) for real-time status
- **CLI:** HTTP/gRPC clients to running instance, local cache in `~/.settle-cli/`

---

## 🔐 7. Security & Compliance
- JWT authentication + RBAC for Admin UI (`admin`, `operator`, `viewer`)
- HMAC verification for external webhooks (if applicable)
- Immutable audit trail for all rate and batch modifications
- Secrets managed via `K8s Secrets` / `.env` (never hardcoded)
- CORS whitelist, rate-limiting on public endpoints
- PCI-DSS/AML-aligned logic (no raw PAN storage, masking, strict access controls)

---

## 📊 8. Observability & Monitoring

| Metric | Type | Purpose |
|--------|------|---------|
| `settlement_transaction_processed_total` | Counter | Total processed payouts |
| `settlement_fx_rate_hit_total` | Counter | Cache hits vs external calls |
| `settlement_batch_duration_seconds` | Histogram | Batch formation latency (p50/p95/p99) |
| `settlement_ledger_mismatch_total` | Counter | Double-entry discrepancies |
| `settlement_dlq_size` | Gauge | Retry queue depth |
| `go_gc_duration_seconds`, `http_request_duration_seconds` | Standard | Runtime profiling & SLA |

**Grafana Dashboards:** Batch SLA, DLQ state, ledger balance sync, consumer lag, memory/goroutine consumption.

---

## ⚠️ 9. Constraints & Assumptions
- Fixed currency set for portfolio scope (USD, EUR, RUB, BYN)
- FX APIs mocked in CI; `mock-fx-server` in `docker-compose`
- No real fund transfers: calculation + state machine only
- Single region, no multi-AZ topology (sufficient for pattern demonstration)
- Go version: `1.21+` (enforced in `go.mod`)

---

## 🎯 10. Project Readiness Criteria
- [ ] `make dev` boots: app, PG, RabbitMQ, Prometheus, Grafana, UI (nginx)
- [ ] All monetary operations are idempotent, tested against duplicate keys
- [ ] Netting uses `REPEATABLE READ` + window functions, `EXPLAIN ANALYZE` documented
- [ ] CLI & Admin UI consume the same backend, auth validated
- [ ] CI green: `golangci-lint` → `go test -race -cover` → `docker build`
- [ ] `README.md` includes: Mermaid architecture, UI/CLI screenshots, run instructions, API examples, profiling results

---

## 🗺️ 11. 6-Iteration Roadmap

| Iteration | Focus | Key Tasks | Artifacts | Acceptance Criteria |
|-----------|-------|-----------|-----------|---------------------|
| **1. Foundation & Ledger Core** | Structure, DB, core business logic | `go.mod 1.21+`, `.golangci.yml`, `Makefile`, Clean Architecture (`cmd/`, `internal/{domain,app,infra,api}`), migrations (`goose`), `transactions`/`currencies` models, double-entry logic, basic REST endpoints | Project scaffold, working `POST /transactions`, `GET /balances`, domain unit tests, `testcontainers` integration | Double-entry balances, rollback on error, green tests, linter passes |
| **2. Async Engine & FX Rates** | RabbitMQ, rates, idempotency, resilience | Producer/consumer (`amqp091-go`), DLX, retry+jitter, FX circuit breaker, `Idempotency-Key` middleware (PG/Redis), gRPC `RateProvider`, rate caching, `pgxpool` tuning | `inbox`/`dlq` queues, idempotency middleware, rate fallback, integration tests on retries | FX fallback on failure, duplicate keys ignored, DLQ messages preserved, `-race` clean |
| **3. Netting & Batching Engine** | Complex SQL, isolation, batch formation | Netting algorithm, `REPEATABLE READ`, `SELECT ... FOR UPDATE SKIP LOCKED`, CTE + window functions (`SUM() OVER`, `RANGE BETWEEN`), partial indexes, `pg_stat_statements` analysis | `netting_batches`, `settlement_items`, background worker, dry-run simulation | Netting correctly offsets mutual obligations, no race conditions, `EXPLAIN` shows index usage |
| **4. Admin UI & CLI TUI** | Operational interfaces, real-time, UX | React/TS (Vite, Zustand, TanStack Query), batch/rate tables, SSE `/stream/batches`, JWT auth, CORS, CLI (`cobra` + `bubbletea`): `status`, `batch-preview`, `export`, `check-balances` | `web/` build, TUI commands, SSE push, nginx proxy in `docker-compose` | UI shows real-time status, CLI outputs consistent data, auth works, UI hot-reloads on `make dev` |
| **5. Observability, CI/CD & K8s** | Monitoring, pipeline, deployment | `slog` JSON, Prometheus middleware, custom metrics, `/health`/`/ready`/`/metrics`, graceful shutdown (`signal.Notify`), GitHub Actions (lint→test→build→docker), `k8s/` manifests (Deployment, Service, ConfigMap, HPA, PDB) | Grafana dashboards, CI pipeline, K8s YAML, `pprof`/`trace` profiling | `kill -15` drains goroutines cleanly, CI green, metrics collected, K8s manifests validated |
| **6. Polish & Portfolio Pack** | Documentation, optimization, packaging | `README.md` (Mermaid, run, screenshots, API examples), `ARCHITECTURE.md`, `EXPLAIN_ANALYSIS.md`, benchmarks (`hey`/`k6`), code audit, final `make lint`/`make test`, archive tutorial branches | Complete repo, documentation, screenshots, coverage report | Project reproducible in 5 commands, coverage `>75%`, all `TODO` resolved, ready for tech lead review |

---

## 💡 12. Execution Recommendations
- **Prioritize depth over speed.** Since immediate job applications aren't planned, stretch iterations to a comfortable pace. Focus on architectural justification (`why`), not just implementation (`how`).
- **Document decisions.** Maintain `ARCHITECTURE.md` and `DECISIONS.md`. Expect interview questions like: *“Why `REPEATABLE READ` over `SERIALIZABLE`?”, “Why `pgxpool` instead of `database/sql`?”, “Why RabbitMQ over Kafka here?”* Your answers should be in your head and documented.
- **Snapshot before refactoring.** Branch or save `EXPLAIN`/metric snapshots before major changes. This creates a visual progress trail for portfolio review.
- **Simulate production.** Add `prometheus`, `grafana`, `nginx` to `docker-compose`. Enforce `go test -race -coverprofile` in CI. Ban `float64` for money via lint rules or code-review checklists.
