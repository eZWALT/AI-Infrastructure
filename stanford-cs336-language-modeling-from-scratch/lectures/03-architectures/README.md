# Lecture 3 — Architectures and Hyperparameters

**Course:** [Stanford CS336: Language Modeling from Scratch](https://cs336.stanford.edu/)<br>
**Offering:** Spring 2026<br>
**Scheduled lecture:** April 6, 2026<br>
**Lecturer:** Tatsu Hashimoto<br>
**Status:** queued for study

[← Course overview](../../README.md)

## Learning goals

After studying the lecture and reproducing the relevant calculations, I want to be able to:

- explain the major architectural decisions inside a modern Transformer language model;
- reason about how model width, depth, head configuration, and context length interact;
- calculate how architecture choices affect parameter count, FLOPs, activation memory, and KV-cache memory;
- distinguish parameters that change model capacity from parameters that primarily change execution cost;
- connect hyperparameter choices to training stability and scaling behavior;
- justify a small model configuration under an explicit compute and memory budget.

## Questions to answer

### Architecture

- Which components dominate the parameter count?
- How should depth and width be balanced?
- What are the practical roles of attention heads and head dimension?
- Which normalization, activation, and positional-encoding choices matter?
- Where do architectural choices affect training versus inference differently?

### Hyperparameters

- Which hyperparameters are structural, optimization-related, or data-related?
- How should learning rate, batch size, sequence length, and token budget co-evolve?
- Which ratios tend to remain stable as models scale?
- What measurements should drive a configuration decision?

### Infrastructure consequences

- What consumes GPU memory during training?
- What consumes memory during autoregressive inference?
- Which dimensions improve hardware utilization?
- Where do tensor, pipeline, data, or sequence parallelism become necessary?
- Which architecture decisions constrain serving later?

## Study workflow

- [ ] Watch the official lecture recording
- [ ] Read the official Lecture 3 slides
- [ ] Write the architecture summary in my own words
- [ ] Derive parameter-count equations
- [ ] Derive approximate training FLOPs
- [ ] Estimate training and inference memory for one small configuration
- [ ] Implement a configuration calculator
- [ ] Compare at least two architecture variants under the same budget
- [ ] Record unresolved questions and infrastructure implications

## Notes

> Add verified notes here while studying. Keep direct course claims linked to the official material and clearly label personal interpretations.

### Core architecture

_Pending._

### Hyperparameter relationships

_Pending._

### Compute and memory model

_Pending._

### Implications for the World Model Platform

_Pending._

## Experiments

Planned:

1. Build a small parameter and FLOP calculator.
2. Compare deep/narrow and shallow/wide configurations.
3. Estimate KV-cache growth across context lengths.
4. Measure whether theoretical resource estimates match a small PyTorch model.

## References

- [Official CS336 course website](https://cs336.stanford.edu/)
- Official Lecture 3 slides and recording are linked from the course schedule.
