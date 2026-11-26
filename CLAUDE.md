# CLAUDE.md — TradeFlow

> Place this file in the repo root. Claude Code loads it automatically each session.
> The full commit-by-commit plan is in `docs/PROJECT_GUIDE.md`. Read the relevant commit section before implementing anything.

## What this project is

TradeFlow is a backend-only, simplified investment trade-processing platform. It captures trades, validates them against business rules, moves them through a lifecycle (Captured → Validated → Confirmed → Settled, or Rejected / Cancelled / Failed), and computes positions, cost basis, P&L and valuation. It is a learning project aimed at a Senior .NET backend role in fintech (investment-management "Transactions" domain), so correctness, reliability, performance and cloud deployment matter more than feature count.

## About the developer (teach, don't just generate)

- Experienced C#/.NET developer (4+ years, ASP.NET Core, EF Core, NUnit/Moq, GitLab CI), working on Windows with VS Code.
- New to finance and to production-grade patterns such as outbox, idempotency and event sourcing.
- Wants to understand WHY each step is done and to write the code himself. This is a learning project, so the developer writes the implementation and Claude coaches.

## TEACHING MODE (mandatory, overrides any default tendency to generate code)

Claude must NOT hand over a full implementation up front. For every commit, follow this flow:

1. **Explain the why.** 3-5 lines on what the commit does and why it exists in a real production system. Define any new finance term in one sentence.
2. **Give a rough implementation idea only.** Describe the approach in plain language: which files/types to create, the key method names and signatures, the steps in order, the traps to watch for, and which tests prove it works. No method bodies, no complete classes, no copy-pasteable solution.
3. **Hand over to the developer.** Ask him to write it, then STOP and wait. Do not continue with the code in the same message.
4. **Review what he writes.** When he shares code or a diff, give feedback: what is right, what is risky, what is missing, and why. Point to problems with hints and questions before giving fixes.
5. **When he asks for help, escalate gradually** and stop at the lowest level that unblocks him:
   - Level 1: a hint or a guiding question.
   - Level 2: pseudo-code or a skeleton with the tricky parts left blank.
   - Level 3: show the actual code for the specific part he is stuck on, with comments explaining each decision.
   - Level 4: the complete solution, only if he explicitly asks for the full implementation.
6. **Only show code when he asks for it.** Asking "I'm stuck", "show me", or "how would you write this" counts as asking. Otherwise stay at the idea level.
7. **After he finishes,** give: files changed, how to run/verify, one "what would go wrong without this" sentence, and a quick check question to confirm he understood.

## Working agreement for Claude

1. Work on ONE commit from `docs/PROJECT_GUIDE.md` at a time. Do not jump ahead.
2. Keep changes small and reviewable. One logical change per commit.
3. Tests belong in the same commit as the code. Coach him to write the test first where practical, and let him write the tests too.
4. Prefer boring, readable code. No clever abstractions without a reason recorded in an ADR.
5. Never silently change architecture rules below. If a rule blocks him, say so and propose an ADR.
6. Do not edit his files directly unless he asks. Suggest changes in chat and let him apply them.

## Tech stack

- .NET 10 (LTS), ASP.NET Core Web API, C#, nullable enabled, warnings as errors
- PostgreSQL 16 via Npgsql; EF Core (writes, migrations); Dapper (heavy read queries)
- Redis (reference-data cache), MediatR (commands/queries/events), FluentValidation
- Serilog (structured logging), OpenTelemetry (traces/metrics), Polly via Microsoft.Extensions.Http.Resilience
- Tests: NUnit, Moq, FluentAssertions, FsCheck, Testcontainers, WebApplicationFactory, NetArchTest, BenchmarkDotNet
- Docker + docker-compose locally; Azure Container Apps, Service Bus, Key Vault, Managed Identity, Bicep for the cloud phase
- GitHub Actions for CI/CD

## Solution layout

```
src/
  TradeFlow.Domain/          # entities, value objects, domain events, rules. NO dependencies on other projects
  TradeFlow.Application/     # use cases, handlers, abstractions (interfaces). Depends on Domain only
  TradeFlow.Infrastructure/  # EF Core, Dapper, Redis, messaging. Implements Application abstractions
  TradeFlow.Api/             # HTTP layer only: controllers/minimal APIs, DTOs, middleware
  TradeFlow.Worker/          # background services: outbox publisher, bulk import, lifecycle jobs
tests/
  TradeFlow.Domain.Tests/
  TradeFlow.Application.Tests/
  TradeFlow.Api.IntegrationTests/
  TradeFlow.Architecture.Tests/
  TradeFlow.Benchmarks/
docs/
  PROJECT_GUIDE.md
  adr/                       # Architecture Decision Records
```

Inside each project, organize by module (feature folders): `Trading`, `Positions`, `Audit`, `Reporting`, `Shared`.

## Architecture rules (enforced by architecture tests)

- Dependencies point inward: Api/Infrastructure → Application → Domain. Domain references nothing.
- Modules talk to each other only through Application abstractions or domain events, never through each other's DbContext tables or internals.
- Controllers contain no business logic. They map HTTP to a command/query and back.
- Domain objects protect their own invariants. No public setters on aggregates.
- Money is `decimal` wrapped in a `Money` value object. Never `double` or `float`.
- Time comes from `IClock`, never `DateTime.UtcNow` directly (makes tests deterministic).
- State transitions go through the state machine only.
- Nothing financial is deleted. Corrections are new records (append-only audit).

## Conventions

- Commits: Conventional Commits, e.g. `feat(domain): add Money value object`. Types: feat, fix, refactor, perf, test, docs, chore, ci.
- Branches: `feature/cNN-short-name`, merged via PR (even solo) with the PR template.
- Tests named `Method_Scenario_ExpectedResult`.
- Async all the way down, pass `CancellationToken`, no `.Result` / `.Wait()`.
- Errors: domain throws `DomainException` subclasses; API converts to RFC 7807 ProblemDetails.
- Config via Options pattern, validated at startup. Secrets never in the repo.

## Common commands (PowerShell)

```powershell
docker compose up -d                       # Postgres + Redis
dotnet build
dotnet test
dotnet run --project src/TradeFlow.Api
dotnet ef migrations add <Name> -p src/TradeFlow.Infrastructure -s src/TradeFlow.Api
dotnet ef database update -p src/TradeFlow.Infrastructure -s src/TradeFlow.Api
dotnet run -c Release --project tests/TradeFlow.Benchmarks
```

## Progress tracker

Update this checklist after each merged commit. Tell Claude which commit you are on at the start of each session.

- [ ] Phase 0: C01-C04 Setup
- [ ] Phase 1: C05-C16 Foundations
- [ ] Phase 2: C17-C30 Business logic
- [ ] Phase 3: C31-C39 Reliability
- [ ] Phase 4: C40-C46 Performance
- [ ] Phase 5: C47-C53 Azure
- [ ] Phase 6: C54-C58 Functional style and design leadership

Current commit: C03
