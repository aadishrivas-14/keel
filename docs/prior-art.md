# Prior art

Searched 2026-11-05. Feature claims come from project pages and docs, not from running the tools.

Both spaces are more crowded than the first design assumed: durable agent execution is a solved category, and MCP gateways with per-tool policy and tool pinning already exist. What was not found is the combination Keel proposes: per-call minted tokens, taint-driven approvals, and an effects ledger that surfaces uncertain side effects.

## Durable agent runtime

| Project | Type | How it survives a crash | Side effects mid-crash |
| --- | --- | --- | --- |
| Temporal | Open source, mature | Replays workflow code against a stored event history; model and tool calls run as Activities and are not repeated on replay | Not exactly-once; idempotency left to the user |
| Restate | Open source, single binary | Journals each step before running it; skips journalled steps on restart | Not covered |
| DBOS | Open source library on Postgres | Resumes from the last completed step; step output commits with its rows | Not covered |
| Inngest | Hosted + open source | Re-runs the function, injecting saved results by step ID | Not covered |
| LangGraph | Open source agent framework | Checkpoints graph state; a resumed node restarts from its beginning | Code before an interrupt runs again |
| Orca Agent Engine (StreamNative) | Commercial, private preview | Agents on Kafka or Pulsar streams; replay for debugging and audit | Tool gateway with auth and rate limits; idempotency not described |
| Agentic Streaming (GitHub, Ugbot) | Open source, small | Event-sourced actors (Pekko) per conversation over Kafka | Says idempotent handling is needed; not verified against the repo |

None of these engines claims exactly-once external effects. A crash after a tool acts but before its result is saved can repeat the action, and every source leaves idempotency to the user. That gap is where FR-A4 and the effects ledger's `unknown` state matter.

## Zero-trust MCP gateway

| Project | Type | Runtime proxy | Per-tool policy | Tool pinning | Audit |
| --- | --- | --- | --- | --- | --- |
| Agent Router (formerly Envoy AI Gateway) | Open source | Yes | Yes, ordered rules with CEL over tool and parameters; OAuth with PKCE, JWT scope checks | Not mentioned | OpenTelemetry traces and metrics; no audit log described |
| Lasso MCP Gateway | Open source (MIT, Python) | Yes | Partial: natural-language custom policy through Lasso's plugin | Not mentioned | Logs requests and responses per guardrail |
| MCP-Scan (Invariant Labs, now part of Snyk) | Open source scanner | No, scans on demand; runtime guardrails product is separate | No | Yes, hashes descriptions to catch rug pulls | No; sends tool names and descriptions to Invariant's API |
| MCP Guardian (eqtylab), Cisco MCP Scanner, MCP-Shield, mcp-pin | Open source | Mixed | Mixed | mcp-pin pins and blocks changed tools | Not checked |

## Where Keel still differs

- **Per-call minted tokens.** Existing gateways check the caller's token or inject a static upstream key. None found mints a short-lived token scoped to one call's arguments that the upstream server verifies (FR-G4).
- **Taint-driven approvals.** None found marks a session tainted after reading untrusted content and then gates high-risk tools on human approval (FR-G5).
- **Uncertain side effects made visible.** Durable engines skip finished steps but leave in-flight effects to the user. Keel's ledger turns them into an explicit `unknown` state with compensation or approval (FR-A4).
- **Runtime and gateway together.** Gateway approvals pause and resume a durable run; the chaos harness injects faults at the gateway. Orca is the closest to this combination and is commercial.

## Follow-ups

- [x] Build component A on Restate or DBOS (ADR-001).
- [ ] Pitch the runtime as side-effect safety, measured against the plain engine (PT-A2).
- [ ] Present tool pinning as table stakes; lead with per-call tokens and taint tracking.
- [ ] Compare the gateway's CEL rules with Agent Router's.
- [ ] Check the Agentic Streaming and mcp-pin repos directly before building.

## Sources

- [Cadenya: durable execution for AI agents compared](https://cadenya.com/other-content/durable-execution-for-ai-agents)
- [StreamNative Orca Agent Engine](https://streamnative.io/products/orca-agent-engine)
- [Agent Router MCP docs](https://theagentrouter.ai/docs/capabilities/mcp/)
- [Lasso MCP Gateway](https://github.com/lasso-security/mcp-gateway)
- [Invariant Labs: introducing MCP-Scan](https://invariantlabs.ai/blog/introducing-mcp-scan)
