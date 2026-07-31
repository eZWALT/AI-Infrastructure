# World Model Platform

> A small, Kubernetes-native AI platform built in public to turn infrastructure concepts into an end-to-end working system.

**Status:** 🌱 architecture and foundation phase

[← Back to the repository](../README.md) · [Roadmap](../ROADMAP.md) · [Contributing](../CONTRIBUTING.md)

## Mission

Build the smallest platform that demonstrates the real lifecycle of a world model:

```text
collect → validate → train → evaluate → register → serve → observe → improve
```

The project is intentionally compact enough to run locally where possible, but structured around patterns that can grow into a production platform.

## Target capabilities

- **Platform foundation** — local Kubernetes, declarative bootstrap, namespaces, ingress, secrets
- **Data plane** — ingestion, validation, versioning, object storage, lineage
- **Training plane** — repeatable jobs, GPU scheduling, experiment tracking, checkpoints
- **Model plane** — evaluation gates, registry, artifact promotion
- **Serving plane** — inference API, autoscaling, rollout and rollback
- **Operations plane** — metrics, logs, traces, alerts, SLOs and runbooks
- **Governance plane** — identity, policy, supply-chain security and cost visibility

## Design constraints

1. A useful local path must exist before adding a cloud path.
2. Every component must have a documented reason to exist.
3. Infrastructure changes must be declarative and reviewable.
4. Each milestone must include verification and teardown.
5. Observability, security, and cost must evolve with the platform.

## Planned layout

```text
world-model-platform/
├── bootstrap/        # cluster creation and local prerequisites
├── platform/         # shared platform services
├── workloads/        # training, evaluation, and serving workloads
├── observability/    # dashboards, alerts, and SLOs
├── tests/            # smoke, integration, and resilience checks
└── docs/
    ├── adr/          # architecture decision records
    ├── runbooks/     # operational procedures
    └── milestones/   # build notes and evidence
```

Directories will be introduced only when the corresponding milestone starts; this page is the contract for the intended shape.

## Milestones

- [ ] **M0 — Bootstrap:** create and destroy a reproducible local cluster
- [ ] **M1 — Platform:** install ingress, storage, secrets, and GitOps
- [ ] **M2 — Observe:** collect platform metrics, logs, and traces
- [ ] **M3 — Data:** ingest and version a small multimodal dataset
- [ ] **M4 — Train:** run a tracked, checkpointed training workload
- [ ] **M5 — Register:** evaluate and promote a model artifact
- [ ] **M6 — Serve:** deploy an observable inference endpoint
- [ ] **M7 — Operate:** define SLOs, alerts, rollback, and recovery
- [ ] **M8 — Harden:** add policy, image provenance, and cost controls

## Definition of done

A milestone is done when another person can:

- understand the design decision and its trade-offs;
- reproduce the result from a clean environment;
- verify expected behavior with an explicit check;
- observe at least one representative failure mode;
- remove created resources without guesswork.

## First build

The implementation has not started yet. The first change will define the local development contract and cluster bootstrap. Until then, the root **[roadmap](../ROADMAP.md)** is the source of truth.
