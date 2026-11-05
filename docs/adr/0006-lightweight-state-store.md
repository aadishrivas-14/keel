# ADR-006: Lightweight state store first

- **Status:** Accepted
- **Date:** 2026-11-05

## Context

Keel needs to store the effects ledger, tool manifests, consent records and the PII vault. Running Postgres from day one adds setup for a single-developer project, and a cache is not yet known to be needed.

## Decision

Start with SQLite (WAL mode) behind one storage interface per component, with separate database files for the ledger and the PII vault. Add Redis only when a measurement shows it is needed, most likely for the token replay cache (ADR-005) or the policy cache. Move to Postgres when a trigger below fires.

## Triggers to move to Postgres

- Workers run on more than one host (the AWS runs of PT-A1 and PT-G2).
- DBOS is chosen in ADR-001 and needs Postgres for its own state.
- Write contention shows up in a proof test.

## Consequences

- Nothing extra to run locally or in CI.
- SQLite only allows one writer at a time and its file cannot be shared safely across hosts, so the multi-host kill test will force the Postgres move; the storage interface keeps that move to one adapter.
- The vault keeps its own file and encryption keys so a ledger bug cannot read PII.
- If Restate is chosen, its server keeps its own journal and the engine needs no database from Keel.
