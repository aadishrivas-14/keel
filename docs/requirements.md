# Requirements

This file is the single source of truth for what Keel must do. The design overview summarises it; if the two disagree, this file wins.

## How to read this spec

- **IDs.** `FR-` functional, `NFR-` non-functional. Letter after the dash = component: `P` platform, `A` runtime, `G` MCP gateway, `C` chaos harness, `X` PII proxy, `D` drift guard. IDs are never reused; a dropped requirement is marked Dropped, not deleted.
- **Priority.** Must = v1 fails without it. Should = planned for v1, can slip. Could = only if time allows.
- **Acceptance criteria.** Each one is testable and maps to a test (`T-<ID>`) in [test-plan.md](test-plan.md).
- **Numbers are targets**, not measured results, until a proof test reports them.
- **Engine-agnostic.** Runtime requirements describe behaviour, not implementation, so they hold whichever engine ADR-001 lands on.

## Platform-wide requirements

These apply to every component and are checked at each component's Definition of Done.

| ID | Requirement | Priority | Acceptance criteria |
| --- | --- | --- | --- |
| FR-P1 | Each component runs standalone and all run together | Must | `docker compose up` starts the full stack on a 16 GB laptop; each component also starts alone with its own compose profile |
| FR-P2 | One trace per agent run across components | Should | A single run shows runtime, gateway and proxy spans under one OpenTelemetry trace ID |
| FR-P3 | Infrastructure as code for AWS | Must | `terraform apply` creates the demo environment; `terraform destroy` removes everything; no console-only resources |
| NFR-P1 | Secrets never in code or images | Must | Secret scanning in CI finds nothing; secrets come from env vars or AWS Secrets Manager |
| NFR-P2 | Encryption in transit | Must | All service-to-service traffic uses TLS; gateway clients use mTLS or OIDC |
| NFR-P3 | Metrics for every service | Must | Each service exposes Prometheus metrics: request rate, errors, latency histogram |
| NFR-P4 | Structured logs | Must | JSON logs with run ID and trace ID; no PII in logs (verified by the PII proxy's detectors in CI) |
| NFR-P5 | Reproducible builds | Should | Pinned dependency versions; container images built in CI only |
| NFR-P6 | Cost guardrail | Must | AWS budget alarm at US$20/month; demo environment torn down after each benchmark run |
| NFR-P7 | Documentation per component | Must | README with quick start under 5 commands, architecture diagram, and the latest proof-test numbers with date and hardware |
| NFR-P8 | Licence and provenance | Must | Licence per ADR-008; no employer code, data or telecom functionality |

## A. Durable runtime

| ID | Requirement | Priority | Acceptance criteria |
| --- | --- | --- | --- |
| FR-A1 | Start, get, cancel, list runs over HTTP/gRPC | Must | API contract tests pass for all four calls, including cancel of a paused run |
| FR-A2 | Every step persisted as an ordered event per run | Must | Event log for a run replays to the same final state in every test run |
| FR-A3 | Resume from last completed step after worker death | Must | Killing the worker mid-run: run completes; completed model calls are not re-sent (mock model call count unchanged) |
| FR-A4 | Effects ledger: no repeated side effect on retry | Must | Kill test: zero duplicates at the mock email and payment services; an in-flight call without Idempotency-Key support ends in `unknown`, never silently retried |
| FR-A5 | Pause for approval, resume on signal | Must | A run waits on approval for at least 24 h of simulated time and resumes on signal |
| FR-A6 | Agent-to-agent handoff over Kafka topics | Should | Two agents complete a handoff task; killing either during handoff loses no work |
| FR-A7 | Python SDK wraps LangGraph or a plain loop | Must | Existing example agent runs durably with a decorator and under 10 changed lines |
| NFR-A1 | Recovery time | Must | Run resumes within 5 s of worker death (p95 over 100 kills) |
| NFR-A2 | Overhead per step | Should | Under 20 ms p99 added per step, excluding model and tool time |
| NFR-A3 | Ordering | Must | Strict order within a run; no ordering promised across runs |

## 1. MCP gateway

| ID | Requirement | Priority | Acceptance criteria |
| --- | --- | --- | --- |
| FR-G1 | Proxy MCP over stdio and Streamable HTTP | Must | Reference MCP client works through the gateway on both transports with no client changes |
| FR-G2 | Pin and sign tool definitions | Must | Changing an upstream tool's description blocks its calls until re-approved; the diff is shown |
| FR-G3 | Per-call policy on identity, user, tool, arguments, labels | Must | Policy test suite: every allow and deny case returns the expected decision and reason |
| FR-G4 | Mint per-call scoped token | Must | Upstream rejects a replayed token, an expired token (TTL under 60 s) and a token used with different arguments |
| FR-G5 | Taint tracking gates high-risk tools | Must | After an untrusted read, `send_email` needs approval; without the read it does not |
| FR-G6 | Scan tool output for injection | Should | Known injection strings in tool output raise a policy signal; scanner alone never blocks a clean call |
| FR-G7 | Hash-chained audit log | Must | Editing or deleting any audit record breaks chain verification |
| FR-G8 | Approvals routed to runtime | Should | Approval request appears as a runtime signal; approve and deny both resume correctly |
| NFR-G1 | Added latency | Must | Under 10 ms p99 per call with policy cached |
| NFR-G2 | Fail closed | Must | Policy or key service down: every call denied with a clear reason |
| NFR-G3 | Key rotation | Must | Keys rotate every 24 h; tokens signed with the previous key verify until expiry |

## B. Chaos harness

| ID | Requirement | Priority | Acceptance criteria |
| --- | --- | --- | --- |
| FR-C1 | Inject all listed fault types | Must | Each fault type has a test that shows the agent saw it |
| FR-C2 | Seeded YAML scenarios | Must | Same seed and mocked model: identical fault sequence on two runs |
| FR-C3 | Adapters for 3 or 4 agent setups | Must (3), Could (4th) | Each adapter runs the full task suite end to end |
| FR-C4 | Record success, recovery, duplicates, unsafe actions, extra cost | Must | Every metric present per run in the results file |
| FR-C5 | Markdown report and comparison table | Must | Report generated from the results file with no manual edits |
| NFR-C1 | CI runtime | Should | 50-task suite under 30 min with a mocked model |
| NFR-C2 | Reproducibility with a real model | Should | Success rate within 2 points across 3 reruns |

## 2. PII proxy

| ID | Requirement | Priority | Acceptance criteria |
| --- | --- | --- | --- |
| FR-X1 | OpenAI- and Anthropic-compatible proxy, with streaming | Must | Official SDKs work by changing only the base URL |
| FR-X2 | Kafka mode | Should | Messages on the input topic appear redacted on the output topic, order kept per key |
| FR-X3 | Structured Indian identifiers with checksums | Must | Detects every type in the test set; checksum-invalid lookalikes are not flagged |
| FR-X4 | Names and addresses in English, Hindi, Hinglish | Must | Meets NFR-X2 on each language slice separately |
| FR-X5 | Stable typed reversible tokens | Must | Same value gives same token within a tenant; response text is restored exactly |
| FR-X6 | Purpose and consent check | Should | Request without a matching consent record is refused with a reason |
| FR-X7 | Audit and erasure | Must | Erasure removes all vault entries for a data principal; later restores return the token, not the value |
| NFR-X1 | Added latency | Should | Under 50 ms p99 for a 2,000-token prompt |
| NFR-X2 | Detection quality | Must | Recall above 95%, precision above 90% on the synthetic test set |
| NFR-X3 | Vault security | Must | AES-256-GCM envelope encryption; key rotation with no downtime |

## 4. Schema-drift guard

| ID | Requirement | Priority | Acceptance criteria |
| --- | --- | --- | --- |
| FR-D1 | Sample JSON topics; use Schema Registry when present | Must | Read-only consumer group; producers unaffected |
| FR-D2 | Infer schema over a rolling window | Must | Inferred schema matches the generator's schema for the test topics |
| FR-D3 | Classify breaking vs additive | Must | All 40 seeded changes classified correctly |
| FR-D4 | Map consumers to fields | Should | For the sample repos, every field access tied to a topic is found |
| FR-D5 | Alert with services, code locations, explanation | Must | Alert names the right services and files for each seeded break |
| FR-D6 | CI mode for producer PRs | Could | A PR that removes a field fails the check with the affected consumers listed |
| FR-D7 | Watch Keel's own topics | Should | Running against `runs.events` raises no false alarms over a 24 h soak |
| NFR-D1 | Alert latency | Must | Under 60 s from first drifted message |
| NFR-D2 | Sampling overhead | Should | Under 1% of topic throughput |
