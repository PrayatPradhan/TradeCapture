# TradeFlow — Project Guide

Save as `docs/PROJECT_GUIDE.md`. Companion file: `CLAUDE.md` in the repo root.

How to use this guide: pick the next commit, tell Claude "We are on C07, read its section in docs/PROJECT_GUIDE.md, explain why first, then implement." Each commit has **Do** (what to build), **Why** (the production reason) and **Done when** (verification).

---

## 1. Goal and scope

Build a backend that behaves like the core of an investment back office:

1. **Capture** trades (REST + bulk import)
2. **Validate** them with pluggable business rules
3. **Process** them through a strict lifecycle
4. **Compute** positions, cost basis, realized/unrealized P&L, valuation in a reporting currency
5. **Audit** every change, immutably
6. **Report** positions as of a date and trade history

**In scope:** equities and FX conversion first. Bonds (accrued interest) are a stretch goal.
**Out of scope:** UI, real market-data feeds, real SWIFT/FIX messaging, order matching, tax, user management (use a simple API key or none).

Why this project: the target role covers backend logic in a long-lived enterprise financial system, with emphasis on complex business rules, reliability, performance, testing and CI/CD, plus Azure as an advantage. Every phase below maps to one of those.

---

## 2. Finance primer

| Term | Meaning | Where it appears in the code |
|---|---|---|
| Portfolio | A collection of holdings managed together | `Portfolio` entity |
| Instrument | Something tradable: equity, bond, FX pair | `Instrument` entity |
| ISIN | 12-char global instrument ID (2-letter country, 9 alphanumerics, 1 check digit) | `Isin` value object |
| Trade | One buy or sell: instrument, side, quantity, price, currency, trade date, settlement date | `Trade` aggregate |
| Trade date (T) | Day the deal is agreed | `Trade.TradeDate` |
| Settlement date | Day cash and securities actually swap, usually T+1 or T+2 business days | `SettlementDateCalculator` |
| Counterparty | The other side of the trade | `Trade.Counterparty` |
| Position | Quantity of an instrument held in a portfolio, derived from trades | `Position` aggregate |
| Lot | A purchased chunk with its own quantity and price, used for cost tracking | `Lot` |
| Cost basis | What you paid for what you still hold | `Position.CostBasis` |
| FIFO | First-in-first-out: sells consume the oldest lots first | `Position.ApplySell` |
| Realized P&L | Profit/loss locked in by selling | returned by `ApplySell` |
| Unrealized P&L | Profit/loss on what you still hold, at current market price | `ValuationService` |
| FX rate | Price of one currency in another, used to convert to a reporting currency | `IFxRateProvider` |
| Valuation | Position quantity × market price, converted to reporting currency | `ValuationService` |
| Confirmation | Both sides agree the trade details match | `Confirm` command |
| Settlement break / fail | Settlement did not happen as expected (seller didn't deliver) | `Failed` status |
| Reconciliation | Comparing your records to the custodian's to find breaks | stretch goal |
| Audit trail | Immutable history of who changed what and when | `audit_events` table |

### Worked example (use this as a test fixture)

1. Buy 100 shares of ACME @ 10.00, then buy 50 @ 12.00. Lots: `[100 @ 10]`, `[50 @ 12]`. Quantity 150, cost basis 1,600, average cost 10.6667.
2. Sell 80 @ 13.00 using FIFO. All 80 come from lot 1. Realized P&L = 80 × (13 − 10) = **240**. Remaining lots: `[20 @ 10]`, `[50 @ 12]`. Quantity 70, cost basis 200 + 600 = 800.
3. Market price is 14.00. Market value = 70 × 14 = 980. Unrealized P&L = 980 − 800 = **180**.
4. Settlement date: a trade made on a Friday with T+2 settles on Tuesday (Saturday and Sunday are skipped), assuming no holiday.

Put these numbers in unit tests early. They are your ground truth.

---

## 3. Architecture

### 3.1 Modular monolith first

One deployable unit, strict internal modules (`Trading`, `Positions`, `Audit`, `Reporting`). Reason: boundaries are guessed wrong early, and microservices make wrong guesses expensive (network failure, distributed transactions, deployment overhead). In Phase 5 you extract one piece so you feel the tradeoff firsthand.

### 3.2 Clean/Onion layering

```
Api ──► Application ──► Domain
Infrastructure ──► Application ──► Domain
```

Domain holds the rules and knows nothing about databases or HTTP. That makes it unit-testable in milliseconds and lets infrastructure change without touching business rules.

### 3.3 Request flow (write path)

```
POST /api/v1/trades  (Idempotency-Key header)
  → middleware: idempotency check
  → handler: build Trade aggregate (invariants enforced)
  → rules pipeline: validate
  → repository + outbox + audit rows saved in ONE DB transaction
  → worker publishes outbox rows to the message bus
  → subscribers (positions, reporting) react
```

### 3.4 Read path

Reporting queries use Dapper against optimized SQL (and later Redis for reference data). Reads never lock or slow writes.

### 3.5 Key patterns and why

| Pattern | Problem it solves |
|---|---|
| Value objects | Prevents mixing currencies, negative quantities, invalid ISINs at compile/construct time |
| State machine | Makes illegal lifecycle transitions impossible, with one place to read the rules |
| Specification / rule pipeline | New rules plug in without modifying old ones (open/closed) |
| Domain events | Decouples "something happened" from "who cares" |
| Idempotency key | Client retry never books twice |
| Optimistic concurrency | Two writers can't silently overwrite each other |
| Transactional outbox | Saving state and publishing an event can't diverge after a crash |
| Inbox / idempotent consumer | At-least-once delivery doesn't double-apply an event |
| Append-only audit | Regulatory traceability and rebuildable history |
| Cache-aside | Fast reference-data reads with explicit invalidation |

### 3.5.1 Trade status transitions

```
Captured ──► Validated ──► Confirmed ──► Settled
   │            │             │
   ▼            ▼             ├──► Failed
Rejected    Cancelled         └──► Cancelled
```

Allowed: Captured→Validated, Captured→Rejected, Validated→Confirmed, Validated→Cancelled, Confirmed→Settled, Confirmed→Failed, Confirmed→Cancelled. Everything else throws `InvalidTradeTransitionException`. Settled, Rejected, Cancelled and Failed are terminal.

### 3.6 Data model sketch (PostgreSQL)

- `instruments(id, isin, symbol, type, currency, is_tradable)`
- `portfolios(id, name, base_currency)`
- `trades(id, portfolio_id, instrument_id, side, quantity, price, currency, trade_date, settlement_date, status, counterparty, version, created_at, updated_at)`
- `positions(id, portfolio_id, instrument_id, quantity, cost_basis, realized_pnl, version)` and `position_lots(id, position_id, quantity_remaining, unit_price, acquired_on)`
- `fx_rates(base, quote, rate, as_of)`
- `idempotency_keys(key, request_hash, response_status, response_body, created_at)`
- `audit_events(id, aggregate_type, aggregate_id, event_type, payload_json, actor, occurred_at)` — insert only, no updates or deletes
- `outbox_messages(id, type, payload_json, occurred_at, processed_at, attempts)`
- `inbox_messages(message_id, consumer, processed_at)`

Money: `numeric(18,4)` for amounts, `numeric(18,8)` for FX rates and unit prices if needed.

### 3.7 API surface (v1)

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/v1/trades` | Capture a trade |
| GET | `/api/v1/trades/{id}` | Trade details |
| GET | `/api/v1/trades` | Search by portfolio, status, date range, paging |
| POST | `/api/v1/trades/{id}/confirm` | Confirm |
| POST | `/api/v1/trades/{id}/settle` | Settle |
| POST | `/api/v1/trades/{id}/cancel` | Cancel |
| POST | `/api/v1/trades/import` | Bulk file import (CSV) |
| GET | `/api/v1/portfolios/{id}/positions?asOf=` | Positions as of a date |
| GET | `/api/v1/portfolios/{id}/valuation?asOf=&currency=` | Valuation |
| GET | `/api/v1/trades/{id}/audit` | Audit history |
| GET | `/health/live`, `/health/ready` | Health checks |

---

## 4. Working method for each commit

This project runs in **teaching mode** (see `CLAUDE.md`): Claude explains and sketches the approach, you write the code, and Claude only shows code when you ask for it.

1. Create a branch `feature/cNN-short-name`.
2. Ask Claude to explain why the commit exists and give a rough implementation idea.
3. Write the failing test first where practical (domain code especially).
4. Implement it yourself. If you get stuck, ask for help and Claude escalates: hint, then skeleton, then code for the stuck part, then the full solution only if you ask.
5. Run `dotnet build` and `dotnet test`, and fix warnings.
6. Share your code for review, fix what Claude flags, then commit with a Conventional Commit message, open a PR to yourself, and review the diff as if it were a teammate's.
7. Tick the box in `CLAUDE.md`.

Useful prompts for Claude in VS Code:
- "We are on C12. Explain why it exists and give me a rough implementation idea. Don't write the code."
- "Here is my implementation. Review it against the architecture rules in CLAUDE.md."
- "I'm stuck on the FIFO lot consumption. Give me a hint first."
- "Show me the code for just the rounding part, with comments."
- "Give me the full solution for this commit." (use this last)
- "What failure would this pattern prevent? Help me write a test that reproduces the failure without it."
- "Quiz me on this commit in three questions, like an interviewer would."

---

## 5. Commit-by-commit plan

### Phase 0 — Setup (C01–C04)

**C01 — `chore: initialize repo with build and style settings`**
- **Do:** `git init`, `.gitignore` (dotnet template), `.editorconfig`, `global.json` pinning the .NET 10 SDK, `Directory.Build.props` (`Nullable`, `ImplicitUsings`, `TreatWarningsAsErrors`, `AnalysisLevel`), `Directory.Packages.props` for central package management.
- **Why:** Consistent builds and one place to pin package versions prevent "works on my machine" and version drift across projects. Warnings-as-errors keeps quality from decaying silently.
- **Done when:** An empty `dotnet build` succeeds and `.editorconfig` is respected by VS Code.

**C02 — `chore: add solution and project skeleton`**
- **Do:** Create the solution, the five `src` projects and four test projects. Wire project references exactly as in `CLAUDE.md`. Add a placeholder test per test project.
- **Why:** Layering is enforced by the compiler through project references. If Domain can't reference Infrastructure, nobody can accidentally couple them.
- **Done when:** `dotnet build` and `dotnet test` pass. Domain has zero project references.

**C03 — `docs: add README, CLAUDE.md and ADR-0001 modular monolith`**
- **Do:** README with goal and run instructions, drop in `CLAUDE.md` and this guide, and write `docs/adr/0001-modular-monolith.md` (context, decision, consequences).
- **Why:** An ADR records why a decision was made, so future readers (and you in six months) don't reverse it blindly. Writing ADRs is also visible evidence of design leadership.
- **Done when:** A new reader can understand the project in five minutes.

**C04 — `ci: add GitHub Actions build and test workflow`**
- **Do:** Workflow on push and PR: restore, build, test, upload test results. Cache NuGet.
- **Why:** CI is the safety net that makes every later refactor cheap. It starts on day one, not at the end.
- **Done when:** A PR shows a green check.

### Phase 1 — Foundations (C05–C16)

**C05 — `feat(domain): add Money value object`**
- **Do:** Immutable `Money(decimal Amount, CurrencyCode Currency)`. Operators `+`, `-`, comparison, multiply by decimal. Throw on mixing currencies. Define a rounding policy (e.g. `MidpointRounding.ToEven`, 4 dp internally, 2 dp for display) in one place. Use `readonly record struct` or a sealed record.
- **Why:** `double` can't represent 0.1 exactly, and errors accumulate across millions of trades. Making currency part of the type turns "added USD to EUR" from a production incident into a compile or construction error.
- **Done when:** Tests cover add/subtract, currency mismatch throws, rounding is deterministic, and equality works.

**C06 — `feat(domain): add Quantity, Price, Isin and CurrencyCode`**
- **Do:** `Quantity` (positive decimal), `Price` (non-negative decimal plus currency), `CurrencyCode` (ISO 4217, three uppercase letters), `Isin` (format plus Luhn check digit validation). Factory methods return validated instances.
- **Why:** "Parse, don't validate": invalid values never enter the domain, so every later method can trust its inputs.
- **Done when:** Known-valid ISINs (e.g. `US0378331005`) pass, altered check digits fail, and zero/negative quantity is rejected.

**C07 — `feat(domain): add Instrument entity`**
- **Do:** `Instrument` with id, `Isin`, symbol, `InstrumentType` (Equity, FxPair; Bond later), trading currency, `IsTradable`. A small seed list of ~20 real-looking equities.
- **Why:** Rules in C20 need instrument data (is it tradable, which currency). Keeping it a proper entity avoids stringly-typed logic.
- **Done when:** Construction invariants tested.

**C08 — `feat(domain): add Trade aggregate`**
- **Do:** `Trade` aggregate root with private setters, `Side` (Buy/Sell), `Create(...)` factory enforcing invariants (settlement date not before trade date, currency matches instrument unless FX). Initial status `Captured`.
- **Why:** An aggregate is the consistency boundary: all rules that must hold together live together. No public setters means no code path can create an invalid trade.
- **Done when:** Tests prove invalid trades can't be constructed.

**C09 — `feat(domain): add Portfolio entity`**
- **Do:** `Portfolio` with id, name, base currency.
- **Why:** Positions and valuation are always scoped to a portfolio, and base currency drives conversion.
- **Done when:** Unit tests pass.

**C10 — `feat(app): add CreateTrade use case and abstractions`**
- **Do:** Interfaces `ITradeRepository`, `IInstrumentRepository`, `IUnitOfWork`, `IClock`. `CreateTradeCommand` and handler (plain class for now, MediatR arrives in C24). Test with Moq.
- **Why:** The application layer orchestrates but depends only on interfaces, so it's testable without a database (dependency inversion).
- **Done when:** Handler tests verify the repository receives the trade and the unit of work commits once.

**C11 — `feat(infra): add EF Core Postgres persistence and first migration`**
- **Do:** `TradeFlowDbContext`, `IEntityTypeConfiguration<T>` per entity, value converters for value objects (or owned types), initial migration, repository implementations, unit of work.
- **Why:** Keeping mapping in configuration classes (not attributes on domain types) keeps Domain free of persistence concerns. Migrations make schema changes versioned and repeatable.
- **Done when:** `dotnet ef database update` creates the schema in local Postgres and a round-trip save/load works.

**C12 — `feat(api): add POST and GET trade endpoints`**
- **Do:** DTOs (request/response), FluentValidation validators, controller or minimal API endpoints for `POST /trades` and `GET /trades/{id}`. Return `201` with `Location` header.
- **Why:** The API layer translates HTTP to commands. DTOs prevent leaking domain internals and allow the API contract to evolve independently.
- **Done when:** Swagger can create and fetch a trade.

**C13 — `feat(api): add ProblemDetails error handling, API versioning and Swagger`**
- **Do:** Global exception handler (`IExceptionHandler`) mapping `DomainException` → 422/409, validation → 400, unknown → 500 with a trace ID. `Asp.Versioning` with `/api/v1`. Swagger with XML docs.
- **Why:** Consistent error shape (RFC 7807) lets clients handle failures programmatically. Versioning lets you evolve the contract without breaking existing clients, which matters in long-lived systems.
- **Done when:** Errors from every layer return ProblemDetails with correlation ID.

**C14 — `feat(infra): add Dapper query for trade search`**
- **Do:** `ITradeReadService` with a Dapper implementation: filters (portfolio, status, date range) plus keyset or offset paging. Index on `(portfolio_id, status, trade_date)`.
- **Why:** Writes need change tracking and invariants (EF Core). Reads need raw speed and exact SQL control (Dapper). This is CQRS-lite, and you'll compare both approaches with numbers in Phase 4.
- **Done when:** Endpoint returns a paged list with correct filtering.

**C15 — `test: add architecture tests`**
- **Do:** NetArchTest rules: Domain depends on nothing, Application doesn't reference Infrastructure/Api, controllers don't reference repositories, no public setters on aggregates.
- **Why:** Rules written only in documents rot. Rules in tests fail the build when someone breaks them.
- **Done when:** Deliberately breaking a rule makes the test fail.

**C16 — `chore: add docker-compose, Serilog and Options pattern`**
- **Do:** `docker-compose.yml` with Postgres and Redis, Serilog console JSON logging with correlation IDs, strongly typed options validated on startup (`ValidateOnStart`).
- **Why:** One command gives any developer a working environment. Structured logs are searchable fields instead of text. Failing at startup on bad config beats failing at 3 a.m. in production.
- **Done when:** `docker compose up -d && dotnet run` works from a clean clone.

### Phase 2 — Business logic (C17–C30)

**C17 — `feat(domain): add trade state machine`**
- **Do:** `TradeStatus` enum, a transition table, `Trade.Validate()`, `Confirm()`, `Settle()`, `Cancel()`, `Reject()`, `Fail()`. Invalid transitions throw `InvalidTradeTransitionException`.
- **Why:** Centralizing legal transitions prevents bugs like settling a cancelled trade. A table is easy to review with business people.
- **Done when:** A test iterates every (from, to) pair and asserts exactly the allowed set succeeds.

**C18 — `feat(domain): add domain events`**
- **Do:** `IDomainEvent`, events (`TradeCaptured`, `TradeValidated`, `TradeConfirmed`, `TradeSettled`, `TradeCancelled`, `TradeRejected`, `TradeFailed`). Aggregates collect events; infrastructure dispatches after save.
- **Why:** Events decouple "what happened" from "who reacts" (positions, audit, reporting), so adding a reaction never touches trade code.
- **Done when:** Each transition raises the right event, tested.

**C19 — `feat(app): add rule abstraction with specification pattern`**
- **Do:** `ITradeRule { RuleResult Evaluate(Trade, ValidationContext) }`, composable specifications (And/Or/Not), `RuleResult` with code and message.
- **Why:** Business rules change constantly. Plug-in rules mean new rules are new classes, not edits to a growing `if` ladder.
- **Done when:** Rules compose and are testable in isolation.

**C20 — `feat(app): add four initial trading rules`**
- **Do:** `InstrumentIsTradableRule`, `QuantityWithinLimitRule`, `SettlementDateIsBusinessDayRule`, `DuplicateTradeRule` (same portfolio, instrument, side, qty, price, date within a window).
- **Why:** Typical front-line controls: wrong instrument, fat-finger quantity, impossible dates, double submission.
- **Done when:** Each rule has pass and fail tests, including edge cases at the limit.

**C21 — `feat(domain): add business calendar and settlement date calculator`**
- **Do:** `IBusinessCalendar` (weekends plus a holiday list per market), `SettlementDateCalculator` for T+N.
- **Why:** Date logic is a classic bug source. Isolating it makes it easy to test exhaustively (weekends, holidays, month and year boundaries).
- **Done when:** Friday + T+2 = Tuesday, and a holiday Monday pushes it to Wednesday.

**C22 — `feat(app): add ValidateTrade use case and rules pipeline`**
- **Do:** Run all registered rules, collect all failures (not just the first), move the trade to `Validated` or `Rejected`, and persist the rejection reasons.
- **Why:** Users want every problem at once. Persisted reasons make rejections explainable in audits.
- **Done when:** A trade violating two rules is rejected with both reasons.

**C23 — `feat(app): add Confirm, Settle and Cancel commands plus endpoints`**
- **Do:** Commands and handlers, HTTP endpoints, and a simulated settlement that can randomly fail for demo purposes (behind a flag).
- **Why:** Completes the lifecycle end to end and gives you a real failure path to test.
- **Done when:** A trade can be driven through each path via Swagger.

**C24 — `refactor(app): introduce MediatR with logging and validation behaviors`**
- **Do:** Move handlers to MediatR requests, add pipeline behaviors for logging, validation, and a transaction/unit-of-work wrapper.
- **Why:** Cross-cutting concerns live in one place instead of being repeated in every handler.
- **Done when:** Existing tests still pass and behavior order is tested.

**C25 — `feat(domain): add Position aggregate with lots`**
- **Do:** `Position` per (portfolio, instrument) holding `Lot`s. `ApplyBuy(quantity, price, date)` adds a lot.
- **Why:** Cost basis requires remembering what was bought and at what price, not just a total quantity.
- **Done when:** The worked example's two buys produce quantity 150 and cost basis 1,600.

**C26 — `feat(domain): add FIFO sells and realized P&L`**
- **Do:** `ApplySell(quantity, price)` consumes lots oldest first, returns realized P&L, rejects overselling (no short selling in v1).
- **Why:** The cost-basis method changes reported profit, and accounting rules require it be explicit and consistent.
- **Done when:** The worked example yields realized P&L of 240 and remaining lots `[20 @ 10]`, `[50 @ 12]`.

**C27 — `feat(domain): add FX rates and currency conversion`**
- **Do:** `IFxRateProvider` (fixed sample rates from a table), `CurrencyConverter` handling direct, inverse and cross rates via a pivot currency.
- **Why:** Portfolios hold assets in many currencies but are reported in one. Cross-rate logic is easy to get subtly wrong.
- **Done when:** USD→EUR→USD roundtrip is within the rounding policy, and a missing rate throws a clear error.

**C28 — `feat(app): add valuation service`**
- **Do:** `ValuationService` computes market value and unrealized P&L per position and for a portfolio in a reporting currency, using a price source interface (fixed prices for now). Expose `GET /portfolios/{id}/valuation`.
- **Why:** This is what a fund manager looks at. It combines positions, prices and FX, which exercises the whole domain.
- **Done when:** The worked example gives market value 980 and unrealized P&L 180.

**C29 — `feat(app): update positions from trade events`**
- **Do:** On `TradeSettled`, a handler applies the trade to the position. In-process dispatch for now. Add `GET /portfolios/{id}/positions?asOf=` (as-of logic reconstructed from settled trades up to the date).
- **Why:** Positions are derived data. Driving them from events shows the event-driven shape that gets reliable in Phase 3.
- **Done when:** Settling a buy and a sell changes the position correctly, and as-of queries match.

**C30 — `test: add property-based tests with FsCheck`**
- **Do:** Properties: position quantity is never negative across any valid sequence; total cost basis equals the sum of remaining lots; buy then sell of the same quantity at the same price yields realized P&L 0; currency conversion roundtrip stays within tolerance; any sequence of transitions either succeeds legally or throws.
- **Why:** Example tests only cover cases you thought of. Properties let the tool search thousands of random cases and shrink failures to a minimal repro.
- **Done when:** Properties pass, and one deliberately injected bug is caught.

*Stretch:* bond support with accrued interest (day-count conventions such as 30/360 and Actual/365). Treat it as a self-contained commit pair (`feat(domain)` plus `test`).

### Phase 3 — Reliability and financial correctness (C31–C39)

**C31 — `feat(api): add idempotency keys`**
- **Do:** Require an `Idempotency-Key` header on POST endpoints. Store key, request hash, and the response; on repeat, return the stored response. Same key with a different body returns 422. Use a unique constraint to win races.
- **Why:** Networks time out and clients retry. Without this, a retry books a trade twice, which means real money.
- **Done when:** Sending the same request twice creates one trade, and 20 parallel identical requests also create one.

**C32 — `feat(infra): add optimistic concurrency`**
- **Do:** Integer `Version` column as an EF concurrency token on `Trade` and `Position`. Catch `DbUpdateConcurrencyException`, return 409 with guidance to refetch.
- **Why:** Two operators or workers acting on the same trade must not silently overwrite each other. Optimistic locking avoids holding database locks while users think.
- **Done when:** A test with two contexts proves the second save fails.

**C33 — `feat(audit): add append-only audit log`**
- **Do:** `audit_events` table written in the same transaction as the change (actor, event type, JSON payload, timestamp). No update or delete permission for the application role. `GET /trades/{id}/audit`.
- **Why:** Regulators and auditors ask who changed what and when. Writing it in the same transaction means audit and state can't disagree.
- **Done when:** Every lifecycle transition produces an audit row and the API returns the history in order.

**C34 — `feat(infra): add transactional outbox`**
- **Do:** `outbox_messages` table. A `SaveChanges` interceptor converts domain events into outbox rows in the same transaction as the aggregate.
- **Why:** "Save to DB, then publish to a bus" is two operations. A crash between them loses an event or publishes a phantom one. The outbox makes it one atomic write and defers publishing.
- **Done when:** Saving a trade creates the matching outbox row, and a rolled-back transaction leaves no outbox row.

**C35 — `feat(worker): add outbox publisher`**
- **Do:** `BackgroundService` polling unprocessed rows with `SELECT ... FOR UPDATE SKIP LOCKED`, publishing through `IMessagePublisher` (in-memory first), marking processed, retrying with an attempts counter and backoff.
- **Why:** `SKIP LOCKED` lets several workers run safely without double-publishing. Delivery is at-least-once, which leads directly to C36.
- **Done when:** Killing the worker mid-batch and restarting loses nothing.

**C36 — `feat(app): add inbox for idempotent consumers`**
- **Do:** `inbox_messages` table keyed by (message id, consumer). Handlers check and insert in the same transaction as their side effect.
- **Why:** At-least-once delivery means duplicates. Consumers must be idempotent, or a duplicated `TradeSettled` double-counts a position.
- **Done when:** Delivering the same event twice changes the position once.

**C37 — `feat(infra): add resilience policies with Polly`**
- **Do:** `Microsoft.Extensions.Http.Resilience` for outbound calls (retry with jitter, timeout, circuit breaker) and DB transient-error retries via `EnableRetryOnFailure`.
- **Why:** Dependencies fail briefly all the time. Bounded retries with backoff survive blips, while circuit breakers stop a failing dependency from taking everything down.
- **Done when:** Tests with a flaky fake dependency show retries then recovery, and a tripped breaker fails fast.

**C38 — `feat(api): add health checks and graceful shutdown`**
- **Do:** `/health/live` (process up) and `/health/ready` (DB and Redis reachable). Configure host shutdown timeout so in-flight requests and the outbox batch finish.
- **Why:** Orchestrators (Kubernetes, Container Apps) use these to route traffic and restart instances. Graceful shutdown prevents half-finished work during deploys. This is the area you already touched at Entain with EKS health checks.
- **Done when:** Stopping the container mid-request completes the request.

**C39 — `test: add Testcontainers integration test suite`**
- **Do:** `WebApplicationFactory` plus a real Postgres container. Tests: full lifecycle, idempotency under parallel calls, concurrency conflict, outbox atomicity, inbox dedupe. Mark them with a category so CI can run them separately. Docker Desktop must be running on Windows.
- **Why:** Mocks can't prove SQL, constraints, locking or transactions behave. Real-database tests catch the bugs that mocks hide.
- **Done when:** The suite passes locally and in CI.

### Phase 4 — Performance engineering (C40–C46)

Rule for this phase: **measure, change one thing, measure again, record it** in `docs/perf.md`. Never optimize without a baseline.

**C40 — `feat(tools): add synthetic trade generator`**
- **Do:** A console tool using Bogus that writes N trades (default 1,000,000) to CSV with realistic distributions (many portfolios, a few hot instruments, some intentional invalid rows).
- **Why:** Performance work needs realistic volume, and generating data avoids paid market-data feeds.
- **Done when:** It generates 1M rows in under a minute.

**C41 — `feat(import): add naive bulk import as baseline`**
- **Do:** Read the CSV, create each trade through the normal handler, save one at a time. Record throughput.
- **Why:** The baseline is your "before" number. It is also deliberately simple, so you can see which part is slow.
- **Done when:** A baseline (trades per second, memory, DB round trips) is written in `docs/perf.md`.

**C42 — `perf(import): batch writes with Npgsql binary COPY`**
- **Do:** Stream the file, validate in memory, batch (e.g. 10k) and write with `COPY ... FROM STDIN (FORMAT BINARY)` (the Postgres equivalent of `SqlBulkCopy`). Stage in a temp table, then insert and handle duplicates.
- **Why:** The baseline is dominated by per-row round trips and change tracking. Bulk copy cuts both.
- **Done when:** Throughput improves by an order of magnitude and results are identical to the baseline.

**C43 — `perf(import): add Channels producer/consumer pipeline`**
- **Do:** `System.Threading.Channels`: one reader producing parsed rows, N validators in parallel, a single batching writer. Bounded channels for back-pressure.
- **Why:** Overlaps I/O and CPU work. Bounded channels stop a fast producer from exhausting memory when the writer is slower.
- **Done when:** Memory stays flat on 1M rows, and throughput scales to the point where the database is the bottleneck.

**C44 — `perf(infra): tune queries and indexes`**
- **Do:** Find N+1 queries (log EF SQL), apply `AsNoTracking`, projection (`Select`) instead of loading entities, compiled queries on hot paths, add or fix indexes using `EXPLAIN (ANALYZE, BUFFERS)`.
- **Why:** Most real-world slowness is queries and indexes, not C#. Reading query plans is a senior-level skill.
- **Done when:** Position and search endpoints show before/after timings, and the plan changes from sequential scan to index scan where expected.

**C45 — `perf(infra): add Redis cache-aside for reference data`**
- **Do:** Cache instruments and FX rates with `IDistributedCache` or StackExchange.Redis, TTL plus explicit invalidation when data changes, and protection against cache stampede (a lock or `Lazy` per key).
- **Why:** Reference data is read constantly and changes rarely. Invalidation is the hard part, so you should deliberately practice it.
- **Done when:** Cache hit rate and latency are measured, and updating a rate invalidates the entry.

**C46 — `perf: add BenchmarkDotNet suite and perf report`**
- **Do:** Benchmarks for `Money` arithmetic, FIFO sells with many lots, rule pipeline, and valuation. Use `[MemoryDiagnoser]` and look at allocations. Try `Span<T>`/`ArrayPool` where parsing allocates. Finish `docs/perf.md` with a before/after table.
- **Why:** Micro-benchmarks find allocation and CPU hot spots that production profiling hides, and they give you the numbers for your resume.
- **Done when:** `docs/perf.md` has a table you could present in an interview.

### Phase 5 — Azure and cloud (C47–C53)

**C47 — `chore: add Dockerfile and full docker-compose`**
- **Do:** Multi-stage Dockerfile for Api and Worker (restore, publish, runtime image, non-root user). Compose runs Api, Worker, Postgres, Redis.
- **Why:** Containers make local, CI and cloud identical. Multi-stage builds keep images small and free of the SDK.
- **Done when:** `docker compose up` runs the whole system.

**C48 — `infra: add Bicep templates`**
- **Do:** Bicep modules for resource group resources: Container Apps environment, Azure Container Registry, Azure Database for PostgreSQL Flexible Server, Key Vault, Service Bus namespace and queue/topic, Log Analytics and Application Insights. Use the free or low-cost tiers and tear down after sessions.
- **Why:** Infrastructure as code makes environments reproducible and reviewable. Equivalent of Terraform or CloudFormation you may know.
- **Done when:** `az deployment group create` stands up the environment from nothing.

**C49 — `feat(infra): use Key Vault and Managed Identity`**
- **Do:** Replace connection-string secrets with Key Vault references and `DefaultAzureCredential`. Assign roles to the app's managed identity.
- **Why:** The best secret is the one that doesn't exist. Managed identity removes passwords from config, a core cloud-security practice.
- **Done when:** The deployed app runs with no secrets in env vars or repo.

**C50 — `feat(messaging): publish outbox to Azure Service Bus`**
- **Do:** Implement `IMessagePublisher` with `Azure.Messaging.ServiceBus`, set message IDs for deduplication, add a consumer service using the inbox from C36. Keep the in-memory publisher for local and tests.
- **Why:** Swapping the transport without touching domain code proves the architecture holds. Service Bus is Azure's equivalent of SQS/SNS, and supports dead-lettering for poison messages.
- **Done when:** An event flows from outbox → Service Bus → consumer → position update, and a poison message lands in the dead-letter queue.

**C51 — `feat(obs): add OpenTelemetry tracing and metrics with Application Insights`**
- **Do:** OpenTelemetry for ASP.NET Core, HttpClient, Npgsql and custom `ActivitySource` spans around lifecycle steps. Metrics: trades captured, rule rejections, outbox lag, settlement failures. Export through the Azure Monitor exporter. You already introduced OpenTelemetry at work, so focus on custom spans and business metrics.
- **Why:** You can't operate what you can't see. Traces show where a request spent time across API, DB and bus.
- **Done when:** One trace shows request → DB → outbox → consumer, and a dashboard shows outbox lag.

**C52 — `ci: add CD pipeline to Azure Container Apps`**
- **Do:** GitHub Actions: build, test, push image to ACR, deploy with a revision, run a smoke test against `/health/ready`, and support rollback to the previous revision. Use OIDC federated credentials (no stored secrets).
- **Why:** Repeatable automated deployment is the difference between a project and a service.
- **Done when:** Merging to `main` deploys, and a bad revision can be rolled back.

**C53 — `refactor: extract positions into a separate deployable`**
- **Do:** Move the Positions module's event consumer and projection into its own Worker deployment (same repo), communicating only through Service Bus. Keep the shared contracts in a small `TradeFlow.Contracts` project.
- **Why:** You now feel the real cost of a split (eventual consistency, versioned contracts, deployment coordination) and the benefit (independent scaling). Record the verdict in an ADR.
- **Done when:** Positions update correctly with the Api and the Positions worker deployed separately.

### Phase 6 — Functional style and design leadership (C54–C58)

**C54 — `refactor(domain): make valuation and P&L pure functions`**
- **Do:** Pull calculations into static, side-effect-free functions over immutable inputs (`ImmutableArray`, records). No I/O or clock inside.
- **Why:** Pure functions are trivially testable, parallelizable and safe to reason about. This is the mindset behind array and functional languages like APL.
- **Done when:** Existing tests pass unchanged and the functions have no dependencies.

**C55 — `perf: add SIMD and array-oriented valuation benchmark`**
- **Do:** Compute valuation of 1M positions as `quantity[] * price[] * fx[]` three ways: naive loop, LINQ, `System.Numerics.Vector<T>` (use `double` or scaled `long` for the SIMD path, noting the precision tradeoff in an ADR). Benchmark all three.
- **Why:** Operating on whole arrays at once is the core idea of APL. This is the closest C# gets, and you'll understand the language you'd meet on the job.
- **Done when:** Benchmark shows the speedup and the ADR explains when `decimal` is still the right choice.

**C56 — `feat(fsharp): add optional F# pricing module`** *(optional)*
- **Do:** A small F# project for pricing/valuation with discriminated unions and pipelines, called from C#.
- **Why:** Gives hands-on feel for functional style, algebraic data types and pattern matching, which is useful if the codebase mixes paradigms.
- **Done when:** The C# app calls the F# function and a test covers it.

**C57 — `docs: add ADRs for key decisions`**
- **Do:** One ADR each: modular monolith first, EF Core + Dapper split, decimal over double (and the SIMD exception), outbox over dual-write, idempotency design, Postgres choice, Service Bus, the extraction in C53.
- **Why:** Each ADR is a short design-review artifact showing options considered and tradeoffs. It's visible design leadership.
- **Done when:** Each ADR has context, decision, alternatives and consequences.

**C58 — `docs: add CONTRIBUTING, PR template and final README`**
- **Do:** `CONTRIBUTING.md` (setup, branching, commit style, how to add a rule), a PR template with checklist, final README with architecture diagram, perf results, how to run locally and deploy.
- **Why:** Writing as if onboarding a teammate shows mentoring ability and makes the project demonstrable in minutes.
- **Done when:** Someone unfamiliar can clone, run and understand it from the README alone.

---

## 6. Definition of done (every commit)

- [ ] Builds with zero warnings
- [ ] New behavior has tests, and all tests pass
- [ ] Architecture tests still pass
- [ ] No secrets, no `double` for money, no direct `DateTime.UtcNow` in domain/application code
- [ ] Commit message follows Conventional Commits
- [ ] You can explain in one minute why this commit exists

## 7. Interview talking points you'll earn

- "Why decimal and a Money type?" (C05)
- "How do you stop a trade being booked twice?" (C31)
- "How do you publish events reliably?" (C34–C36)
- "Tell me about a performance problem you fixed." (C40–C46, with numbers)
- "How would you move from a monolith to services?" (C53, C57)
- "How would you deploy and observe this in Azure?" (C47–C52)
- "What's different about array-oriented thinking?" (C54–C55)

## 8. Cautions

- Don't build everything at once. Finish a phase before starting the next, and prefer a smaller finished project over a larger abandoned one.
- Real platforms are far bigger, and SimCorp's internals differ. Treat this as practice with the same classes of problems.
- Azure can cost money. Use low tiers, set budget alerts, and delete resource groups after sessions.
- If a commit takes more than a few days, split it.
