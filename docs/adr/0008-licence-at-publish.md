# ADR-008: Licence chosen at publish time

- **Status:** Proposed (deferred)
- **Date:** 2026-11-05

## Context

The repo stays private until component A is done (ways-of-working.md), so the licence does not block any work now.

## Decision

Deferred. Candidates are Apache 2.0 (explicit patent grant, common for infrastructure companies adopt) and MIT (shorter, no patent grant). Decide before the first public push.

## Consequences

- Until then, check every third-party dependency against both licences.
- Add licence headers only once the choice is made.
