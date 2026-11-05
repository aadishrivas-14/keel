# ADR-001: Build the runtime on an existing durable engine

- **Status:** Accepted
- **Date:** 2026-11-05

## Context

Prior art (see ../prior-art.md) shows durable agent execution is solved by Temporal, Restate, DBOS and Inngest. None of them handles a side effect interrupted mid-call; all leave idempotency to the user.

## Decision

Use Restate or DBOS for step persistence and recovery. Keel's original work is the effects ledger (FR-A4), Kafka-based agent handoff (FR-A6) and the SDK wrapper (FR-A7).

The engine is chosen by a one-week evaluation after documentation, scored on:

- self-hostable
- licence
- Python and Go SDK support
- how a step boundary can host the ledger hook
- fit with a lightweight store (ADR-006) and Kafka
- ease of running the PT-A1 kill test

## Consequences

- Component A shrinks from about 5 weeks to about 3 (estimate).
- The pitch moves from "durable agents" to "side-effect safety for agents".
- PT-A2 compares duplicates with and without the ledger on the same engine.
- Keel inherits the engine's determinism rules for agent code.
