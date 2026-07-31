# Roadmap

This roadmap covers two distinct efforts:

1. growing a source-by-source AI infrastructure learning library;
2. building one flagship Kubernetes-native World Model Platform.

It describes direction, not a fixed delivery schedule.

[← Back to the repository](README.md)

## North star

Study high-signal material from strong external sources, preserve clear provenance and practical notes, and apply the best ideas to one reproducible end-to-end platform.

## Now

### Learning library

- [x] Preserve a selected introductory AI infrastructure course
- [x] Add Stanford CS336 as a foundational learning source
- [x] Add Camilo's curated inference optimization study path
- [ ] Add original source, author, and license metadata to that course
- [ ] Define a small provenance template for future learning folders
- [ ] Complete Stanford CS336 Lecture 3 architecture notes and experiments
- [ ] Select the next deep technical source after the active CS336 work

### World Model Platform

- [x] Define mission, scope, constraints, and milestones
- [x] Publish the target architecture
- [ ] Record the first architecture decisions
- [ ] Select and document the local cluster toolchain
- [ ] Bootstrap and tear down a clean local cluster

## Next

### Expand the learning library

Potential source tracks, added only when active:

- inference optimization and high-performance serving;
- Hugging Face distributed-training handbooks;
- MIT LLM systems and engineering courses;
- GPU programming and distributed systems;
- production Kubernetes and AI platform engineering.

Each new folder should identify the source, authorship, original license, scope, and the distinction between imported material and personal work.

### Build the platform foundation

- [ ] Add declarative cluster and platform configuration
- [ ] Install ingress, storage, secrets management, and GitOps
- [ ] Add metrics, logs, traces, and a first platform dashboard
- [ ] Define smoke tests and a fast developer feedback loop

## Later

### Run a complete model lifecycle

- [ ] Ingest and version a small multimodal dataset
- [ ] Run a tracked training job with checkpointing
- [ ] Evaluate and register a model artifact
- [ ] Deploy an observable inference endpoint
- [ ] Exercise rollout, rollback, and recovery

### Mature the platform

- [ ] Add GPU-aware scheduling and utilization dashboards
- [ ] Add workload identity, policy enforcement, and image provenance
- [ ] Add resource quotas, cost attribution, and capacity experiments
- [ ] Add resilience testing and incident runbooks
- [ ] Document a cloud deployment path and compare trade-offs

## Success criteria

The repository is succeeding when:

- every learning folder clearly identifies its source, author, and license;
- unrelated resources remain independently navigable;
- notes distinguish original work from adapted or imported material;
- practical exercises include verification, failure notes, and cleanup;
- the World Model Platform can be reproduced from a clean environment;
- platform decisions explain context, alternatives, and consequences;
- the library improves the flagship build, and the build exposes new learning questions.

## Scope boundaries

This repository does not aim to:

- turn every resource into one artificial linear curriculum;
- complete every session in an external course;
- duplicate low-signal or repetitive material for a progress counter;
- build multiple large portfolio projects;
- provide a turnkey production platform for every organization;
- require expensive infrastructure to understand core concepts.
