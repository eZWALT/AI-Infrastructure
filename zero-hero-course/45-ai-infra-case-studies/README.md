# 🧪 315. Lab – Analyze Case Study Infra Architecture

## 🎯 Objective
- Select one **real-world AI infra case study**.
- Decompose its architecture into **compute**, **data**, **orchestration**, **serving**, and **governance**.
- Identify trade-offs, risks, and lessons for your own infra design.

---

## Step 1: Choose a Case Study

Pick **one** of the following:
- **OpenAI** (GPT infra)
- **Google DeepMind** (TPU + JAX systems)
- **Meta** (LLaMA infra)
- **Tesla** (FSD + Dojo + fleet data)
- **Netflix** (recsys infra)
- **Healthcare** (federated, compliant AI infra)

> **✅ Expected:** Case study selected with clear focus.

---

## Step 2: Map the Infra Stack

For the chosen case, draw out the stack:
- **Compute layer:** GPUs, TPUs, ASICs, custom chips
- **Networking:** interconnects, latency optimization, replication
- **Data pipelines:** ingestion, cleaning, storage, feature stores
- **Training orchestration:** distributed training (e.g., FSDP, ZeRO, JAX/XLA)
- **Serving layer:** APIs, multi-tenant infra, edge nodes
- **Governance:** compliance, safety, monitoring

> 📌 *Use a whiteboard tool (Miro, Lucidchart) or paper sketch.*  
> **✅ Expected:** Architecture diagram of chosen case.

---

## Step 3: Identify Key Optimizations

Analyze where infra design is **optimized for scale**:
- **OpenAI** → ZeRO sharding + Azure supercomputers
- **DeepMind** → TPUs + JAX/XLA compiler stack
- **Meta** → PyTorch FSDP + open-weight efficiency
- **Tesla** → Dojo + OTA fleet learning loop
- **Netflix** → low-latency recsys pipelines + A/B infra
- **Healthcare** → federated learning + compliance-first pipelines

> **✅ Expected:** List of 3–5 optimizations unique to the case.

---

## Step 4: Evaluate Trade-Offs

Answer:
- What did the infra prioritize? *(cost, speed, openness, compliance?)*
- What trade-offs were made?
  - *Ex: Tesla prioritized fleet data scale but risked regulatory hurdles.*
  - *Ex: Meta prioritized openness but limited commercial monetization.*
  - *Ex: Healthcare infra prioritizes compliance over raw efficiency.*

> **✅ Expected:** Written analysis (200–300 words).

---

## Step 5: Compare with Another Case

Contrast your chosen case with **one other**:
- How is Netflix’s infra different from Healthcare?
- Why does Tesla’s infra diverge from OpenAI’s?
- What can be borrowed across domains?

> **✅ Expected:** 1–2 paragraph comparison.

---

## Step 6: Reflection Questions

1. If you were architect of this infra, what would you do differently?
2. Could this infra scale globally under new constraints *(regulation, cost, latency)*?
3. What are the **generalizable lessons** for building AI infra in your own domain?

> **✅ Expected:** Short answers to reflection prompts.

---

## Step 7 (Optional Extension): Re-Architect It

Redesign the infra for **a different constraint** *(e.g., lower budget, higher compliance, edge-first)*.

- *Example:* How would OpenAI GPT infra look if it had to run under EU AI Act compliance?
- *Example:* How would Tesla FSD infra adapt for regions with poor internet coverage?

> **✅ Expected:** Modified architecture sketch + rationale.

---

## 📝 Wrap-Up

In this lab, you:
- Chose a real-world case study (*OpenAI, DeepMind, Meta, Tesla, Netflix, Healthcare*).
- Broke down its **infrastructure layers**.
- Analyzed **optimizations, trade-offs, and risks**.
- Compared across domains and reflected on lessons for your own infra designs.
