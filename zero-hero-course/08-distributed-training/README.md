
# Lab 56 — Compare Multi-GPU Training Strategies

## Goal
Train **ResNet-18** on **CIFAR-10** using several PyTorch distributed training strategies and compare their performance.

**Estimated Time:** 2–3 hours

**Tools**

- Python 3.10+
- PyTorch
- torchvision
- CUDA-enabled system with 2+ NVIDIA GPUs
- (Optional) Hugging Face Accelerate
- (Optional) DeepSpeed

---

# 1. Verify the Environment

```bash
nvidia-smi
```

```python
import torch

print("CUDA:", torch.cuda.is_available())
print("GPUs:", torch.cuda.device_count())

for i in range(torch.cuda.device_count()):
    print(i, torch.cuda.get_device_name(i))
```

---

# 2. Prepare the Dataset

```python
import torchvision
import torchvision.transforms as transforms

transform = transforms.Compose([
    transforms.RandomHorizontalFlip(),
    transforms.RandomCrop(32, padding=4),
    transforms.ToTensor()
])

trainset = torchvision.datasets.CIFAR10(
    root="./data",
    train=True,
    download=True,
    transform=transform
)

testset = torchvision.datasets.CIFAR10(
    root="./data",
    train=False,
    download=True,
    transform=transforms.ToTensor()
)
```

---

# 3. Build the Model

```python
import torchvision.models as models

def build_model():
    return models.resnet18(weights=None, num_classes=10)
```

---

# 4. Strategy 1 — Single GPU

```python
import torch
import torch.nn as nn
import torch.optim as optim

device = "cuda"

model = build_model().to(device)

criterion = nn.CrossEntropyLoss()
optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.9)
```

Launch:

```bash
python train_single.py
```

Record epoch time and GPU memory.

---

# 5. Strategy 2 — DataParallel

Replace model creation with:

```python
import torch.nn as nn

model = build_model()

model = nn.DataParallel(model)

model = model.cuda()
```

Launch:

```bash
python train_dataparallel.py
```

Observe that only one Python process is created while all GPUs are used.

---

# 6. Strategy 3 — DistributedDataParallel (DDP)

Create `train_ddp.py`.

```python
import torch
import torch.distributed as dist
import torch.multiprocessing as mp
import torch.nn as nn
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms
import torchvision.models as models

from torch.nn.parallel import DistributedDataParallel as DDP

def setup(rank, world_size):
    dist.init_process_group("nccl", rank=rank, world_size=world_size)
    torch.cuda.set_device(rank)

def cleanup():
    dist.destroy_process_group()

def train(rank, world_size):

    setup(rank, world_size)

    transform = transforms.Compose([transforms.ToTensor()])

    trainset = torchvision.datasets.CIFAR10(
        root="./data",
        train=True,
        download=True,
        transform=transform
    )

    sampler = torch.utils.data.distributed.DistributedSampler(
        trainset,
        num_replicas=world_size,
        rank=rank
    )

    loader = torch.utils.data.DataLoader(
        trainset,
        batch_size=128,
        sampler=sampler
    )

    model = models.resnet18(weights=None, num_classes=10).to(rank)

    model = DDP(model, device_ids=[rank])

    criterion = nn.CrossEntropyLoss().to(rank)
    optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.9)

    for epoch in range(2):

        sampler.set_epoch(epoch)

        for images, labels in loader:

            images = images.to(rank)
            labels = labels.to(rank)

            optimizer.zero_grad()

            loss = criterion(model(images), labels)

            loss.backward()

            optimizer.step()

    if rank == 0:
        torch.save(model.state_dict(), "resnet_ddp.pth")

    cleanup()

def main():
    world_size = torch.cuda.device_count()
    mp.spawn(train, args=(world_size,), nprocs=world_size)

if __name__ == "__main__":
    main()
```

Launch:

```bash
torchrun --standalone --nproc_per_node=2 train_ddp.py
```

---

# 7. Strategy 4 — Fully Sharded Data Parallel (FSDP)

Install (if necessary):

```bash
pip install torch
```

Replace the DDP wrapper with:

```python
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP

model = build_model().to(rank)

model = FSDP(model)
```

Everything else remains nearly identical.

Launch:

```bash
torchrun --standalone --nproc_per_node=2 train_fsdp.py
```

Compare GPU memory usage against DDP.

---

# 8. Strategy 5 — Hugging Face Accelerate

Install:

```bash
pip install accelerate
```

Configure once:

```bash
accelerate config
```

Training script:

```python
from accelerate import Accelerator
import torch.optim as optim

accelerator = Accelerator()

model = build_model()

optimizer = optim.SGD(
    model.parameters(),
    lr=0.01,
    momentum=0.9
)

model, optimizer, trainloader = accelerator.prepare(
    model,
    optimizer,
    trainloader
)
```

Launch:

```bash
accelerate launch train_accelerate.py
```

---

# 9. Strategy 6 — DeepSpeed

Install:

```bash
pip install deepspeed
```

Create `ds_config.json`

```json
{
  "train_batch_size": 128,
  "zero_optimization": {
    "stage": 2
  }
}
```

Initialize:

```python
import deepspeed

model_engine, optimizer, _, _ = deepspeed.initialize(
    model=model,
    optimizer=optimizer,
    config="ds_config.json"
)
```

Launch:

```bash
deepspeed train_deepspeed.py
```

---

# 10. Monitor GPU Utilization

```bash
watch -n 1 nvidia-smi
```

Optional:

```bash
nvidia-smi dmon
```

---

# 11. Benchmark Results

| Strategy | GPUs | Epoch Time | Peak GPU Memory | Notes |
|-----------|-----:|-----------:|----------------:|-------|
| Single GPU | 1 | | | |
| DataParallel | 2 | | | |
| DDP | 2 | | | |
| FSDP | 2 | | | |
| Accelerate | 2 | | | |
| DeepSpeed | 2 | | | |

---

# 12. Cleanup

```bash
pkill -f train
```

---

# Deliverables

- Screenshot of `nvidia-smi`
- Training logs
- Completed benchmark table
- Short reflection comparing all strategies.

---

# Learning Outcomes

- Train on a single GPU.
- Train using DataParallel.
- Train using DistributedDataParallel.
- Train using Fully Sharded Data Parallel.
- Launch distributed jobs with Hugging Face Accelerate.
- Understand the basics of DeepSpeed ZeRO.
- Compare speed, scalability, and GPU memory usage.
