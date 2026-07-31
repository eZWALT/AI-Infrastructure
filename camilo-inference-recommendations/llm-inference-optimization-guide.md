# LLM Inference Optimization — Resources

## Suggested order

Handbook Foundations → 2. kipply's arithmetic → 3. Run vLLM and benchmark → 4. Handbook Optimization + papers → 5. Kernel optimization → 6. Curated lists, ongoing.

---

## Guides

**LLM Inference Handbook** — https://handbook.modular.com/
The best entry point. Foundations → planning → model prep → optimization → kernels → infra. Interactive GPU memory and KV cache calculators. Append `.md` to any URL for clean markdown.

**Transformer Inference Arithmetic** — https://kipp.ly/p/transformer-inference-arithmetic
Back-of-envelope math for latency, bandwidth, and KV cache. The most load-bearing post here.

**Mastering LLM Techniques** (NVIDIA) — https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/
Vendor survey of the technique landscape.

**Optimizing LLMs for Speed and Memory** (Hugging Face) — https://huggingface.co/docs/transformers/llm_tutorial_optimization
Runnable code for quantization, FlashAttention, and architectural tricks.

**The Ultra-Scale Playbook** (Nanotron) — https://huggingface.co/spaces/nanotron/ultrascale-playbook?section=high-level_overview
Clearest explanation of tensor/pipeline parallelism and GPU collectives anywhere. Written for *training*, but those chapters transfer.

**LLM Inference Optimization Guide** (System Design Handbook) — https://www.systemdesignhandbook.com/blog/llm-inference-optimization/
Interview framing and trade-off vocabulary. High-level, not technically deep.

---

## Curated lists

**Awesome-LLM-Inference** — https://github.com/xlite-dev/Awesome-LLM-Inference
Papers with code: FlashAttention, PagedAttention, MLA, quantization, parallelism.

**Awesome-Efficient-LLM** — https://github.com/horseee/Awesome-Efficient-LLM
Broader, organized by technique.

**Awesome-LLM-Inference-Engine** — https://github.com/sihyeong/Awesome-LLM-Inference-Engine
Companion to a survey of ~25 open-source and commercial engines.

**Recommended Papers** (find via the lists above): Efficiently Scaling Transformer Inference → Orca → FlashAttention 1/2/3 → PagedAttention → GQA/MLA → GPTQ/AWQ/SmoothQuant → speculative decoding (Leviathan, EAGLE) → RadixAttention → DistServe.

---

## Courses

**MIT 6.5940 — TinyML and Efficient Deep Learning Computing** — https://hanlab.mit.edu/courses/2024-fall-65940
Free, rigorous, taught by Song Han. Pruning/quantization/distillation, then LLM inference and long context, then distributed training.

**Udacity — LLM Inference Optimization** — https://www.udacity.com/course/llm-inference-optimization--cd14455
Short and project-based. Good if you want guided hands-on work.

**Coursera — LLM Optimization & Evaluation** — https://www.coursera.org/specializations/llm-optimization-evaluation#courses
Pairs optimization with evaluation, which is the part most people skip. Audit mode is usually free.

**Udemy — LLM Performance Optimization path** — https://business.udemy.com/learning-path/llm-performance-optimization/
Enterprise learning path; quality is decent


---

## Engines

Gain experience, do tutorials and hands on with these inference engines:

**vLLM** — PagedAttention, continuous batching, OpenAI-compatible server. The default self-hosted choice.

**SGLang** — RadixAttention prefix caching; strong on long context, shared system prompts, and structured output.

**NVIDIA Triton Inference Server** — serving layer, not an engine: multi-model, multi-backend, dynamic batching, metrics. Usually fronts TensorRT-LLM.
