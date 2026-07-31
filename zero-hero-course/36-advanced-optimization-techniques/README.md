# Day 252 Lab – Train with Mixed Precision (AMP)

## 🎯 Lab Objective
* Understand **mixed precision training** with FP16 + FP32.
* Use **Automatic Mixed Precision (AMP)** in PyTorch & TensorFlow.
* Compare **speed, memory usage, and accuracy** vs FP32 baseline.

---

## Step 1: Environment Setup
Make sure you have a **GPU that supports Tensor Cores** (Volta, Turing, Ampere, Hopper).

Install dependencies:
```bash
# PyTorch
pip install torch torchvision

# TensorFlow
pip install tensorflow
```

Verify GPU availability:
```python
import torch
print(torch.cuda.get_device_name(0))

import tensorflow as tf
print(tf.config.list_physical_devices('GPU'))
```

✅ **Expected:** You should see your GPU model listed (e.g., NVIDIA A100).

---

## Step 2: Baseline FP32 Training
We’ll start with FP32 (default) training to establish baseline.

**PyTorch (ResNet18, CIFAR-10):**
```python
import torch, torchvision
import torch.nn as nn, torch.optim as optim
from torchvision import datasets, transforms

# Data
transform = transforms.Compose([transforms.ToTensor()])
trainset = datasets.CIFAR10('./data', train=True, download=True, transform=transform)
trainloader = torch.utils.data.DataLoader(trainset, batch_size=128, shuffle=True)

# Model
model = torchvision.models.resnet18().cuda()
criterion = nn.CrossEntropyLoss()
optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.9)

# Train loop (FP32 baseline)
for epoch in range(1):
    for x, y in trainloader:
        x, y = x.cuda(), y.cuda()
        optimizer.zero_grad()
        output = model(x)
        loss = criterion(output, y)
        loss.backward()
        optimizer.step()
```

⏱ Track **time per epoch** and **GPU memory usage** (e.g., via `nvidia-smi`).

---

## Step 3: Enable PyTorch AMP

```python
scaler = torch.cuda.amp.GradScaler()

for epoch in range(1):
    for x, y in trainloader:
        x, y = x.cuda(), y.cuda()
        optimizer.zero_grad()
        # Autocast context
        with torch.cuda.amp.autocast():
            output = model(x)
            loss = criterion(output, y)
        # Scaled backprop
        scaler.scale(loss).backward()
        scaler.step(optimizer)
        scaler.update()
```

✅ **Expected:**
* Training speed increases (check epoch time).
* GPU memory usage drops (allows bigger batch sizes).
* Accuracy ≈ same as FP32.

---

## Step 4: TensorFlow AMP

```python
import tensorflow as tf
from tensorflow.keras import layers, models

# Enable mixed precision globally
policy = tf.keras.mixed_precision.Policy('mixed_float16')
tf.keras.mixed_precision.set_global_policy(policy)

# Model
model = models.Sequential([
    layers.Flatten(input_shape=(28,28)),
    layers.Dense(128, activation='relu'),
    layers.Dense(10)
])

model.compile(optimizer='adam',
              loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
              metrics=['accuracy'])

# Train
(x_train, y_train), _ = tf.keras.datasets.mnist.load_data()
model.fit(x_train, y_train, epochs=3, batch_size=512)
```

✅ **Expected:** Faster training than FP32 baseline, same accuracy.

---

## Step 5: Compare Results

Record results for **baseline FP32 vs AMP**:

| Metric | FP32 | AMP (Mixed Precision) |
| :--- | :--- | :--- |
| **Training Speed (s/epoch)** | ~baseline | ~1.5–2.5× faster |
| **GPU Memory (MB)** | ~baseline | ~30–50% lower |
| **Accuracy (%)** | ~baseline | ≈ same |

---

## Step 6: Experiment (Optional)
* Try **larger batch sizes** with AMP (should now fit in GPU memory).
* Try **different models** (ResNet, Transformer).
* Introduce **loss scaling manually** to see impact of underflow.

---

## Step 7: Wrap-Up
* Mixed Precision = **FP16 math + FP32 stability**.
* AMP automates casting & loss scaling.
* **Real gains:** speed ↑, memory ↓, accuracy ≈ same.
* Industry standard for training large AI models.

---

**✅ End of Lab —** You’ve trained models with mixed precision in PyTorch & TensorFlow, observed improvements, and compared against FP32.
