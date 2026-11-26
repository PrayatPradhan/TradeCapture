# TradeFlow

TradeFlow is a backend-only, simplified investment trade-processing platform. It captures trades, validates them against business rules, moves them through a strict lifecycle, and computes positions, cost basis, P&L and valuation. It is a learning project modelled on the "Transactions" part of an investment back office, so correctness, reliability, performance and cloud deployment matter more than feature count.

## What it does

1. **Capture** trades (REST and bulk CSV import)
2. **Validate** them with pluggable business rules
3. **Process** them through a lifecycle: Captured → Validated → Confirmed → Settled (or Rejected / Cancelled / Failed)
4. **Compute** positions, FIFO cost basis, realized and unrealized P&L, and valuation in a reporting currency
5. **Audit** every change in an append-only log
6. **Report** positions as of a date and trade history

Out of scope: UI, real market-data feeds, real SWIFT/FIX messaging, order matching, tax, user management.

## Tech stack

- .NET 10 (LTS), ASP.NET Core Web API, C#, nullable enabled, warnings as errors
- PostgreSQL via Npgsql, EF Core for writes, Dapper for heavy reads
- Redis (reference-data cache), MediatR, FluentValidation
- Serilog, OpenTelemetry, Polly
- NUnit, Moq, FluentAssertions, FsCheck, Testcontainers, NetArchTest, BenchmarkDotNet
- Docker and docker-compose locally; Azure Container Apps and Bicep in the cloud phase

Several of these arrive in later commits. See the plan for when.

## Architecture

TradeFlow is a **modular monolith** with Clean/Onion layering. One deployable unit, with strict internal modules (`Trading`, `Positions`, `Audit`, `Reporting`). The reasoning is in [ADR-0001](docs/adr/0001-modular-monolith.md).

```
Api ──────────► Application ──► Domain
Infrastructure ─► Application ──► Domain
```

- **Domain**: entities, value objects, domain events and rules. References nothing.
- **Application**: use cases and abstractions (interfaces). Depends on Domain only.
- **Infrastructure**: EF Core, Dapper, Redis, messaging. Implements Application abstractions.
- **Api**: HTTP layer only.
- **Worker**: background services (outbox publisher, bulk import, lifecycle jobs).

The compiler enforces the layering through project references, and architecture tests will enforce the rest (from C15).

```
src/
  TradeFlow.Domain/
  TradeFlow.Application/
  TradeFlow.Infrastructure/
  TradeFlow.Api/
  TradeFlow.Worker/
tests/
  TradeFlow.Domain.Tests/
  TradeFlow.Application.Tests/
  TradeFlow.Api.IntegrationTests/
  TradeFlow.Architecture.Tests/
docs/
  PROJECT_GUIDE.md
  adr/
```

## Getting started

Prerequisites:

- The .NET SDK pinned in [`global.json`](global.json) (10.0.x)
- Docker Desktop (needed from C16 for Postgres and Redis)

Build and test:

```powershell
dotnet build
dotnet test
```

Running the API and the docker-compose environment arrive in C16, and this section will be extended then.

## Documentation

- [Project guide](docs/PROJECT_GUIDE.md): scope, finance primer, architecture, and the commit-by-commit plan
- [Architecture Decision Records](docs/adr/): why key decisions were made
- [CLAUDE.md](CLAUDE.md): working agreement and conventions for the AI coding assistant used on this project

## Conventions

- Conventional Commits, for example `feat(domain): add Money value object`
- Branches `feature/cNN-short-name`, merged via pull request
- Tests named `Method_Scenario_ExpectedResult`
- Money is `decimal` inside a `Money` value object, never `double`
- Time comes from `IClock`, never `DateTime.UtcNow`
