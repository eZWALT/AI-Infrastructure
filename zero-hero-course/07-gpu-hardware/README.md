# Lab 49 — Train a PyTorch Model on GPU with CUDA

**Goal:** Train and evaluate a neural network using a CUDA-enabled GPU with PyTorch, monitor GPU utilization, and compare CPU vs GPU performance.

**Estimated Time:** 60–90 minutes

**Requirements:**

- Python 3.10+
- PyTorch with CUDA support
- NVIDIA GPU
- NVIDIA Drivers + CUDA Toolkit
- `nvidia-smi`

---

# Learning Objectives

By the end of this lab you will be able to:

- Verify CUDA installation
- Move tensors and models between CPU and GPU
- Train a neural network using CUDA
- Monitor GPU utilization during training
- Compare CPU and GPU execution times
- Understand the PyTorch CUDA workflow

---

# Project Structure

```
.
├── train.py
└── data/
```

---

# Step 1 — Verify Your GPU

Before writing any code, verify that the NVIDIA drivers are correctly installed.

```bash
nvidia-smi
```

Example output:

```
+----------------------------------------------------+
| GPU  Name               Memory-Usage               |
|  0   NVIDIA RTX 4090    800MiB / 24564MiB          |
+----------------------------------------------------+
```

If `nvidia-smi` is not found or reports no devices, verify your NVIDIA drivers before continuing.

---

## Verify CUDA from PyTorch

Create a small script:

```python
import torch

print("CUDA Available:", torch.cuda.is_available())
print("CUDA Version:", torch.version.cuda)
print("GPUs:", torch.cuda.device_count())

if torch.cuda.is_available():
    print("Device:", torch.cuda.get_device_name(0))
```

Expected output:

```
CUDA Available: True
CUDA Version: 12.x
GPUs: 1
Device: NVIDIA RTX 4090
```

---

# Step 2 — Load the Dataset

We'll use the MNIST handwritten digits dataset.

```python
import torch
import torchvision
import torchvision.transforms as transforms

transform = transforms.Compose([
    transforms.ToTensor()
])

trainset = torchvision.datasets.MNIST(
    root="data",
    train=True,
    download=True,
    transform=transform
)

trainloader = torch.utils.data.DataLoader(
    trainset,
    batch_size=64,
    shuffle=True
)
```

---

# Step 3 — Build a Neural Network

Create a simple Multi-Layer Perceptron.

```python
import torch.nn as nn
import torch.nn.functional as F

class Net(nn.Module):

    def __init__(self):
        super().__init__()

        self.fc1 = nn.Linear(28 * 28, 128)
        self.fc2 = nn.Linear(128, 64)
        self.fc3 = nn.Linear(64, 10)

    def forward(self, x):

        x = x.view(-1, 28 * 28)

        x = F.relu(self.fc1(x))
        x = F.relu(self.fc2(x))

        return self.fc3(x)

model = Net()
```

---

# Step 4 — Move the Model to the GPU

Select the execution device:

```python
device = torch.device(
    "cuda" if torch.cuda.is_available() else "cpu"
)

model = model.to(device)
```

Every tensor used during training must also be moved to the same device.

---

# Step 5 — Configure Training

```python
import torch.optim as optim

criterion = nn.CrossEntropyLoss()

optimizer = optim.Adam(
    model.parameters(),
    lr=1e-3
)
```

---

# Step 6 — Train on the GPU

```python
for epoch in range(2):

    running_loss = 0

    for images, labels in trainloader:

        images = images.to(device)
        labels = labels.to(device)

        optimizer.zero_grad()

        outputs = model(images)

        loss = criterion(outputs, labels)

        loss.backward()

        optimizer.step()

        running_loss += loss.item()

    print(
        f"Epoch {epoch+1}: "
        f"{running_loss/len(trainloader):.4f}"
    )
```

You should observe the loss decreasing after each epoch.

---

# Step 7 — Monitor GPU Utilization

Open another terminal while training.

```bash
watch -n 1 nvidia-smi
```

Observe:

- GPU utilization
- Memory usage
- Running Python processes
- CUDA compute usage

This confirms that training is executing on the GPU.

---

# Step 8 — Evaluate the Model

```python
testset = torchvision.datasets.MNIST(
    root="data",
    train=False,
    download=True,
    transform=transform
)

testloader = torch.utils.data.DataLoader(
    testset,
    batch_size=100
)

correct = 0
total = 0

with torch.no_grad():

    for images, labels in testloader:

        images = images.to(device)
        labels = labels.to(device)

        outputs = model(images)

        _, predicted = outputs.max(1)

        total += labels.size(0)
        correct += (predicted == labels).sum().item()

print(f"Accuracy: {100*correct/total:.2f}%")
```

Expected accuracy:

```
≈95–98%
```

after only a few epochs.

---

# Step 9 — Compare CPU vs GPU Performance

Measure training time using Python's `time` module.

```python
import time

start = time.time()

# training loop

end = time.time()

print(f"Training time: {end-start:.2f}s")
```

Run the experiment twice.

## CPU

```python
device = "cpu"
```

## GPU

```python
device = "cuda"
```

Record your results.

| Device | Time (s) | Speedup |
|---------|---------:|---------:|
| CPU | | |
| GPU | | |

---

# Bonus Experiments

Try modifying one variable at a time and observe its effect on training speed and GPU memory usage.

### Larger Batch Sizes

```python
batch_size = 256
```

### More Epochs

```python
range(10)
```

### Deeper Network

Add additional hidden layers.

### Larger Hidden Dimension

```python
nn.Linear(784, 512)
```

### Mixed Precision (Optional)

If your GPU supports Tensor Cores, experiment with Automatic Mixed Precision (AMP) using `torch.cuda.amp`.

---

# Deliverables

Submit:

- Screenshot of `nvidia-smi` during training
- Output showing CUDA detection
- Final test accuracy
- CPU vs GPU timing comparison
- A short reflection (≈150 words) discussing the observed performance difference

---

# Summary

In this lab you learned how to:

- Verify CUDA support with PyTorch
- Move models and tensors to the GPU
- Train a neural network using CUDA
- Monitor GPU utilization with `nvidia-smi`
- Evaluate a trained model
- Measure the performance difference between CPU and GPU execution

This workflow forms the foundation of GPU-accelerated deep learning with PyTorch and prepares you for training larger models on modern NVIDIA hardware.

# Extra Challenges — Going Beyond Basic CUDA

These optional exercises build upon the main lab and introduce many of the optimization techniques used in real-world deep learning systems. They are designed to be completed independently and can typically be finished within an afternoon.

---

# Challenge 1 — Benchmark CPU vs GPU Correctly

GPU operations execute **asynchronously**, meaning Python may continue execution before CUDA kernels have completed. To obtain accurate timing measurements, synchronize the GPU before stopping the timer.

```python
import time
import torch

device = "cuda" if torch.cuda.is_available() else "cpu"

start = time.time()

# ------------------------
# Training or inference
# ------------------------

if device == "cuda":
    torch.cuda.synchronize()

end = time.time()

print(f"Execution time: {end-start:.3f} seconds")
```

Run the benchmark twice:

```python
device = "cpu"
```

and

```python
device = "cuda"
```

Record your results.

| Device | Execution Time (s) | Speedup |
|---------|-------------------:|---------:|
| CPU | | |
| GPU | | |

---

# Challenge 2 — Explore GPU Memory Usage

PyTorch exposes several APIs to inspect GPU memory consumption.

```python
print("Allocated:",
      torch.cuda.memory_allocated()/1024**2,
      "MB")

print("Reserved:",
      torch.cuda.memory_reserved()/1024**2,
      "MB")
```

Reset the peak statistics before training:

```python
torch.cuda.reset_peak_memory_stats()
```

After training:

```python
print(
    "Peak:",
    torch.cuda.max_memory_allocated()/1024**2,
    "MB"
)
```

Print a detailed memory report:

```python
print(torch.cuda.memory_summary())
```

Experiment with different batch sizes and observe how memory allocation changes.

---

# Challenge 3 — Experiment with Batch Size

Modify your DataLoader:

```python
trainloader = torch.utils.data.DataLoader(
    trainset,
    batch_size=32,
    shuffle=True
)
```

Repeat the experiment with:

```python
batch_size = 64
```

```python
batch_size = 128
```

```python
batch_size = 256
```

```python
batch_size = 512
```

For each experiment measure:

- Training time
- Peak GPU memory
- Final accuracy

| Batch Size | Time (s) | Peak Memory (MB) | Accuracy |
|------------:|---------:|-----------------:|----------:|
| 32 | | | |
| 64 | | | |
| 128 | | | |
| 256 | | | |
| 512 | | | |

Observe how increasing the batch size affects throughput, memory consumption, and convergence.

---

# Challenge 4 — Automatic Mixed Precision (AMP)

Modern NVIDIA GPUs contain **Tensor Cores** capable of performing FP16 and BF16 operations much faster than FP32.

Automatic Mixed Precision allows PyTorch to automatically select lower precision where it is numerically safe.

Replace your training loop with:

```python
scaler = torch.cuda.amp.GradScaler()

for images, labels in trainloader:

    images = images.to(device)
    labels = labels.to(device)

    optimizer.zero_grad()

    with torch.cuda.amp.autocast():

        outputs = model(images)
        loss = criterion(outputs, labels)

    scaler.scale(loss).backward()

    scaler.step(optimizer)
    scaler.update()
```

Measure:

| Configuration | Training Time | Peak Memory | Accuracy |
|---------------|--------------:|------------:|----------:|
| FP32 | | | |
| AMP | | | |

Questions:

- Does AMP reduce memory usage?
- Does your GPU support Tensor Cores?
- Is the accuracy significantly affected?

---

# Challenge 5 — Profile Your Model

PyTorch includes a built-in profiler capable of measuring CPU execution, CUDA kernels, memory allocations, and operator timings.

Create a profiler:

```python
import torch.profiler

with torch.profiler.profile(

    activities=[
        torch.profiler.ProfilerActivity.CPU,
        torch.profiler.ProfilerActivity.CUDA
    ],

    record_shapes=True,
    profile_memory=True,

    with_stack=True

) as prof:

    outputs = model(images)

print(
    prof.key_averages().table(
        sort_by="cuda_time_total",
        row_limit=20
    )
)
```

Export the trace:

```python
prof.export_chrome_trace("trace.json")
```

You can visualize this trace using Chrome.

Open:

```
chrome://tracing
```

Load:

```
trace.json
```

or visualize it with TensorBoard:

```bash
pip install tensorboard
```

```bash
tensorboard --logdir .
```

Things to investigate:

- Which CUDA kernels consume the most time?
- Which operations allocate the most memory?
- Does data loading become a bottleneck?
- Which layer is the slowest?

---

# Challenge 6 — Visualize the Computation Graph

Install:

```bash
pip install torchviz graphviz
```

Generate the graph:

```python
from torchviz import make_dot

output = model(images)

graph = make_dot(
    output,
    params=dict(model.named_parameters())
)

graph.render("network")
```

Inspect:

- Input tensors
- Hidden layers
- Parameters
- Gradient flow

---

# Challenge 7 — Compare Optimizers

Replace Adam with different optimizers.

SGD:

```python
optimizer = torch.optim.SGD(
    model.parameters(),
    lr=0.01
)
```

Adam:

```python
optimizer = torch.optim.Adam(
    model.parameters(),
    lr=0.001
)
```

AdamW:

```python
optimizer = torch.optim.AdamW(
    model.parameters(),
    lr=0.001
)
```

Compare:

| Optimizer | Training Time | Accuracy |
|-----------|--------------:|----------:|
| SGD | | |
| Adam | | |
| AdamW | | |

---

# Challenge 8 — Compare Different Architectures

Replace the MLP with modern architectures.

Example:

```python
import torchvision.models as models

model = models.resnet18(
    num_classes=10
)
```

Try:

- LeNet
- ResNet18
- MobileNetV3
- EfficientNet-B0

Record:

| Model | Parameters | Time | Accuracy |
|--------|-----------:|-----:|----------:|
| MLP | | | |
| LeNet | | | |
| ResNet18 | | | |
| MobileNetV3 | | | |
| EfficientNet-B0 | | | |

---

# Challenge 9 — Optimize Data Loading

Modify your DataLoader.

Baseline:

```python
trainloader = DataLoader(
    trainset,
    batch_size=64
)
```

Optimized:

```python
trainloader = DataLoader(
    trainset,
    batch_size=64,
    shuffle=True,
    num_workers=4,
    pin_memory=True,
    persistent_workers=True
)
```

Experiment with:

```python
num_workers = 0
```

```python
num_workers = 2
```

```python
num_workers = 4
```

```python
num_workers = 8
```

Observe GPU utilization using:

```bash
watch -n 1 nvidia-smi
```

---

# Challenge 10 — Multi-GPU Training

Check available GPUs.

```python
import torch

print(torch.cuda.device_count())
```

If more than one GPU is available:

```python
model = torch.nn.DataParallel(model)

model = model.to(device)
```

Train exactly as before.

Measure:

- Training speed
- GPU utilization
- Throughput

---

# Challenge 11 — Monitor GPU in Real Time

While training, open another terminal.

```bash
watch -n 1 nvidia-smi
```

Alternative:

```bash
nvidia-smi dmon
```

Continuous logging:

```bash
nvidia-smi --loop=1
```

Observe:

- GPU utilization
- Memory usage
- Temperature
- Power draw
- Clock frequency
- Running CUDA processes

---

# Challenge 12 — Measure Throughput

Measure the number of processed images per second.

```python
training_time = end - start

throughput = len(trainset) / training_time

print(
    f"{throughput:.2f} images/sec"
)
```

Compare:

| Device | Images/sec |
|---------|-----------:|
| CPU | |
| GPU | |

---

# Challenge 13 — Save and Reload Your Model

Save the trained model.

```python
torch.save(
    model.state_dict(),
    "mnist.pt"
)
```

Reload:

```python
model = Net()

model.load_state_dict(
    torch.load("mnist.pt")
)

model.eval()
```

Run inference again and verify that the predictions remain unchanged.

---

# Challenge 14 — Export the Model to ONNX

ONNX (Open Neural Network Exchange) is an interoperable model format that allows models trained in PyTorch to run in other inference engines such as **ONNX Runtime**, **TensorRT**, **OpenVINO**, or **NVIDIA Triton Inference Server**.

Install the required packages:

```bash
pip install onnx onnxruntime
```

Export the model:

```python
dummy = torch.randn(
    1,
    1,
    28,
    28
).to(device)

torch.onnx.export(

    model,

    dummy,

    "mnist.onnx",

    input_names=["input"],

    output_names=["output"],

    dynamic_axes={
        "input": {0: "batch"},
        "output": {0: "batch"}
    },

    opset_version=17

)
```

Validate the exported model:

```python
import onnx

model = onnx.load("mnist.onnx")

onnx.checker.check_model(model)

print("Model is valid!")
```

Run inference using ONNX Runtime:

```python
import onnxruntime as ort
import numpy as np

session = ort.InferenceSession(
    "mnist.onnx"
)

dummy = np.random.rand(
    1,
    1,
    28,
    28
).astype(np.float32)

output = session.run(
    None,
    {"input": dummy}
)

print(output[0].shape)
```

Compare:

- PyTorch inference latency
- ONNX Runtime latency
- Model file size

Research:

- TensorRT
- OpenVINO
- Triton Inference Server

How do these systems optimize inference compared to native PyTorch?

---

# Challenge 15 — Stress Test Your GPU

Increase the workload until your GPU runs out of memory.

Start by increasing:

```python
batch_size = 512
```

Then:

```python
batch_size = 1024
```

```python
batch_size = 2048
```

Increase the hidden dimension:

```python
self.fc1 = nn.Linear(
    784,
    2048
)
```

If CUDA reports:

```
RuntimeError:
CUDA out of memory
```

Inspect memory usage:

```python
print(torch.cuda.memory_summary())
```

Release unused memory:

```python
torch.cuda.empty_cache()
```

Determine:

- Maximum batch size
- Maximum model size
- Peak memory usage

---

# Stretch Goal — Mini Performance Report

Prepare a short report summarizing your experiments.

Include:

- GPU model
- CUDA version
- NVIDIA Driver version
- PyTorch version
- Training time
- Test accuracy
- Peak GPU memory
- Images/sec throughput
- CPU vs GPU speedup
- Effect of AMP
- Effect of batch size
- Best-performing optimizer
- Largest model successfully trained
- ONNX export success
- Profiler screenshots
- Biggest lesson learned

---

# Challenge FINAL — Pinned Memory and Faster CPU → GPU Transfers

By default, tensors stored in CPU memory are **pageable**, meaning the operating system may move them between physical memory and disk. Before transferring pageable memory to the GPU, CUDA must first copy it into a temporary page-locked (pinned) buffer.

Pinned memory (also called **page-locked memory**) avoids this extra copy, allowing significantly faster transfers from CPU to GPU.

## Baseline DataLoader

```python
trainloader = torch.utils.data.DataLoader(
    trainset,
    batch_size=64,
    shuffle=True,
    pin_memory=False
)
```

Train the model and record the training time.

---

## Enable Pinned Memory

Now modify the DataLoader:

```python
trainloader = torch.utils.data.DataLoader(
    trainset,
    batch_size=64,
    shuffle=True,
    pin_memory=True
)
```

When moving tensors to the GPU, enable asynchronous transfers:

```python
images = images.to(
    device,
    non_blocking=True
)

labels = labels.to(
    device,
    non_blocking=True
)
```

The `non_blocking=True` argument only provides asynchronous copies when the source tensor resides in pinned memory.

---

## Benchmark

Measure training time using both configurations.

| Configuration | Time (s) | GPU Utilization |
|--------------|---------:|----------------:|
| pin_memory=False | | |
| pin_memory=True | | |

---

## Visualize GPU Activity

Monitor GPU utilization:

```bash
watch -n 1 nvidia-smi
```

For more detailed metrics:

```bash
nvidia-smi dmon
```

---

## Experiment

Repeat the benchmark with different numbers of DataLoader workers.

```python
num_workers = 0
```

```python
num_workers = 2
```

```python
num_workers = 4
```

```python
num_workers = 8
```

Record your observations.

| Workers | pin_memory | Images/sec |
|---------:|-----------:|-----------:|
| 0 | False | |
| 0 | True | |
| 4 | False | |
| 4 | True | |
| 8 | False | |
| 8 | True | |

---

## Discussion

Investigate the following questions:

- Does pinned memory always improve performance?
- Why is the speedup larger when using a GPU than a CPU?
- What happens if the dataset already resides entirely on the GPU?
- Why is `pin_memory=True` commonly enabled in production training pipelines?