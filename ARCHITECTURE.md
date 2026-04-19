# Architecture

Design principles, layering model, and module boundaries for `clean-go-starter`. This is the template we bootstrap commercial Go backends from.

## Principles

1. **Business logic has no infrastructure dependencies.** Domain code does not import `net/http`, `database/sql`, or any SDK. It is pure.
2. **Dependencies point inward.** Adapters depend on use cases; use cases depend on ports; ports depend on domain. Nothing in domain depends on anything outside.
3. **Modules are strictly isolated.** Even within the same monolith, module A cannot import module B's internals. Cross-module communication happens through **published interfaces** in `port/`.
4. **Transactions span one module.** A use case runs one transaction, in one module. Cross-module coordination is **eventual** via the outbox + event bus.
5. **Every external call is mocked in tests.** No network, no DB, no real clock in unit tests.

## Layering

```
┌───────────────────────────────────────────────────────┐
│                   Adapter Layer                       │
│  HTTP handlers · DB repositories · message publishers │
└─────────────────────┬─────────────────────────────────┘
                      │ implements
                      ▼
┌───────────────────────────────────────────────────────┐
│                   Port Layer                          │
│  Interfaces: inbound (driving), outbound (driven)     │
└─────────────────────┬─────────────────────────────────┘
                      │ used by
                      ▼
┌───────────────────────────────────────────────────────┐
│                 Use Case Layer                        │
│  Application orchestration — no infra, no persistence │
└─────────────────────┬─────────────────────────────────┘
                      │ uses
                      ▼
┌───────────────────────────────────────────────────────┐
│                   Domain Layer                        │
│  Entities · Value Objects · Domain Services · Rules   │
└───────────────────────────────────────────────────────┘
```

### Domain

Entities like `Order`, `Payment`, `ActivationToken`, value objects like `Money`, `DeviceId`, and pure functions that encode invariants. No dependencies on anything outside `domain/`.

### Use Case

Orchestration. An example: `ConsumeActivationToken` loads a token, delegates invariant checks to the domain, calls the outbound port to persist, publishes an event. Does not know whether persistence is Postgres or an in-memory map.

### Port

Interfaces. Two kinds:

- **Inbound ports** (driving): what the use case can be told to do. Usually called by HTTP handlers.
- **Outbound ports** (driven): what the use case can ask for. Satisfied by adapters.

### Adapter

Concrete implementations. `pgRepository` implements `OrderRepository` outbound port using a Postgres driver. `httpHandler` implements the inbound port by decoding HTTP, calling the use case, encoding the result.

## Modules

A module is a **self-contained vertical slice** — domain + use case + ports + adapters. Each business capability lives in its own module.

```
internal/modules/
├── orders/        ── order lifecycle, pricing
├── payments/      ── payment intents, webhooks, refunds
├── activation/    ── activation tokens, device slots
└── access/        ── access-node orchestration (apply/revoke)
```

### Inter-module contracts

When `payments` needs to tell `activation` that an order has paid, it does **not** import `activation` internals. It publishes a `PaymentSucceeded` event. A handler in the `activation` module picks up the event and runs its own use case.

Enforced via:
- `internal/modules/X/...` is only importable by adapters of the same module's composition root
- `internal/modules/X/port/public` exposes cross-module interfaces (rare)
- The import linter (`tools/importlint.go`) fails CI on illegal cross-module imports

## Transactional outbox

The outbox pattern keeps business state and downstream events in sync without two-phase commit.

```
BEGIN;
  -- 1. Business state change
  INSERT INTO orders (...) VALUES (...);

  -- 2. Event row in the outbox
  INSERT INTO outbox_events (aggregate_id, event_type, payload) VALUES (...);
COMMIT;

-- Dispatcher goroutine (separate process or same binary, different goroutine):
SELECT * FROM outbox_events WHERE published_at IS NULL ORDER BY id LIMIT 100;
-- publish to internal event bus / external message broker
UPDATE outbox_events SET published_at = now() WHERE id IN (...);
```

Properties:
- **At-least-once delivery** — events may be duplicated; consumers must be idempotent
- **Ordered within an aggregate** — events for the same `aggregate_id` dispatched in order
- **Durable** — events survive process crashes

## Configuration

Three-layer with explicit precedence:

1. Defaults — constants in `internal/config/defaults.go`
2. Environment variables — prefixed `APP_`
3. CLI flags — highest priority

Fail-fast at startup: unknown keys, missing required, or validation errors stop the process before port binding.

## Observability

- **Logs** — structured JSON via slog, one line per request + one per domain event
- **Metrics** — Prometheus endpoint on an admin port; module-tagged histograms + counters
- **Traces** — OpenTelemetry via OTLP, spans at HTTP/DB/external-call boundaries
- **Request ID propagation** — generated at edge, threaded through context, included in all logs

## Graceful shutdown

On SIGTERM:
1. HTTP server stops accepting new connections
2. Readiness probe starts returning 503
3. In-flight requests get a 30 s grace window
4. Outbox dispatcher finishes its current batch
5. DB pool closes
6. Telemetry exporters flush
7. Process exits 0

Hard deadline: 45 s total. If exceeded, process exits 1 and orchestrator restarts.

## Deployment

- Docker image from a small multi-stage build (`~25 MB`)
- Binary from the Dockerfile runs as non-root
- Secrets injected as env vars at runtime (no baked secrets)
- Healthchecks at `/live` (process up) and `/ready` (dependencies healthy)

## Anti-patterns we avoid

- ❌ Shared helper packages (`/utils`, `/common`) — breeds dependency spaghetti
- ❌ Anaemic domain models — business logic belongs on entities, not on service layers
- ❌ ORMs that abstract the query — we use sqlc for explicit, typed SQL
- ❌ Global state outside `main.go`
- ❌ Magic constants — every behavior is config, every config has a default
- ❌ Tests that depend on time.Now() or real network — everything is injected

## References

- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) — Uncle Bob
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) — Alistair Cockburn
- [Modular Monoliths](https://kamil.grzybek.com/design/modular-monolith-primer/) — Kamil Grzybek
- Sibling repo: [`ddd-workout-kit`](https://github.com/tunacosgun/ddd-workout-kit) — full DDD reference we study alongside this template
