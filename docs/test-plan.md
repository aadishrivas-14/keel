# Test plan

Every Must requirement has at least one automated test before its component is called done, and every number published in a README comes from a proof test in this plan.

## Test strategy

| Level | What it checks | Tools | Runs |
| --- | --- | --- | --- |
| Unit | Pure logic: policy evaluation, schema inference, detectors, ledger state machine | Go `testing`, pytest | Every commit |
| Contract | Public APIs and protocols: runtime API, MCP over both transports, OpenAI/Anthropic proxy shape | OpenAPI/gRPC schema tests, MCP reference client, official SDKs | Every PR |
| Integration | One component with real dependencies | Testcontainers (Redpanda), SQLite in-process | Every PR |
| End to end | Full stack with example agents | Docker Compose, scripted mock model | Every PR to main |
| Fault | Worker kills, network faults, dependency outages | Kill script, Toxiproxy | Nightly and before each milestone |
| Security | Policy cases, injection benchmark, secrets, dependencies | Policy test suite, AgentDojo, gitleaks, govulncheck, pip-audit | PR (fast checks), milestone (benchmark) |
| Performance | Latency and throughput targets | k6, ghz | Milestone only, on AWS |
| Model quality | PII detection recall and precision | Labelled synthetic set, pytest report | PRs touching detectors |

**Environments.** Local (Compose), CI (GitHub Actions, no cloud), and an ephemeral AWS environment created by Terraform for performance runs and destroyed after.

**Mock model.** A scripted fake LLM server returns fixed responses per test. It makes runs deterministic, free, and fast; real-model runs are limited to NFR-C2 and the injection benchmark.

## Proof tests and benchmark rules

Proof tests produce the numbers used in product claims, benchmarks and READMEs. Each has a fixed script in the repo so any user can rerun it and verify the result.

| Test ID | Proves | Setup | Pass when | Number published |
| --- | --- | --- | --- | --- |
| PT-A1 | Runtime survives worker loss without duplicates | 1,000 runs, each sends a mock email and a mock payment; kill 30% of workers at random | All runs complete; zero duplicates at both mocks | Runs completed, duplicates, p95 recovery time |
| PT-A2 | Side-effect safety beats a plain durable engine | Same kill test on the chosen engine without the ledger | Ledger version has fewer duplicates or `unknown` states, never more | Duplicates with and without ledger |
| PT-G1 | Gateway blocks injection attacks | Injection benchmark (AgentDojo candidate), with and without gateway, same model | Attack success drops; task utility loss reported | Attack success and utility, both ways |
| PT-G2 | Gateway overhead | 10k calls at fixed rate on AWS, policy cached | p99 added latency under 10 ms | p50 and p99 added latency |
| PT-C1 | Harness compares agent setups fairly | 50 tasks x 3 or 4 setups x fixed fault plan, 3 seeds | Report generated with no manual edits | Per-setup success, recovery, duplicates, unsafe actions |
| PT-X1 | PII detection quality | Synthetic labelled set, per language slice | Recall above 95%, precision above 90% | Recall and precision per slice |
| PT-X2 | PII proxy overhead | 2,000-token prompts at fixed rate | p99 under 50 ms | p50 and p99 added latency |
| PT-D1 | Drift guard catches breaks | Replay a recorded topic with 20 breaking and 20 additive seeded changes | All breaking caught; false alarm rate reported | Caught, missed, false alarms, alert latency |

**Benchmark rules**

- State hardware, instance type, versions and date next to every number.
- Run each benchmark 3 times; publish the median and note the spread.
- Warm up before measuring; exclude model latency from overhead numbers.
- Publish failures and regressions too. A weak number with an honest setup still builds user trust.
- Keep raw results files in the repo under `results/<date>/`.

## Traceability

Each requirement's test is named `T-<requirement ID>` (for example `T-FR-A3`), so the link is never lost. The table shows the level each test runs at and which proof test also covers it.

| Requirement | Test level | Also covered by |
| --- | --- | --- |
| FR-P1 | End to end | All proof tests |
| FR-P2 | End to end | PT-A1 |
| FR-P3 | Manual check at milestone (apply, destroy) | PT-G2, PT-X2 |
| NFR-P1, NFR-P5 | CI checks | |
| NFR-P2, NFR-P3, NFR-P4 | Integration | |
| NFR-P6, NFR-P7, NFR-P8 | Milestone checklist | |
| FR-A1 | Contract | |
| FR-A2, NFR-A3 | Integration | PT-A1 |
| FR-A3, FR-A4, NFR-A1 | Fault | PT-A1, PT-A2 |
| FR-A5, FR-A6, FR-A7 | End to end | |
| NFR-A2 | Performance | |
| FR-G1 | Contract | |
| FR-G2, FR-G3, FR-G4, FR-G5, FR-G7 | Unit + integration | PT-G1 (G3, G5) |
| FR-G6 | Unit | PT-G1 |
| FR-G8 | End to end | |
| NFR-G1 | Performance | PT-G2 |
| NFR-G2, NFR-G3 | Fault | |
| FR-C1, FR-C2 | Integration | PT-C1 |
| FR-C3, FR-C4, FR-C5 | End to end | PT-C1 |
| NFR-C1, NFR-C2 | CI timing, real-model rerun | PT-C1 |
| FR-X1 | Contract | |
| FR-X2 | Integration | |
| FR-X3, FR-X4, NFR-X2 | Model quality | PT-X1 |
| FR-X5, FR-X6, FR-X7, NFR-X3 | Unit + integration | |
| NFR-X1 | Performance | PT-X2 |
| FR-D1, FR-D2, FR-D3 | Unit + integration | PT-D1 |
| FR-D4, FR-D5, FR-D6 | End to end with sample repos | PT-D1 |
| FR-D7 | 24 h soak | |
| NFR-D1, NFR-D2 | Performance | PT-D1 |

A CI job fails if any Must requirement in requirements.md has no `T-` test in the repo.

## Test data rules

- Synthetic data only. Never real personal data, never employer data, never production logs.
- Indian identifiers in tests are generated to pass checksums but are not real numbers; the generator is in the repo.
- Mock services (email, payments, file system) count every call so duplicates are measured, not inferred.
- Recorded Kafka topics used for replay are generated by the repo's own producers.
- Benchmark datasets keep their original licence; check it before committing any copy.

## Entry and exit criteria

**A component enters testing when**

- [ ] Its requirements are in requirements.md with acceptance criteria.
- [ ] Its test cases (`T-` IDs) are written down, even if not yet automated.
- [ ] Mocks and test data it needs exist.

**A component exits (milestone done) when**

- [ ] All Must tests pass in CI three runs in a row.
- [ ] Its proof tests have run and results are committed with date and hardware.
- [ ] No open critical or high security finding.
- [ ] Flaky tests are fixed or quarantined with an issue, never silently retried.
