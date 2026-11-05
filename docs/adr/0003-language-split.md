# ADR-003: Language split

- **Status:** Accepted
- **Date:** 2026-11-05

## Context

The gateway and drift sampler are latency-sensitive and benefit from a compiled, low-overhead language; the agent and NLP ecosystems are in Python.

## Decision

Go for the MCP gateway and the drift sampler, where latency matters. Python for the SDK, PII proxy, chaos harness and drift explainer. The runtime's language follows the engine chosen in ADR-001.

## Consequences

- Two toolchains in CI.
- Shared contracts (event schemas, audit records) are defined once as JSON Schema and code-generated for both languages.
