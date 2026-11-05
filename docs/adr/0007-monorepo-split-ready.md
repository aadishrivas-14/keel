# ADR-007: Monorepo now, designed to split later

- **Status:** Accepted
- **Date:** 2026-11-05

## Context

A monorepo is easiest for a small team to develop and release, but the gateway or the chaos harness may later deserve its own repo, community and contributors.

## Decision

One monorepo, with rules that keep each component splittable at any time.

### Split-ready rules

1. Each component folder is self-contained: its own `go.mod` or `pyproject.toml`, Dockerfile, README, CHANGELOG and tests.
2. Components never import each other's code. They share only `contracts/` (JSON Schemas plus generated Go and Python packages). A CI lint fails on any cross-component import.
3. `contracts/` is versioned, and components depend on a version, not the folder. After a split, the published package replaces the folder with no code change.
4. Each component has its own CI workflow, filtered by path, and its own release tags (`gateway/v0.1.0`).
5. Integration between components happens only over the network (HTTP, MCP, Kafka), as in production.
6. Shared tooling (lint config, Compose files) lives in `deploy/` and `tooling/`, which a split repo copies.

### How a split happens

1. Publish `contracts` as a Go module and a PyPI package at the current version.
2. Extract the folder with its history: `git filter-repo --subdirectory-filter <component>`.
3. Copy shared tooling, switch the dependency from folder to package, and run the component's CI in the new repo.
4. In the monorepo, replace the folder with a pointer in the README and pin the new repo's release in `deploy/`.

### When to split

A component gets outside users or contributors, needs a release cadence of its own, or is being pitched on its own. Likely order: gateway first, then chaos harness.

## Consequences

Slightly more ceremony now (versioned contracts, path-filtered CI) in exchange for a split that takes about a day, not a rewrite.
