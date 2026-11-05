# Architecture decision records

Eight of nine decisions are accepted (reviewed 2026-11-05). The licence (ADR-008) is left open until the repo is published. A changed decision gets a new ADR that supersedes the old one; accepted records are never edited.

| ADR | Decision | Status | Blocks |
| --- | --- | --- | --- |
| [ADR-001](0001-runtime-on-existing-engine.md) | Build the runtime on Restate or DBOS; original work = effects ledger + Kafka coordination | Accepted | Component A |
| [ADR-002](0002-kafka-backbone-redpanda-local.md) | Kafka-compatible backbone; Redpanda for local dev | Accepted | All |
| [ADR-003](0003-language-split.md) | Go for gateway and drift sampler; Python for SDK, PII proxy, harness | Accepted | All |
| [ADR-004](0004-cel-policy-language.md) | CEL as the gateway policy language | Accepted | Component 1 |
| [ADR-005](0005-per-call-tokens.md) | Per-call tokens are asymmetric-signed JWTs with a JWKS, rotated every 24 h | Accepted | Component 1 |
| [ADR-006](0006-lightweight-state-store.md) | Lightweight embedded store (SQLite) first; Postgres or Redis only on a measured need | Accepted | All |
| [ADR-007](0007-monorepo-split-ready.md) | One monorepo, built so any component can be split out later | Accepted | All |
| [ADR-008](0008-licence-at-publish.md) | Licence chosen at publish time (Apache 2.0 or MIT) | Proposed | Publishing |
| [ADR-009](0009-documentation-first.md) | Documentation first; no code before Definition of Ready | Accepted | All |

Template for new records: Status, Date, Context, Decision, Consequences.
