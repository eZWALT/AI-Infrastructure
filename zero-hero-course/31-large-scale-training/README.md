# Day 217 Lab: Train Transformer Across Multiple Nodes

## Learning Goals
* Understand multi-node distributed training with PyTorch Distributed Data Parallel (DDP)
* Launch training jobs with `torchrun` across nodes
* Train a BERT model on a classification task (IMDB sentiment)
* Save, resume, and evaluate the distributed model

---

## 0. Prerequisites
* Two or more GPU nodes (bare metal or cloud VMs)
* PyTorch 2.x with NCCL backend
* Shared filesystem or checkpoint sync directory
* Install required packages:

```bash
pip install torch torchvision torchaudio transformers datasets
```

---

## 1. Networking Setup
On each node, set environment variables:

```bash
export MASTER_ADDR="node0_ip"   # IP of rank 0 node
export MASTER_PORT=29500        # Any free port
export WORLD_SIZE=2             # Total number of nodes
export NODE_RANK=0              # 0 for master, 1 for worker, etc.
```

---

## 2. Training Script (DDP)
Create a file named `train_ddp.py`:

```python
import os
import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.utils.data import DataLoader, DistributedSampler
from transformers import AutoTokenizer, AutoModelForSequenceClassification
from datasets import load_dataset
 
def setup():
    dist.init_process_group("nccl")
 
def cleanup():
    dist.destroy_process_group()
 
def main():
    setup()
    rank = dist.get_rank()
    device = torch.device(f"cuda:{rank % torch.cuda.device_count()}")
 
    # Load dataset & tokenizer
    dataset = load_dataset("imdb")
    tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
 
    def tokenize(batch):
        return tokenizer(batch["text"], padding="max_length", truncation=True, max_length=128)
 
    tokenized = dataset["train"].map(tokenize, batched=True)
    tokenized.set_format("torch", columns=["input_ids", "attention_mask", "label"])
 
    sampler = DistributedSampler(tokenized)
    dataloader = DataLoader(tokenized, batch_size=8, sampler=sampler)
 
    # Model
    model = AutoModelForSequenceClassification.from_pretrained("bert-base-uncased", num_labels=2)
    model.to(device)
    model = DDP(model, device_ids=[device])
 
    optimizer = torch.optim.AdamW(model.parameters(), lr=5e-5)
    loss_fn = torch.nn.CrossEntropyLoss()
 
    # Training loop
    for epoch in range(2):
        sampler.set_epoch(epoch)
        for batch in dataloader:
            inputs = batch["input_ids"].to(device)
            attn = batch["attention_mask"].to(device)
            labels = batch["label"].to(device)
            
            outputs = model(inputs, attention_mask=attn)
            loss = loss_fn(outputs.logits, labels)
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        if rank == 0:
            print(f"Epoch {epoch} done.")
 
    if rank == 0:
        torch.save(model.module.state_dict(), "bert_ddp.pt")
 
    cleanup()
 
if __name__ == "__main__":
    main()
```

---

## 3. Launch Multi-Node Training
On each node, run the following command:

```bash
torchrun --nnodes=$WORLD_SIZE          --nproc_per_node=4 \   # GPUs per node
         --node_rank=$NODE_RANK          --master_addr=$MASTER_ADDR          --master_port=$MASTER_PORT          train_ddp.py
```

---

## 4. Validate Model Checkpoint
On the master node, execute:

```python
import torch
from transformers import AutoModelForSequenceClassification
 
model = AutoModelForSequenceClassification.from_pretrained("bert-base-uncased", num_labels=2)
model.load_state_dict(torch.load("bert_ddp.pt"))
model.eval()
print("Loaded checkpoint successfully!")
```

---

## 5. Stretch Goals
* Run on more than 2 nodes with InfiniBand interconnect
* Switch optimizer → AdamW + ZeRO/FSDP for memory scaling
* Replace IMDB with WikiText or C4 for larger runs
* Deploy with Slurm or Kubernetes + TorchElastic for elastic training

---

> **Outcome:** You trained a Transformer with multi-node DDP, learned about setup, distributed data loading, and checkpoint handling.
