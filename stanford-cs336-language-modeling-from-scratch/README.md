# Stanford CS336 — Language Modeling from Scratch

> A foundational source for understanding how language models are designed, trained, optimized, scaled, evaluated, and deployed from first principles.

**Offering:** Spring 2026<br>
**Institution:** Stanford University<br>
**Canonical source:** [cs336.stanford.edu](https://cs336.stanford.edu/)<br>
**Role in this repository:** fundamental learning source<br>
**Status:** active study

[← Back to the learning library](../README.md)

## Why this course matters

CS336 approaches language models like a systems course: build the major pieces rather than treating the stack as a black box.

It connects directly to the infrastructure topics this repository is meant to explore:

- transformer implementation and architecture choices;
- FLOPs, memory, arithmetic intensity, and resource accounting;
- GPU kernels, Triton, and FlashAttention;
- memory-efficient and distributed training;
- scaling laws and experimental design;
- data collection, filtering, deduplication, and mixing;
- evaluation, inference, post-training, and alignment.

The course will provide much of the technical foundation used later in the [`world-model-platform`](../world-model-platform/) project.

## Course scope

The Spring 2026 offering is organized around five implementation-heavy assignments:

1. **Basics** — tokenizer, Transformer architecture, optimizer, and minimal LM training
2. **Systems** — profiling, benchmarking, Triton kernels, FlashAttention, and distributed training
3. **Scaling** — Transformer components, scaling laws, and training experiments
4. **Data** — Common Crawl processing, filtering, deduplication, and data quality
5. **Alignment and reasoning RL** — supervised fine-tuning, reinforcement learning, reasoning, and optional safety alignment

The official course expects strong Python and software-engineering skills, familiarity with PyTorch, deep learning, systems optimization, and the GPU memory hierarchy.

## Notes structure

```text
stanford-cs336-language-modeling-from-scratch/
├── README.md
└── lectures/
    └── 03-architectures/
        └── README.md
```

Additional lecture or assignment folders will be created only when I actively study them.

Each note should distinguish:

- facts and diagrams from the official course;
- my own explanations and mental models;
- implementation experiments;
- open questions and follow-up reading.

## Active topic

### Lecture 3 — Architectures and hyperparameters

The current study entry focuses on how architecture and hyperparameter decisions affect model quality, compute, memory, and scaling behavior.

→ **[Open the Lecture 3 study notes](lectures/03-architectures/)**

## Official resources

- [Course website and schedule](https://cs336.stanford.edu/)
- [Assignment 1 — Basics](https://github.com/stanford-cs336/assignment1-basics)
- [Assignment 2 — Systems](https://github.com/stanford-cs336/assignment2-systems)
- [Assignment 3 — Scaling](https://github.com/stanford-cs336/assignment3-scaling)
- [Assignment 4 — Data](https://github.com/stanford-cs336/assignment4-data)

## Attribution and academic integrity

CS336 and its course materials belong to Stanford University and the course staff. This folder is an independent personal study guide and is not affiliated with or endorsed by Stanford.

Assignment repositories currently publish their code under the MIT License. Other materials, including lecture slides and recordings, may have different terms. Link to official materials rather than copying them unless redistribution is explicitly permitted.

The official course applies an honor code and restricts using AI tools to directly solve assignments. Any implementation work recorded here should respect those rules and remain my own.
