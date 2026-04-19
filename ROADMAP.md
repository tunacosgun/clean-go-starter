# Roadmap

Planned work on `clean-go-starter`, maintained by Tunasoft Yazılım.

## Near term

### 🚧 Import linter GitHub Action
The `tools/importlint.go` script enforces module boundaries. Packaging it as a reusable GitHub Action so downstream teams can adopt module isolation without copying the file.

### 🚧 Migration squash helper
As migrations accumulate, `goose up` on a fresh DB gets slow. Add a `make migrate-squash` that consolidates N old migrations into a single snapshot while keeping a rollback path.

### 🚧 Pre-commit hook for sqlc drift
When you change a `query.sql` without regenerating, CI catches it — but only after push. Add a local pre-commit hook that blocks the commit.

## Next

### ⏳ Module scaffolding CLI
A `clean-go scaffold module <name>` command that creates the full `domain/usecase/port/adapter` skeleton for a new module, with correct import linter rules registered. Removes 15 minutes of boilerplate.

### ⏳ Event bus abstraction
Today the outbox dispatcher writes to a single in-process channel. Abstract to a `Bus` interface so we can plug in NATS or Redis streams for cross-process bus when a product outgrows a single node.

### ⏳ Typed config builder
Config today is a struct-with-tags + viper combo. Move to a typed builder pattern so missing-required-key errors point to the caller line, not the config file.

### ⏳ Example payment webhook module
The template ships with orders, payments, activation, access modules — but the payments module is minimal. Expand it with a full Stripe + iyzico webhook receiver demonstrating idempotent-by-event-id patterns. This would make the template a better starting point for subscription products.

## Later

### 📦 gRPC transport option
Add a parallel gRPC adapter layer for internal callers. Keeps HTTP as the public edge but lets in-VPC services use gRPC.

### 📦 End-to-end test harness
Compose-based harness with a seeded DB and a fake payment provider. `make test-e2e` runs the full stack and asserts.

### 📦 Rate-limit middleware with Redis backend
Today the template's rate limit is in-memory. Add Redis-backed sliding window for horizontal scaling.

### 📦 Multi-region database plumbing
Template for read-replica + write-primary setups with explicit query routing.

## Accepted, not scheduled

- Terraform modules for AWS / GCP deployment
- Helm chart for Kubernetes deployment (we ship with Docker Compose + systemd by default)
- Admin UI scaffold (React + TanStack Query)

## Non-goals

- ❌ Competing with sqlc / goose / chi — we integrate them, we don't replace them
- ❌ Generic microservices framework — the whole point is "modular monolith as default"
- ❌ Being a general-purpose "everything" template — we keep it focused on commercial subscription / internal-tool backends

## Release cadence

Tagged releases when a meaningful integration feature lands.

## Contributing to the roadmap

Contact [info@tunahancosgun.dev](mailto:info@tunahancosgun.dev) if you want a roadmap item prioritized for a commercial engagement.
