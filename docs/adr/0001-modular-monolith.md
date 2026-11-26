# ADR-0001: Start as a modular monolith

- **Status:** Accepted
- **Date:** 2026-11-26

## Context

TradeFlow captures trades, runs them through a lifecycle, and derives positions, valuation and an audit trail. It is a long-lived, correctness-critical kind of system, but it is being built from scratch by one developer who is new to the finance domain.

Two facts drive the decision:

1. **Boundaries are guessed early and guessed wrong.** Where trading ends and positions begin, or what belongs in audit versus reporting, only becomes clear with use. Changing a boundary is cheap inside one codebase and expensive across service contracts.
2. **Distribution has a fixed cost.** Splitting into services brings network failure, eventual consistency, distributed transactions, versioned contracts, and more to deploy and monitor. That cost is paid whether or not the split brings any benefit.

The system does have real module concerns (`Trading`, `Positions`, `Audit`, `Reporting`) that we want to keep apart.

## Decision

Build **one deployable unit** (plus a background Worker host sharing the same code) with **strict internal modules** and Clean/Onion layering:

- Layers: `Api` and `Infrastructure` depend on `Application`, which depends on `Domain`. `Domain` references nothing.
- Modules (`Trading`, `Positions`, `Audit`, `Reporting`) are organised as feature folders inside each project.
- Modules communicate only through Application abstractions or domain events, never through each other's database tables or internals.
- The boundaries are enforced mechanically: project references in the compiler, and architecture tests (NetArchTest) from C15.

## Alternatives considered

- **Microservices from day one.** Rejected. The strongest argument for it is independent scaling and deployment, but we have no scaling problem and no second team. It would front-load distributed-system costs before we know where the boundaries are, and slow down learning the domain.
- **A plain layered monolith with no module boundaries.** Rejected. It is the simplest start, but without enforced boundaries, modules tend to couple through shared tables and classes, which makes a later split or even a refactor painful.

## Consequences

**Good**

- Simple build, deployment and local development (one process, one database).
- In-process calls and a single transaction: audit and outbox rows can be committed atomically with the aggregate.
- Wrong boundaries are cheap to correct with a refactor.
- A clean path to extract a module later, because modules already talk only through abstractions and events.

**Costs and risks**

- One scaling unit: the whole application scales together.
- A shared database. Module separation depends on discipline and architecture tests, not on a physical wall.
- A bad deployment affects everything.

**Follow-up**

In C53, the Positions module is deliberately extracted into its own deployable, communicating through Service Bus. The experience, including eventual consistency and contract versioning, will be recorded in a later ADR as the verdict on this decision.
