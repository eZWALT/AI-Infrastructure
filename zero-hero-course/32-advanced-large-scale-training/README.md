# Day 224 Lab: Train with DeepSpeed ZeRO-3

## Learning Goals
* Configure **DeepSpeed ZeRO-3** for sharded training
* Train a Hugging Face Transformer model at scale
* Monitor GPU memory savings and throughput improvements
* Save and reload ZeRO-3 checkpoints

---

## 0. Prerequisites
* Multi-GPU system or cluster (NCCL backend)
* Install dependencies:

```bash
pip install torch transformers datasets deepspeed
```

* Verify DeepSpeed installation:

```bash
deepspeed --version
```

---

## 1. Prepare Dataset & Model
We’ll use **BERT fine-tuned on IMDb sentiment classification**.

Create a script file named `train_ds_zero3.py`:

```python
import torch
from transformers import AutoTokenizer, AutoModelForSequenceClassification, Trainer, TrainingArguments
from datasets import load_dataset

# Load dataset
dataset = load_dataset("imdb")
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

def tokenize(batch):
    return tokenizer(batch["text"], truncation=True, padding="max_length", max_length=128)

dataset = dataset.map(tokenize, batched=True)
dataset.set_format("torch", columns=["input_ids", "attention_mask", "label"])

# Model
model = AutoModelForSequenceClassification.from_pretrained("bert-base-uncased", num_labels=2)

# Training args with DeepSpeed ZeRO-3 config
training_args = TrainingArguments(
    output_dir="./outputs",
    per_device_train_batch_size=4,
    evaluation_strategy="steps",
    num_train_epochs=1,
    save_steps=500,
    logging_steps=50,
    report_to="none",
    deepspeed="ds_config_zero3.json",  # Link to config file
)
```

---

## 2. DeepSpeed ZeRO-3 Config
Create a configuration file named `ds_config_zero3.json`:

```json
{
  "train_batch_size": 32,
  "train_micro_batch_size_per_gpu": 4,
  "gradient_accumulation_steps": 2,
  "zero_optimization": {
    "stage": 3,
    "overlap_comm": true,
    "contiguous_gradients": true,
    "reduce_bucket_size": 5e8,
    "stage3_prefetch_bucket_size": 5e8,
    "stage3_param_persistence_threshold": 1e5,
    "offload_optimizer": {
      "device": "cpu",
      "pin_memory": true
    }
  },
  "bf16": { "enabled": true },
  "gradient_clipping": 1.0,
  "steps_per_print": 100,
  "wall_clock_breakdown": false
}
```

---

## 3. Integrate with Trainer
Complete `train_ds_zero3.py` by adding the execution logic:

```python
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=dataset["train"].shuffle().select(range(5000)),  # Subset for speed
    eval_dataset=dataset["test"].select(range(1000)),
)

trainer.train()
```

---

## 4. Run Training with DeepSpeed
Launch training across 2 GPUs using the DeepSpeed launcher:

```bash
deepspeed --num_gpus=2 train_ds_zero3.py
```

> **Expected Outcome:** Lower GPU memory per device vs vanilla DDP. Checkpoints are saved in sharded format under `./outputs`.

---

## 5. Reload Checkpoint
Validate loading the saved ZeRO-3 checkpoint:

```python
from transformers import AutoModelForSequenceClassification

model = AutoModelForSequenceClassification.from_pretrained("./outputs/checkpoint-500")
print("Checkpoint reloaded successfully!")
```

---

## 6. Monitor GPU Memory Savings
Monitor GPU utilization using `nvidia-smi` or DeepSpeed log outputs:
* Memory per GPU should drop significantly compared to non-ZeRO runs.
* Larger batch sizes are possible without running into Out-Of-Memory (OOM) errors.

---

## 7. Stretch Goals
* Try a larger model (e.g., `roberta-large`, `gpt2-xl`)
* Scale to multiple nodes with SLURM or Kubernetes
* Enable **ZeRO-Infinity** to offload parameters/activations to NVMe
* Profile `tokens/sec` and compare throughput vs standard DDP

---

> **Outcome:** You trained a Transformer with **DeepSpeed ZeRO-3**, observed memory savings, and learned how to scale beyond single-GPU limits.
