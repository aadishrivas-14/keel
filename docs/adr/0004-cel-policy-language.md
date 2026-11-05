# ADR-004: CEL for gateway policy

- **Status:** Accepted
- **Date:** 2026-11-05

## Context

Options are CEL and OPA/Rego. Agent Router (formerly Envoy AI Gateway), the closest open-source peer, uses CEL for per-tool MCP rules.

## Decision

CEL. It embeds in Go, evaluates fast, and is easier to read in policy reviews.

## Consequences

- Less familiar to teams standardised on OPA.
- Revisit if policies need data joins that Rego handles better.
