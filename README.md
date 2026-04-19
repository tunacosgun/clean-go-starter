# clean-go-starter

> Production-grade Clean Architecture starter template for Go services. The base we reach for when bootstrapping a modular monolith — payments, auth, subscriptions, access orchestration — under a single deployable binary.

<p align="left">
  <img alt="Go" src="https://img.shields.io/badge/Go-1.22%2B-00ADD8?style=flat-square&logo=go&logoColor=white">
  <img alt="Architecture" src="https://img.shields.io/badge/Architecture-Modular%20Monolith-blue?style=flat-square">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green?style=flat-square">
  <img alt="Maintained" src="https://img.shields.io/badge/Maintained%20by-Tunasoft%20Yazılım-181717?style=flat-square">
</p>

---

## What this is

`clean-go-starter` is the Tunasoft Yazılım flavor of a Clean Architecture Go template. It is the iskelet (skeleton) we drop in on day one of a new commercial backend. Ships with:

- **Modular monolith layout** — domain modules with enforced boundaries (`internal/` + import linter rules)
- **Hexagonal ports & adapters** — business logic isolated from HTTP, DB, and external services
- **HTTP stack** — chi router, middleware chain, structured errors
- **Postgres persistence** — sqlc for typed queries, goose for migrations
- **Observability** — OpenTelemetry traces, Prometheus metrics, slog structured logging
- **Config management** — 3-layer (default → env → flag) with validation
- **Outbox pattern** — transactional event emission, separate dispatcher goroutine
- **Graceful shutdown** — signal handling, in-flight request drain
- **Docker Compose** for local dev (Postgres + Redis + Jaeger)
- **Makefile + pre-commit hooks** — lint, vet, test, migration verify

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the layering model and module boundaries, and [`ROADMAP.md`](./ROADMAP.md) for planned work.

## Why we maintain this

In practice, 80% of the commercial backends we deliver have the same non-business bootstrap concerns: HTTP setup, migrations, observability, config, graceful shutdown, transactional outbox. Having a template that is **opinionated, modular, and production-tuned** means we spend day one on the domain, not on plumbing.

We deliberately chose **modular monolith over microservices** for this scale (subscription apps, access brokers, internal tools) because the added operational complexity of microservices rarely pays for itself until traffic and team size justify it. Modules are strictly isolated inside the same process, and if a piece needs to break out later, the module boundaries give us a clean extraction path.

## Layout

```
.
├── cmd/
│   └── app/
│       └── main.go              # wiring, config, graceful shutdown
├── internal/
│   ├── app/                     # orchestration, routing
│   ├── config/                  # 3-layer config with validation
│   ├── modules/
│   │   ├── orders/              # domain module: order lifecycle
│   │   │   ├── domain/          # entities, value objects, rules
│   │   │   ├── usecase/         # application logic (orchestration)
│   │   │   ├── adapter/         # HTTP handlers, DB repository
│   │   │   └── port/            # inbound/outbound interfaces
│   │   ├── payments/            # domain module: payment + webhook
│   │   ├── activation/          # domain module: token + device binding
│   │   └── access/              # domain module: access node orchestration
│   ├── shared/
│   │   ├── database/            # Postgres pool, migrations runner
│   │   ├── httpserver/          # chi wrapper, middleware
│   │   ├── outbox/              # transactional outbox + dispatcher
│   │   ├── telemetry/           # OTel setup
│   │   └── logger/              # slog wrapper
│   └── platform/
│       └── errors/              # domain-level error taxonomy
├── migrations/                  # goose migrations
├── docs/
│   ├── adr/                     # Architecture Decision Records
│   └── runbook.md
├── deploy/
│   ├── docker-compose.yml
│   └── Dockerfile
├── Makefile
└── go.mod
```

## Getting started

### Requirements

- Go 1.22+
- Docker + Docker Compose
- `make`

### Bootstrap

```bash
git clone https://github.com/tunacosgun/clean-go-starter.git
cd clean-go-starter
make compose-up      # Postgres + Redis + Jaeger
make migrate-up
make run
```

Hit it:

```bash
curl http://localhost:8080/health
```

### Make targets

```
make run              # run the service
make test             # unit + integration tests
make lint             # golangci-lint + goimports check
make migrate-up       # apply pending migrations
make migrate-down     # revert last migration
make mock             # regenerate mocks from port interfaces
make sqlc             # regenerate typed DB queries
```

## How we use it

Every new project we deliver starts by:

1. `git clone` this template
2. Rename the service in `go.mod`, `cmd/app/main.go`, and `docker-compose.yml`
3. Keep only the modules relevant to the product; delete the rest
4. Write the first domain module (business logic only — no infra)
5. Wire the module's ports to the HTTP layer and DB

Day two, the domain is usable end-to-end and we can start shipping features.

## Related Tunasoft projects

- [`ddd-workout-kit`](https://github.com/tunacosgun/ddd-workout-kit) — reference DDD project we study alongside this template when modeling complex domains
- [`go-cookbook`](https://github.com/tunacosgun/go-cookbook) — snippet-sized examples for webhook idempotency, JWT, OAuth, payments
- [`identity-hub`](https://github.com/tunacosgun/identity-hub) — auth server built on the same layering patterns

## Credits

Built on top of the excellent [`go-clean-template`](https://github.com/evrone/go-clean-template) by Evrone (MIT). This fork is maintained by Tunahan Coşgun and Duygu Durmuş at Tunasoft Yazılım and extended with modular-monolith layout, transactional outbox, and production-tuned defaults we use in commercial delivery.

## License

MIT — see [`LICENSE`](./LICENSE).

## Contact

- 📧 [info@tunahancosgun.dev](mailto:info@tunahancosgun.dev)
- 🌐 [tunahancosgun.dev](https://tunahancosgun.dev)
- 📅 [Book a consultation](https://cal.com/tunacosgun/intro)
