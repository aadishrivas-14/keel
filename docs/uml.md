# Keel UML diagrams

UML models derived from the design docs ([design-overview.md](design-overview.md),
[requirements.md](requirements.md), [adr/](adr/)) as of the 2026-11-05 snapshot.

All diagrams are PlantUML. Render with the PlantUML CLI, the server, or any IDE
plugin. These model the *intended* design — the repo is documentation-only, so no
code is reflected, only the specified behaviour and contracts.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## 1. Platform component diagram

How the five components wire together around the Kafka backbone. The runtime is
central: model calls exit via the PII proxy, tool calls via the MCP gateway, every
step and audit record lands on Kafka, and the drift guard watches the topics. The
chaos harness has no data path of its own — it drives fault hooks on the two guards.

```plantuml
@startuml platform-components
title Keel — platform component diagram

skinparam componentStyle rectangle
skinparam shadowing false

actor "Agent code\n(LangGraph / plain loop)" as Agent

package "Keel platform" {

  component "Python SDK\n(decorator wrapper)" as SDK

  component "[A] Durable runtime\n(engine: Restate/DBOS)" as Runtime
  component "[1] Zero-trust\nMCP gateway (Go)" as Gateway
  component "[2] DPDP PII proxy\n(FastAPI)" as PII
  component "[4] Schema-drift guard\n(Go sampler + Py explainer)" as Drift
  component "[B] Chaos harness\n(Python)" as Chaos

  queue "Kafka / Redpanda\nruns.*, tools.*, audit" as Kafka

  database "SQLite stores\nledger | manifests\nconsent | PII vault" as Store
}

cloud "External LLM\n(OpenAI / Anthropic)" as LLM
component "Upstream MCP\ntool servers" as Tools

Agent --> SDK
SDK --> Runtime : start/get/cancel/list\n(HTTP/gRPC)

Runtime --> Kafka : persist step events
Runtime --> PII : model calls
Runtime --> Gateway : tool calls
Runtime --> Store : snapshots, effects ledger

PII --> LLM : redacted prompt
PII --> Store : PII vault (AES-256-GCM)

Gateway --> Tools : forward + per-call JWT
Gateway --> Store : manifests
Gateway --> Kafka : hash-chained audit
Gateway ..> Runtime : approval signals

Drift --> Kafka : read-only sample
Drift --> LLM : explain diff (Py explainer)

Chaos ..> Gateway : fault hook (per tool call)
Chaos ..> PII : fault hook (per model call)
Chaos --> Runtime : drive tasks / adapters

note bottom of Chaos
  Dashed = fault injection, no
  production data path of its own
end note
@enduml
```

## 2. Deployment diagram

Local (`docker compose up`, 16 GB laptop) and the AWS demo environment
(Terraform, ECS Fargate or EKS). Observability via OpenTelemetry + Prometheus.

```plantuml
@startuml platform-deployment
title Keel — deployment (local + AWS demo)

skinparam shadowing false

node "Developer laptop (local)" as Laptop {
  node "Docker Compose" as Compose {
    artifact "runtime" as rt
    artifact "gateway" as gw
    artifact "pii-proxy" as pp
    artifact "drift-guard" as dg
    artifact "harness" as ha
    artifact "Redpanda (Kafka)" as rp
    database "SQLite files" as sq
  }
}

cloud "AWS demo env (Terraform)" as AWS {
  node "ECS Fargate / EKS" as ECS {
    artifact "runtime task" as rt2
    artifact "gateway task" as gw2
    artifact "pii-proxy task" as pp2
    artifact "drift-guard task" as dg2
  }
  queue "Kafka-compatible broker\n(single self-run, demo)" as msk
  database "SQLite -> Postgres\n(on measured trigger, ADR-006)" as pg
  component "Secrets Manager" as sm
  component "Budget alarm\nUS$20/month" as budget
}

node "Observability" as Obs {
  component "OpenTelemetry\n(one trace per run)" as otel
  component "Prometheus\nmetrics" as prom
}

Compose --> Obs
ECS --> msk
ECS --> pg
ECS --> sm
ECS --> Obs
AWS --> budget

note right of pg
  ADR-006: SQLite first.
  Move to Postgres when workers
  span >1 host (AWS kill test) or
  DBOS engine is chosen.
end note
@enduml
```

## 3. Class / domain model — [A] Durable runtime

Event-sourced per run. Steps persist as ordered events keyed by `run_id`; the
effects ledger guards side effects by `hash(run_id, step, tool, args)`.

```plantuml
@startuml class-runtime
title [A] Durable runtime — domain model

skinparam shadowing false
hide empty members

class Run {
  +run_id : UUID
  +status : RunStatus
  +current_step : int
  +created_at : ts
  +snapshot_version : int
}

enum RunStatus {
  RUNNING
  AWAITING_SIGNAL
  COMPLETED
  FAILED
  CANCELLED
}

abstract class RunEvent {
  +run_id : UUID
  +seq : int
  +ts : ts
}
class RunStarted
class StepScheduled
class ModelCallCompleted
class ToolCallRequested
class ToolCallCompleted
class AwaitingSignal
class StepCompleted
class RunCompleted
class RunFailed

class Snapshot {
  +run_id : UUID
  +version : int
  +state_blob : bytes
  +at_seq : int
}

class EffectLedgerEntry {
  +key : hash(run_id,step,tool,args)
  +state : EffectState
  +result : bytes
  +idempotency_key : string
}
enum EffectState {
  INTENT
  DONE
  UNKNOWN
}

class ToolCall {
  +run_id : UUID
  +step : int
  +tool : string
  +args : json
  +idempotency_key : string
}

Run "1" o-- "many" RunEvent : ordered log
Run "1" o-- "many" Snapshot : every 50 ev / 30 s
RunEvent <|-- RunStarted
RunEvent <|-- StepScheduled
RunEvent <|-- ModelCallCompleted
RunEvent <|-- ToolCallRequested
RunEvent <|-- ToolCallCompleted
RunEvent <|-- AwaitingSignal
RunEvent <|-- StepCompleted
RunEvent <|-- RunCompleted
RunEvent <|-- RunFailed
Run "1" -- "1" RunStatus
ToolCall "1" -- "1" EffectLedgerEntry : guarded by
EffectLedgerEntry -- EffectState
@enduml
```

## 4. Class / domain model — [1] MCP gateway

```plantuml
@startuml class-gateway
title [1] MCP gateway — domain model

skinparam shadowing false
hide empty members

class ToolCallRequest {
  +agent_id : string
  +end_user : string
  +tool : string
  +args : json
  +data_labels : string[]
}

class ToolManifest {
  +tool : string
  +definition_hash : string
  +signature : bytes
  +approver : string
  +version : int
}

class Policy {
  +id : string
  +cel_expr : string
  +source : git_ref
}

class PolicyDecision {
  +allow : bool
  +reason : string
  +requires_approval : bool
}

class ScopedToken {
  +jti : UUID
  +audience : string
  +tool : string
  +arg_hash : string
  +ttl : < 60s
  +signed_with : kid
}

class SessionTaint {
  +session_id : string
  +tainted : bool
  +source_tool : string
}

class AuditRecord {
  +seq : int
  +prev_hash : string
  +this_hash : string
  +decision : PolicyDecision
  +ts : ts
}

class JWKS {
  +keys : key[]
  +rotates_every : 24h
}

ToolCallRequest --> ToolManifest : pinned-hash check
ToolCallRequest --> Policy : evaluated by (CEL)
Policy --> PolicyDecision : yields
PolicyDecision --> ScopedToken : mint if allow
ScopedToken --> JWKS : verifiable via
ToolCallRequest --> SessionTaint : updates
AuditRecord --> AuditRecord : prev_hash chain
PolicyDecision --> AuditRecord : recorded in
@enduml
```

## 5. Class / domain model — [2] PII proxy

```plantuml
@startuml class-pii
title [2] DPDP PII proxy — domain model

skinparam shadowing false
hide empty members

class RedactionRequest {
  +tenant_id : string
  +prompt : text
  +purpose : string
}

class DetectedSpan {
  +start : int
  +end : int
  +type : PiiType
  +source : rule|ner
}

enum PiiType {
  AADHAAR
  PAN
  GSTIN
  UPI
  IFSC
  MOBILE
  VOTER_ID
  PASSPORT
  VEHICLE_REG
  NAME
  ADDRESS
}

class Token {
  +value : "<PAN_7f3a>"
  +type : PiiType
  +hmac : string
}

class VaultEntry {
  +tenant_id : string
  +token : string
  +ciphertext : bytes  'AES-256-GCM
  +data_principal : string
}

class ConsentRecord {
  +data_principal : string
  +purpose : string
  +granted : bool
}

class TenantKey {
  +tenant_id : string
  +data_key : wrapped
  +master_key_ref : string
}

RedactionRequest "1" o-- "many" DetectedSpan
DetectedSpan --> PiiType
DetectedSpan --> Token : tokenised to
Token --> VaultEntry : persisted (reversible)
VaultEntry --> TenantKey : encrypted under
RedactionRequest --> ConsentRecord : purpose check
@enduml
```

## 6. Class / domain model — [4] Schema-drift guard

```plantuml
@startuml class-drift
title [4] Schema-drift guard — domain model

skinparam shadowing false
hide empty members

class TopicSample {
  +topic : string
  +window : duration
  +messages_seen : int
}

class FieldStat {
  +field_path : string
  +types_seen : map<type,count>
  +presence : float  '>99.9% = required
}

class InferredSchema {
  +topic : string
  +fields : FieldStat[]
}

class SchemaChange {
  +field_path : string
  +kind : ChangeKind
  +from : type
  +to : type
}
enum ChangeKind {
  BREAKING
  ADDITIVE
}

class ConsumerRef {
  +service : string
  +repo : string
  +file : string
  +line : int
  +field_path : string
}

class DriftAlert {
  +topic : string
  +change : SchemaChange
  +consumers : ConsumerRef[]
  +explanation : text  'LLM-written
}

TopicSample "1" o-- "many" FieldStat
TopicSample --> InferredSchema : inferred (rolling)
InferredSchema --> SchemaChange : diff vs new window
SchemaChange --> ConsumerRef : mapped (tree-sitter index)
SchemaChange --> DriftAlert
ConsumerRef --> DriftAlert
@enduml
```

## 7. Class / domain model — [B] Chaos harness

```plantuml
@startuml class-chaos
title [B] Chaos harness — domain model

skinparam shadowing false
hide empty members

class Scenario {
  +name : string
  +seed : int
  +faults : FaultSpec[]
}

class FaultSpec {
  +type : FaultType
  +target : gateway|llm_proxy
  +trigger : step|probability
}
enum FaultType {
  TOOL_TIMEOUT
  HTTP_429
  HTTP_500
  MALFORMED_JSON
  TRUNCATED_OUTPUT
  SCHEMA_BREAK
  LATENCY_SPIKE
  INJECTED_INSTRUCTION
  CONTEXT_FLOOD
  MODEL_OUTAGE
}

class AgentAdapter {
  +framework : string  'LangGraph|OpenAI SDK|plain|Keel
  +run(task) : RunResult
}

class RunResult {
  +success : bool
  +recovered : bool
  +duplicates : int
  +unsafe_actions : int
  +extra_tokens : int
  +cost : float
}

class Report {
  +scenarios : Scenario[]
  +results : RunResult[]
  +comparison_table : md
}

Scenario "1" o-- "many" FaultSpec
FaultSpec --> FaultType
AgentAdapter --> RunResult : produces
Scenario --> AgentAdapter : driven against
RunResult --> Report : aggregated into
@enduml
```

## 8. Sequence — durable run with crash recovery (FR-A2/A3/A4)

```plantuml
@startuml seq-durable-run
title Durable run: step persistence + crash recovery + effects ledger

skinparam shadowing false
actor Agent
participant SDK
participant "Runtime\n(worker W1)" as W1
participant "Runtime\n(worker W2)" as W2
queue "Kafka\nruns.events" as K
database "Ledger\n(SQLite)" as L
participant "MCP gateway" as GW
participant "Email tool" as Tool

Agent -> SDK : run(task)
SDK -> W1 : start run
W1 -> K : RunStarted, StepScheduled
W1 -> K : ModelCallCompleted (recorded)
W1 -> L : INTENT key=hash(run,step,send_email,args)
W1 -> GW : tool call send_email
GW -> Tool : forward + Idempotency-Key
note over W1 : worker W1 dies before DONE written
... rebalance (< 5 s) ...
K -> W2 : partition reassigned
W2 -> K : replay events (model NOT re-called)
W2 -> L : lookup key
alt DONE exists
  L --> W2 : stored result
else INTENT only, tool supports idempotency
  W2 -> GW : retry with same Idempotency-Key
  GW -> Tool : dedup -> no duplicate send
  W2 -> L : DONE
else INTENT only, no idempotency support
  W2 -> L : mark UNKNOWN
  W2 -> K : AwaitingSignal (compensation/approval)
end
W2 -> K : StepCompleted, RunCompleted
W2 --> SDK : result
@enduml
```

## 9. Sequence — zero-trust tool call (FR-G2..G8)

```plantuml
@startuml seq-gateway-call
title MCP gateway: per-call policy, scoped token, taint, audit

skinparam shadowing false
participant "Runtime" as RT
participant "Gateway" as GW
participant "Policy (CEL)" as POL
participant "Manifest store" as MAN
participant "JWKS signer" as JW
participant "Upstream tool" as UP
queue "Kafka audit" as AUD

RT -> GW : tool call (agent, user, tool, args, labels)
GW -> MAN : check definition hash
alt hash changed
  MAN --> GW : mismatch
  GW -> AUD : audit DENY (re-approval needed)
  GW --> RT : blocked (show diff)
else hash pinned
  GW -> POL : evaluate per-call policy
  alt deny
    POL --> GW : deny + reason
    GW -> AUD : audit DENY
    GW --> RT : denied (fail closed)
  else allow, needs approval (tainted + high-risk)
    POL --> GW : allow, requires_approval
    GW --> RT : approval signal
    RT --> GW : approved
    GW -> JW : mint scoped JWT (ttl<60s, arg_hash)
    GW -> UP : forward + token
    UP --> GW : response
    GW -> GW : scan output, update taint
    GW -> AUD : hash-chained record
    GW --> RT : result
  else allow
    POL --> GW : allow
    GW -> JW : mint scoped JWT
    GW -> UP : forward + token
    UP --> GW : response
    GW -> GW : scan output, update taint
    GW -> AUD : hash-chained record
    GW --> RT : result
  end
end
@enduml
```

## 10. Sequence — PII redaction round trip (FR-X1/X3/X5)

```plantuml
@startuml seq-pii
title PII proxy: redact -> LLM -> restore (streaming)

skinparam shadowing false
participant "Caller\n(OpenAI SDK)" as C
participant "PII proxy" as P
participant "Rule detectors\n(regex+checksum)" as R
participant "NER model\n(Indic)" as N
database "Vault\n(AES-256-GCM)" as V
participant "Consent store" as CS
participant "External LLM" as LLM

C -> P : prompt (base URL swapped)
P -> CS : purpose/consent check
alt no matching consent
  CS --> P : none
  P --> C : refused (reason)
else consent ok
  P -> P : normalise (Unicode, digit forms)
  P -> R : detect structured IDs
  P -> N : detect names/addresses
  P -> P : merge overlapping spans
  loop each span
    P -> V : HMAC token; store ciphertext (reversible)
  end
  P -> LLM : redacted prompt (<PAN_7f3a> ...)
  LLM --> P : streamed response (tokens)
  loop each chunk
    P -> V : resolve tokens (buffer split boundaries)
    P --> C : restored chunk
  end
end
@enduml
```

## 11. Sequence — schema-drift detection (FR-D2..D5)

```plantuml
@startuml seq-drift
title Schema-drift guard: detect, map, explain

skinparam shadowing false
queue "Kafka topic\n(e.g. runs.events)" as K
participant "Sampler (Go)\nread-only group" as S
participant "Schema inferencer" as I
participant "Code index\n(tree-sitter)" as CI
participant "Explainer (Py)" as E
participant "LLM" as LLM
actor "On-call" as OC

K -> S : sample messages (<1% throughput)
S -> I : field stats over rolling window
I -> I : infer schema (presence >99.9% = required)
I -> I : diff vs previous window
alt breaking change
  I -> CI : lookup field_path across consumer repos
  CI --> I : services + file:line
  I -> E : diff + schemas + matched snippets
  E -> LLM : write explanation (deterministic match, LLM only explains)
  LLM --> E : alert text
  E -> OC : alert (< 60 s) with services, locations, why
else additive
  I -> I : record, no alert
end
@enduml
```

## 12. Sequence — chaos injection run (FR-C1..C5)

```plantuml
@startuml seq-chaos
title Chaos harness: seeded fault injection + scoring

skinparam shadowing false
participant "Harness" as H
participant "AgentAdapter" as A
participant "Gateway\nfault hook" as GW
participant "LLM proxy\nfault hook" as LP
participant "Mock services\n(count side effects)" as MS

H -> H : load scenario (seed, faults)
H -> GW : arm tool-call faults
H -> LP : arm model-call faults
H -> A : run task suite
loop each step
  A -> GW : tool call
  alt fault armed
    GW --> A : timeout / 429 / malformed / injected
  else clean
    GW -> MS : execute (counted)
    MS --> GW : ok
    GW --> A : result
  end
  A -> LP : model call
  LP --> A : fault or completion
end
A --> H : RunResult (success, recovery, duplicates, unsafe, cost)
H -> H : aggregate -> markdown report + comparison table
@enduml
```

## 13. State machine — run lifecycle (FR-A1/A3/A5)

```plantuml
@startuml state-run
title Run lifecycle

skinparam shadowing false
[*] --> Running : RunStarted
Running --> Running : StepCompleted
Running --> AwaitingSignal : AwaitingSignal\n(approval / HITL)
AwaitingSignal --> Running : signal (approve/deny/resume)
Running --> Completed : RunCompleted
Running --> Failed : RunFailed
Running --> Cancelled : cancel command
AwaitingSignal --> Cancelled : cancel command
Running --> Running : worker death\n(resume < 5 s, replay log)
Completed --> [*]
Failed --> [*]
Cancelled --> [*]
@enduml
```

## 14. State machine — effects ledger entry (FR-A4)

```plantuml
@startuml state-effect
title Effects ledger entry — exactly-once intent

skinparam shadowing false
[*] --> Intent : write INTENT before tool call
Intent --> Done : side effect confirmed
Intent --> Unknown : crash mid-call,\nno idempotency support
Intent --> Done : retry w/ Idempotency-Key\n(upstream dedup)
Unknown --> Done : compensation / manual approval
Unknown --> Failed : give up
Done --> [*] : stored result returned on retry
Failed --> [*]
@enduml
```

## 15. State machine — session taint (FR-G5)

```plantuml
@startuml state-taint
title Gateway session taint

skinparam shadowing false
[*] --> Clean
Clean --> Tainted : untrusted tool output read
Clean --> Clean : trusted output
Tainted --> Tainted : further reads
state "High-risk tool requested" as HR
Tainted --> HR : e.g. send_email
HR --> Approved : approval signal granted
HR --> Blocked : approval denied
Approved --> Tainted : call proceeds
Blocked --> Tainted : call refused
Clean --> LowRiskAllowed : high-risk w/o taint\nallowed directly
@enduml
```

## Traceability

| Diagram | Primary requirements |
| --- | --- |
| 1 Platform components | FR-P1, FR-P2 |
| 2 Deployment | FR-P1, FR-P3, NFR-P3, NFR-P6, ADR-006 |
| 3 Runtime class model | FR-A2, FR-A4 |
| 4 Gateway class model | FR-G2..G7, ADR-004, ADR-005 |
| 5 PII class model | FR-X3, FR-X5, NFR-X3, ADR-006 |
| 6 Drift class model | FR-D2..D5 |
| 7 Chaos class model | FR-C1..C5 |
| 8 Durable-run sequence | FR-A2, FR-A3, FR-A4, NFR-A1 |
| 9 Gateway-call sequence | FR-G2, FR-G3, FR-G4, FR-G5, FR-G7, FR-G8, NFR-G2 |
| 10 PII sequence | FR-X1, FR-X3, FR-X5, FR-X6 |
| 11 Drift sequence | FR-D2, FR-D3, FR-D4, FR-D5, NFR-D1 |
| 12 Chaos sequence | FR-C1, FR-C2, FR-C4, FR-C5 |
| 13 Run lifecycle | FR-A1, FR-A3, FR-A5 |
| 14 Effects ledger | FR-A4 |
| 15 Session taint | FR-G5 |
