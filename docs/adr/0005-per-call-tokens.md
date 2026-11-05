# ADR-005: Per-call tokens

- **Status:** Accepted
- **Date:** 2026-11-05

## Context

FR-G4 needs short-lived tokens bound to one call's arguments that upstream servers can verify without calling back.

## Decision

JWT signed with an asymmetric key (exact algorithm chosen at design time), with claims for audience, tool, argument hash, and a unique ID for replay checks. TTL under 60 s. Public keys published as a JWKS and rotated every 24 h.

## Consequences

- Upstream servers need a small verifier library, which Keel ships.
- Replay protection needs a short-lived seen-ID cache (a candidate for Redis under ADR-006, only if measured).
