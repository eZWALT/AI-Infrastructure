<div align="center">
  <img src="assets/hero.svg" alt="AI Infrastructure learning library and platform build" width="100%">

  <h1>AI Infrastructure</h1>

  <p>
    <strong>A growing library of the resources I use to learn AI infrastructure,<br>
    plus one flagship project where I turn that knowledge into a working platform.</strong>
  </p>

  <p>
    <a href="#learning-library"><strong>Learning library</strong></a>
    ·
    <a href="world-model-platform/"><strong>World Model Platform</strong></a>
    ·
    <a href="#current-progress"><strong>Progress</strong></a>
    ·
    <a href="ROADMAP.md"><strong>Roadmap</strong></a>
  </p>

  <p>
    <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-7c3aed.svg?style=flat-square" alt="Apache 2.0 license"></a>
    <img src="https://img.shields.io/badge/resources-4-06b6d4.svg?style=flat-square" alt="4 resources">
    <img src="https://img.shields.io/badge/flagship_projects-1-6366f1.svg?style=flat-square" alt="1 flagship project">
    <img src="https://img.shields.io/badge/status-growing-c084fc.svg?style=flat-square" alt="Growing repository">
  </p>
</div>

## About

Each top-level learning folder represents one source I am studying, such as a course, handbook, repository, or focused reading list. Sources remain independent and keep their own attribution, notes, and exercises.

The only large build is [`world-model-platform/`](world-model-platform/). It applies lessons from the library to a small Kubernetes-native AI platform.

## Repository structure

```text
AI-Infrastructure/
├── zero-hero-course/                 # selected introductory labs
├── stanford-cs336-.../               # language modeling from first principles
├── camilo-inference-recommendations/ # curated inference study path
├── world-model-platform/             # the flagship platform project
├── assets/                           # diagrams and artwork
├── ROADMAP.md
├── CONTRIBUTING.md
├── SECURITY.md
└── LICENSE
```

The structure follows three rules:

- one source per top-level learning folder;
- no empty folders for resources that are not being studied yet;
- one major implementation project: the World Model Platform.

## Learning library

### 01. Zero-to-Hero AI Infrastructure

An external introductory course used to establish basic vocabulary and hands-on exposure to cloud, containers, Kubernetes, GPUs, training, serving, observability, MLOps, and security.

This is a selected subset, not a completion challenge. Roughly 36 useful labs were retained, while repetitive or low-value sessions were skipped.

[Browse the selected labs](zero-hero-course/)

### 02. Stanford CS336: Language Modeling from Scratch

A fundamental systems course covering architecture, resource accounting, GPU optimization, distributed training, scaling, data, evaluation, inference, and post-training.

Current focus: Lecture 3, Architectures and Hyperparameters.

[Open the CS336 study folder](stanford-cs336-language-modeling-from-scratch/)

### 03. Camilo's Inference Optimization Resources

A practical study path curated by Camilo. It moves from inference arithmetic and memory fundamentals to vLLM benchmarking, optimization techniques, GPU kernels, engines, and research papers.

[Open the inference optimization resources](camilo-inference-recommendations/)

## Flagship project

### World Model Platform

A compact Kubernetes-native platform for the full model lifecycle:

```text
collect -> validate -> train -> evaluate -> register -> serve -> observe -> improve
```

The project will cover reproducible infrastructure, data pipelines, GPU training, experiment tracking, model serving, observability, security, cost controls, architecture decisions, and runbooks.

<div align="center">
  <img src="assets/platform-blueprint.svg" alt="World Model Platform target architecture" width="96%">
</div>

This is a target architecture. The implementation is still in its foundation phase.

[Explore the World Model Platform](world-model-platform/)

## Current progress

- **Learning sources:** 3
- **Zero-to-Hero:** roughly 36 selected introductory labs
- **Stanford CS336:** Lecture 3 study plan added
- **Inference optimization:** Camilo's curated path added
- **World Model Platform:** scope, architecture, and milestones defined; implementation not started

See [ROADMAP.md](ROADMAP.md) for active and upcoming work.

## Use this repository

```bash
git clone https://github.com/eZWALT/AI-Infrastructure.git
cd AI-Infrastructure
```

Open any learning folder independently, or follow the World Model Platform to see the concepts combined in one system.

Requirements differ by source. Each folder should document its tools, hardware, cloud access, expected cost, verification, and cleanup.

## Attribution and licensing

Some folders contain notes or adaptations based on third-party material. Their original authorship and licensing must be documented in the relevant folder.

The repository-level [Apache License 2.0](LICENSE) applies to my original contributions unless a file or source folder states otherwise. It does not override third-party licenses.

Corrections and attribution fixes are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md). Security-sensitive reports should follow [SECURITY.md](SECURITY.md).

<div align="center">
  <sub>Study strong sources. Keep the useful parts. Build one real system.</sub>
</div>
