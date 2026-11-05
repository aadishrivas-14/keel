# Ways of working

Keel is built with limited resources, so these rules keep scope small, decisions written down, and every number honest and reproducible by users.

## Principles

1. **Docs before code.** A component starts only when it meets the Definition of Ready (ADR-009).
2. **Measure, then claim.** No public, marketing or README claim is made until a proof test produced the number.
3. **Finish before starting.** One component in progress at a time; the next starts when the current one is done.
4. **Small and shippable.** Each week ends with something merged: a doc section, a test, or code.
5. **Write decisions down.** Anything that affects more than one component, or that is hard to reverse, gets an ADR.

## Weekly cadence

| When | Activity | Time | Output |
| --- | --- | --- | --- |
| Monday | Plan the week: pick 2 or 3 issues from the board | 20 min | Week's issues moved to In progress |
| Weekdays | Focused work blocks | 3 x 2 h | Merged PRs or doc updates |
| Weekend | Longer block for harder work | 3 to 4 h | Merged PRs |
| Sunday | Review: what shipped, what slipped, update the board and the docs | 30 min | Short entry in `CHANGELOG.md` |
| Each milestone | Run proof tests, update READMEs, write a short post | 1 day | Published numbers and a post |

## Definition of Ready (before any code for a component)

- [ ] Requirements written in requirements.md, each with priority and acceptance criteria.
- [ ] Design section in design-overview.md updated: data model, APIs, request flow.
- [ ] Every ADR that blocks this component is Accepted.
- [ ] Test cases listed with `T-` IDs; proof test scripted in words.
- [ ] Prior art checked for this component in the last 30 days.
- [ ] Work broken into issues of at most one week each.

## Definition of Done (for an issue)

- [ ] Code merged to main through a PR, CI green.
- [ ] Tests for the change, linked to requirement IDs.
- [ ] Docs updated in the same PR (README, API docs, design docs if behaviour changed).
- [ ] No new security findings; no secrets; no PII in logs.

## Definition of Done (for a component milestone)

- [ ] All exit criteria in test-plan.md met.
- [ ] Proof-test numbers published with date and hardware.
- [ ] Demo runs from a clean clone in under 5 commands.
- [ ] Landing page and public changelog updated with the new capability and its measured number.

## Engineering practices

**Repo layout (ADR-007).** See the README. Each component folder is self-contained and shares only `contracts/`.

**Branching and commits**

- Trunk-based: short branches off `main`, merged within 2 or 3 days.
- Conventional commit messages (`feat(gateway): ...`, `test(runtime): ...`), with the requirement ID in the body when one applies.
- Every change goes through a PR, even working alone. The PR template asks: which requirement, which test, what changed in docs.

**CI (GitHub Actions)**

- On every PR: lint, unit, contract, integration, secret scan, dependency scan.
- On merge to main: end to end with the mock model.
- Nightly: fault tests. Proof tests run manually at milestones.
- Main stays green; a red main is fixed before anything else.

**Tracking**

- GitHub Projects board: Backlog, Ready, In progress, Review, Done. An issue moves to Ready only if it meets the Definition of Ready.
- Labels per component (`comp:gateway`) and per type (`req`, `test`, `docs`, `adr`).

**Using AI coding agents**

- Allowed and encouraged; it lets a small team ship a broad platform faster.
- Write the test or acceptance criterion first, then let the agent implement.
- Every generated change is read line by line before merge; no unreviewed code on main.
- Agents get repo context only. No customer data, credentials or secrets in prompts.

## How the docs change

- The Claude doc is the source of truth until code exists; then ADRs and design live in `docs/` in the repo, and the doc links there.
- Requirement changes: edit the row, keep the ID, add a dated note under that component's table if the meaning changed. Never reuse an ID.
- Decisions: a new ADR, never an edit to an accepted one.
- Every Sunday review checks that the docs still match reality.

## Boundaries

- **IP cleanliness.** The codebase and docs contain only original or properly licensed work. No third-party proprietary code, designs, data or internal names, even as examples.
- **Scope.** No telecom, eSIM or entitlement features; the product stays focused on the five components.
- **Data.** Synthetic data only (test-plan.md). No real personal data in tests, demos, screenshots or prompts.
- **Claims.** Public and marketing claims quote proof-test numbers only, with the conditions they were measured under, so any user can reproduce them.

## Publishing

- The repo goes public when component A meets its Definition of Done, not before.
- Each milestone gets a short launch post (blog or developer channels) with the number, the setup, and one honest limitation.
- The chaos-harness comparison is published as a standalone write-up; it doubles as marketing and lead generation.
- After each milestone, update the public changelog and landing page so prospective users see current capabilities and benchmarks.
