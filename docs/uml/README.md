# Keel UML diagrams

Rendered UML diagrams for the Keel platform, derived from the design docs
([../design-overview.md](../design-overview.md), [../requirements.md](../requirements.md),
[../adr/](../adr/)) as of the 2026-11-05 snapshot.

The repo is documentation-only, so these model the *intended* design — specified
behaviour and contracts, not existing code.

## Folder layout

```
uml/
  src/   PlantUML source (.puml) — edit these
  png/   rendered PNG images
  svg/   rendered SVG images
```

## Diagrams

| Diagram | Kind | Files | Requirements |
| --- | --- | --- | --- |
| Platform components | component | `platform-components` | FR-P1, FR-P2 |
| Deployment (local + AWS) | deployment | `platform-deployment` | FR-P1, FR-P3, NFR-P3/6, ADR-006 |
| [A] Durable runtime | class | `class-runtime` | FR-A2, FR-A4 |
| [1] MCP gateway | class | `class-gateway` | FR-G2..G7, ADR-004/005 |
| [2] PII proxy | class | `class-pii` | FR-X3, FR-X5, NFR-X3 |
| [4] Schema-drift guard | class | `class-drift` | FR-D2..D5 |
| [B] Chaos harness | class | `class-chaos` | FR-C1..C5 |
| Durable run + recovery | sequence | `seq-durable-run` | FR-A2/A3/A4, NFR-A1 |
| Zero-trust tool call | sequence | `seq-gateway-call` | FR-G2..G8, NFR-G2 |
| PII redaction round trip | sequence | `seq-pii` | FR-X1/X3/X5/X6 |
| Schema-drift detection | sequence | `seq-drift` | FR-D2..D5, NFR-D1 |
| Chaos injection run | sequence | `seq-chaos` | FR-C1/C2/C4/C5 |
| Run lifecycle | state | `state-run` | FR-A1/A3/A5 |
| Effects ledger entry | state | `state-effect` | FR-A4 |
| Gateway session taint | state | `state-taint` | FR-G5 |

## Regenerating

After editing any `src/*.puml`:

```sh
cd docs/uml
plantuml -tpng -o "$(pwd)/png" src/*.puml
plantuml -tsvg -o "$(pwd)/svg" src/*.puml
```

Validate syntax without rendering:

```sh
plantuml -checkonly src/*.puml
```
