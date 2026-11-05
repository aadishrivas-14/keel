# ADR-002: Kafka-compatible backbone, Redpanda locally

- **Status:** Accepted
- **Date:** 2026-11-05

## Context

Handoff, audit and drift detection all need an ordered, replayable log, and Kafka is a core strength.

## Decision

Use the Kafka protocol everywhere. Run Redpanda (single binary) for local dev and CI; on AWS, run a single broker on EC2 or Fargate for demos.

## Consequences

- No managed Kafka bill.
- Code must use only Kafka-protocol features that Redpanda also supports, including transactions.
