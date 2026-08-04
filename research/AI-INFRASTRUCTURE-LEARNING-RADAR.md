# AI Infrastructure Learning Radar

Research snapshot from July 31, 2026.

This report extends the current learning library with resources selected for:

- first-principles systems knowledge;
- implementation-heavy exercises;
- measurable performance work;
- production failure analysis;
- direct relevance to the World Model Platform.

Camilo's [inference optimization recommendations](../camilo-inference-recommendations/) were used as an exclusion baseline. Resources already present there, including the Hugging Face Ultra-Scale Playbook, are not presented as new discoveries.

## Recommended next additions

### 1. GPU MODE Lectures

**Source:** GPU MODE<br>
**Area:** GPU performance and inference<br>
**Link:** https://github.com/gpu-mode/lectures

The strongest next learning source. It moves from profiling and GPU fundamentals into CUDA, Triton, CUTLASS, attention kernels, quantization, and distributed systems.

Suggested first step: complete the profiling, CUDA, Triton, FlashAttention, and quantization sequence.

### 2. Scalable AI, Spring 2026

**Source:** UC Berkeley<br>
**Area:** End-to-end AI systems<br>
**Link:** https://scalable-ai.eecs.berkeley.edu/

A current course spanning performance, pretraining, post-training, inference engines, deployment, and serving economics.

Suggested first step: audit the performance and inference blocks, then inspect the serving assignment.

### 3. Efficient Deep Learning Systems, 2026

**Source:** HSE University and Yandex School of Data Analysis<br>
**Area:** Distributed training<br>
**Link:** https://github.com/mryab/efficient-dl-systems

Implementation-heavy material covering profiling, collectives, DeviceMesh, DTensor, FSDP2, and distributed checkpointing.

Suggested first step: implement the PyTorch Distributed seminars after the Stanford CS336 systems material.

### 4. Machine Learning Systems

**Source:** Harvard and MIT Press<br>
**Area:** ML systems foundations<br>
**Link:** https://mlsysbook.ai/index.html

A unified curriculum for hardware limits, compilers, benchmarking, serving, reliability, and fleet-scale infrastructure.

Suggested first step: use the simulator and inference labs to predict a deployment before benchmarking it.

### 5. Machine Learning Engineering Open Book

**Source:** Stas Bekman<br>
**Area:** Cluster operations<br>
**Link:** https://github.com/stas00/ml-engineering

A field guide to accelerators, networking, storage, orchestration, diagnostics, and failed training runs.

Suggested first step: use it as the operational companion to every multi-GPU experiment.

### 6. CS 285: Deep Reinforcement Learning

**Source:** UC Berkeley<br>
**Area:** World-model foundations<br>
**Link:** https://rail.eecs.berkeley.edu/deeprlcourse/

Provides the model-based reinforcement learning, planning, variational inference, and control theory needed for the World Model Platform.

Suggested first step: study the model-based RL block before selecting the first world-model workload.

### 7. DreamerV3

**Source:** Danijar Hafner and collaborators<br>
**Area:** World models<br>
**Link:** https://github.com/danijar/dreamerv3

A complete first workload for validating training, replay, checkpointing, evaluation, and imagined rollouts.

Suggested first step: run one small pixel task and turn its data, artifacts, and metrics into platform contracts.

### 8. Kubeflow Community Distribution

**Source:** Kubeflow<br>
**Area:** AI platform engineering<br>
**Link:** https://github.com/kubeflow/community-distribution

A runnable integration reference for Pipelines, Trainer, KServe, model registry, tenancy, authentication, and Ray.

Suggested first step: map its components and select a minimal subset rather than installing everything.

### 9. How to Scale Your Model

**Source:** Google and JAX contributors<br>
**Area:** Distributed systems theory<br>
**Link:** https://jax-ml.github.io/scaling-book/

A derivation-based treatment of rooflines, collectives, sharding, topology, scaling limits, and inference.

Suggested first step: build a notebook that predicts memory and communication costs before each training run.

### 10. Stanford CS349D: AI Inference Infrastructure

**Source:** Stanford University<br>
**Area:** Inference systems<br>
**Link:** https://web.stanford.edu/class/cs349d/

A serving-engine curriculum covering continuous batching, PagedAttention, context caching, speculative decoding, and prefill/decode disaggregation.

Suggested first step: use its project milestones as a design rubric even if all starter code is not public.

## Extensions to Camilo's inference path

These resources add layers that are not deeply covered by the existing inference guide.

### Machine Learning Compilation

**Link:** https://mlc.ai/summer22/

Adds intermediate representations, graph lowering, TensorIR, scheduling, autotuning, and hardware code generation.

Experiment: lower and optimize a Transformer operator, then explain how the optimization changes its roofline position.

### Kubernetes Inference Perf

**Link:** https://github.com/kubernetes-sigs/inference-perf

Adds trace replay, shared-prefix workloads, multi-turn traffic, goodput, and Kubernetes-native load generation.

Experiment: find the saturation point at which p99 time to first token violates an explicit SLO.

### BLIS Inference Simulator

**Link:** https://inference-sim.github.io/inference-sim/

A CPU-only laboratory for batching, routing, KV-cache pressure, admission control, and capacity planning.

Experiment: compare queue-aware and cache-aware routing under bursty traffic.

### llm-d

**Link:** https://github.com/llm-d/llm-d

Adds a Kubernetes-native control plane for routing, disaggregation, flow control, and model-aware scheduling.

Experiment: compare round-robin and prefix-aware routing using a small model pool.

### LMCache

**Link:** https://github.com/LMCache/LMCache

Makes KV-cache reuse, offload, transport, eviction, persistence, and observability concrete.

Experiment: measure time to first token and cache-hit ratio for repeated long-prefix requests.

### LLM Compressor

**Link:** https://github.com/vllm-project/llm-compressor

Turns quantization theory into calibrated FP8, INT8, INT4, GPTQ, AWQ, and KV-cache artifacts.

Experiment: compare BF16 and one compressed 1B to 3B model on size, quality, latency, and throughput.

### KEDA

**Link:** https://github.com/kedacore/keda

Adds metric-driven Kubernetes autoscaling and scale-to-zero behavior.

Experiment: scale a deployment from queue-depth metrics while replaying bursty inference traffic.

### Chaos Mesh

**Link:** https://github.com/chaos-mesh/chaos-mesh

Adds deliberate pod, network, I/O, DNS, and stress failures.

Experiment: terminate a decode worker and add network latency while asserting availability and tail-latency SLOs.

### Zeus

**Link:** https://github.com/ml-energy/zeus

Measures CPU, GPU, and DRAM energy so performance work can optimize joules per token.

Experiment: sweep GPU power limits and compare latency against energy per output token.

### Sarathi-Serve

**Link:** https://github.com/microsoft/sarathi-serve

Explains chunked prefill and stall-free scheduling as an alternative to complete phase disaggregation.

Experiment: reproduce the prefill chunk-size versus decode-latency tradeoff.

### CacheGen

**Link:** https://github.com/UChi-JCL/CacheGen

Treats KV state as a bandwidth-adaptive compressed object that can be streamed between systems.

Experiment: measure compression, transfer latency, and quality under several network bandwidth limits.

## Additional strong resources

### Inference systems

- [Inference Engineering](https://inferenceengineering.tech/): production-focused map of hardware, engines, capacity, SLOs, autoscaling, and cost
- [FlashInfer](https://docs.flashinfer.ai/): kernel substrate for decode, prefill, paged KV caches, MLA, MoE, and low precision
- [GuideLLM](https://github.com/vllm-project/guidellm): SLO-oriented load testing and capacity sweeps
- [MLPerf Inference](https://github.com/mlcommons/inference): standardized benchmark methodology and compliance
- [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo): distributed inference, KV routing, cache transfer, and disaggregation
- [Mooncake](https://github.com/kvcache-ai/Mooncake): distributed KV-cache storage and transfer
- [Llumnix](https://github.com/AlibabaPAI/llumnix): live request and KV-state migration across serving instances
- [LoongServe](https://github.com/LoongServe/LoongServe): elastic sequence parallelism for long-context serving
- [QServe](https://github.com/mit-han-lab/qserve): quantization and kernel co-design for serving
- [Helix](https://github.com/Thesys-lab/Helix-ASPLOS25): heterogeneous GPU placement and routing

### Distributed training

- [PyTorch Distributed and TorchTitan](https://github.com/pytorch/torchtitan): composable FSDP2, TP, PP, CP, elasticity, and checkpointing
- [NCCL tests](https://github.com/NVIDIA/nccl-tests): collective performance and topology validation
- [Megatron Core](https://docs.nvidia.com/megatron-core/developer-guide/latest/): production model parallelism and MoE
- [JAX distributed programming](https://docs.jax.dev/en/latest/notebooks/Distributed_arrays_and_automatic_parallelization.html): explicit sharding and multi-host arrays
- [MLPerf Storage](https://github.com/mlcommons/storage): realistic data-ingestion and checkpoint-storage benchmarks

### Platform engineering

- [KubeRay](https://docs.ray.io/en/latest/cluster/kubernetes/index.html): distributed data, simulation, training, evaluation, and serving
- [KServe](https://kserve.github.io/website/docs/intro): Kubernetes-native serving control plane
- [Kubeflow Trainer](https://www.kubeflow.org/docs/components/trainer/): declarative distributed training jobs
- [KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler): GPU-aware scheduling, quotas, preemption, and topology
- [OpenCost](https://github.com/opencost/opencost): workload and GPU cost attribution
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter): GPU telemetry for Prometheus
- [Google SRE Book, Part III](https://sre.google/sre-book/part-III-practices/): SLOs, overload, cascading failures, and reliable launches

### World models and embodied AI

- [TD-MPC2](https://github.com/nicklashansen/tdmpc2): decoder-free latent world models and trajectory optimization
- [V-JEPA 2](https://github.com/facebookresearch/vjepa2): video representation learning and action-conditioned prediction
- [LeRobotDataset v3](https://huggingface.co/docs/lerobot/lerobot-dataset-v3): multimodal robotics data contract
- [ManiSkill 3](https://github.com/haosulab/ManiSkill): GPU-parallel simulation and reproducible trajectories
- [WorldBench](https://world-bench.github.io/): physical-consistency evaluation for world models
- [NVIDIA Cosmos](https://github.com/NVIDIA/Cosmos): large-scale world foundation model reference architecture

## Suggested sequence

### Phase 1: Build the systems model

Continue Stanford CS336, then study Scalable AI, Machine Learning Systems, and selected chapters from How to Scale Your Model.

### Phase 2: Learn to measure

Use GPU MODE profiling, Kubernetes Inference Perf, NCCL tests, BLIS, GuideLLM, and MLPerf Storage.

### Phase 3: Implement the internals

Work through TorchTitan, Machine Learning Compilation, FlashInfer, and selected CUDA or Triton kernels.

### Phase 4: Define a real workload

Study Berkeley CS285 and run DreamerV3 on one small ManiSkill task. Define the dataset, checkpoints, metrics, and evaluation jobs before building a large platform.

### Phase 5: Build the minimum platform

Choose one training control plane, one serving control plane, and one observability stack. Add routing, caching, autoscaling, and chaos testing only after baseline behavior is measured.

## Recommended repository strategy

Add only two learning folders next:

1. GPU MODE for performance engineering
2. Berkeley CS285 for model-based reinforcement learning foundations

Let actual World Model Platform bottlenecks determine whether the following source should cover TorchTitan, KubeRay, KServe, FlashInfer, storage, or cluster operations.

Avoid broad MLOps catalogs, generic certification tracks, and vendor product tours. They add vocabulary but little systems intuition, and the introductory course already covers that layer.
