# JAX Scaling Book

A student-side study folder for *How to Scale Your Model*, the Google DeepMind / JAX systems book on scaling Transformers across TPUs and GPUs.

**Authors:** Jacob Austin, Sholto Douglas, Roy Frostig, Anselm Levskaya, Charlie Chen, Sharad Vikram, Federico Lebron, Peter Choy, Vinay Ramasesh, Albert Webson, Reiner Pope<br>
**Publisher:** Google DeepMind<br>
**Canonical site:** [jax-ml.github.io/scaling-book](https://jax-ml.github.io/scaling-book/)<br>
**Source repository:** [github.com/jax-ml/scaling-book](https://github.com/jax-ml/scaling-book)<br>
**Local PDF:** [`references/How-To-Scale-Your-Model.pdf`](references/How-To-Scale-Your-Model.pdf)<br>
**Role in this repository:** quantitative systems foundations for distributed training and inference<br>
**Status:** queued for study

[Back to the learning library](../README.md)

## Why this book belongs here

This book answers the questions that sit between model architecture and cluster reality:

- how close a workload is to its compute, memory, or communication roofline;
- how TPUs and GPUs move data and why that bounds strong scaling;
- how to count Transformer parameters, FLOPs, and KV-cache cost;
- how to choose data, tensor, pipeline, and expert parallelism;
- how training and inference change the memory and latency landscape;
- how to profile and debug real JAX/XLA programs.

It pairs well with Stanford CS336, Camilo's inference path, and the World Model Platform. CS336 explains how to build the model; this book explains how expensive it should be to train and serve at scale.

## What this folder is

This is **not** a clone or mirror of the upstream book. Keep notes and exercises here; read the book from the local PDF or official site.

```text
jax-scaling-book/
├── README.md
├── references/                 # PDF + citation
├── 00-introduction/            # notes only
├── 01-rooflines/               # notes + exercises
├── ...
└── 12-gpus/
```

## Chapters

| # | Chapter | Mode |
|---|---|---|
| 0 | [Introduction](00-introduction/) | notes |
| 1 | [Roofline Analysis](01-rooflines/) | notes + exercises |
| 2 | [How to Think About TPUs](02-tpus/) | notes + exercises |
| 3 | [Sharded Matrices](03-sharding/) | notes + exercises |
| 4 | [Transformer Math](04-transformers/) | notes + exercises |
| 5 | [Training Parallelism](05-training/) | notes + exercises |
| 6 | [Training LLaMA 3](06-applied-training/) | notes + exercises |
| 7 | [Transformer Inference](07-inference/) | notes + exercises |
| 8 | [Serving LLaMA 3](08-applied-inference/) | notes + exercises |
| 9 | [Profiling](09-profiling/) | notes + exercises |
| 10 | [Programming TPUs in JAX](10-jax-programming/) | notes + exercises |
| 11 | [Conclusions](11-conclusion/) | notes |
| 12 | [How to Think About GPUs](12-gpus/) | notes + exercises |

Intro and conclusion are notes-only. Every other chapter has `exercises.md` beside its `README.md`.

## Suggested study path

1. Read Chapters 0 to 3 and start a personal roofline calculator.
2. Work through Chapter 4 until parameter, FLOP, and KV-cache estimates are automatic.
3. Study Chapter 5 and choose a parallelism plan for a small training budget.
4. Study Chapter 7 and estimate serving latency, batch size, and KV-cache memory.
5. Use Chapter 12 when translating TPU intuition to NVIDIA GPUs.
6. Apply the estimates to one World Model Platform training or serving design decision.

## Attribution

*How to Scale Your Model* belongs to its authors and Google DeepMind. The upstream book is released under the MIT License. The local PDF is a community LaTeX build from [NTT123/scaling_book.tex](https://github.com/NTT123/scaling_book.tex); prefer the live site for the newest corrections.

This folder is an independent study guide and is not affiliated with or endorsed by Google DeepMind or the JAX project.
