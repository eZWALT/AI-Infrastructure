# Day 266 Lab: Apply Differential Privacy in Training

## 🎯 Objective
* Understand how to apply **differential privacy (DP)** during training.
* Use **gradient clipping + noise injection (DP-SGD)**.
* Compare accuracy vs. privacy tradeoffs across TensorFlow Privacy and PyTorch Opacus.

---

## Step 1: Environment Setup
Install required libraries for TensorFlow Privacy and PyTorch Opacus:

```bash
# TensorFlow Privacy
pip install tensorflow tensorflow-privacy

# PyTorch Opacus
pip install torch torchvision opacus
```

Verify installation:

```python
import tensorflow as tf
import torch

print("TF:", tf.__version__, "Torch:", torch.__version__)
```

> **Expected:** Versions print without errors.

---

## Step 2: Load Dataset
We’ll use **MNIST** (simple, but enough to observe privacy-accuracy tradeoffs).

```python
# TensorFlow
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.mnist.load_data()
x_train, x_test = x_train / 255.0, x_test / 255.0

# PyTorch
from torchvision import datasets, transforms
import torch.utils.data

train_loader = torch.utils.data.DataLoader(
    datasets.MNIST('.', train=True, download=True, transform=transforms.ToTensor()),
    batch_size=256,
    shuffle=True
)
```

---

## Step 3: Baseline Model (No DP)

### TensorFlow Baseline
```python
import tensorflow as tf

model = tf.keras.Sequential([
    tf.keras.layers.Flatten(),
    tf.keras.layers.Dense(128, activation='relu'),
    tf.keras.layers.Dense(10)
])

model.compile(
    optimizer="adam",
    loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
    metrics=["accuracy"]
)

model.fit(x_train, y_train, epochs=3, batch_size=256)
```

### PyTorch Baseline
```python
import torch
import torch.nn as nn
import torch.optim as optim

class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(28 * 28, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        x = x.view(-1, 28 * 28)
        return self.fc2(torch.relu(self.fc1(x)))

model = Net()
optimizer = optim.SGD(model.parameters(), lr=0.01)
```

> **Expected:** Accuracy ~97–98% (without differential privacy).

---

## Step 4: Apply Differential Privacy

### TensorFlow (DP-SGD with `tensorflow-privacy`)
```python
import tensorflow_privacy as tfp

optimizer = tfp.DPKerasSGDOptimizer(
    l2_norm_clip=1.0,
    noise_multiplier=1.1,
    num_microbatches=256,
    learning_rate=0.15
)

model.compile(
    optimizer=optimizer,
    loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
    metrics=["accuracy"]
)

model.fit(x_train, y_train, epochs=3, batch_size=256)
```

> **Expected:** Accuracy ~90–95%. DP introduces noise into gradients, resulting in a slight drop in accuracy.

### PyTorch (Opacus for DP)
```python
import torch.nn as nn
from opacus import PrivacyEngine

privacy_engine = PrivacyEngine()
model, optimizer, train_loader = privacy_engine.make_private(
    module=model,
    optimizer=optimizer,
    data_loader=train_loader,
    noise_multiplier=1.0,
    max_grad_norm=1.0,
)

criterion = nn.CrossEntropyLoss()
for epoch in range(3):
    for x, y in train_loader:
        optimizer.zero_grad()
        loss = criterion(model(x), y)
        loss.backward()
        optimizer.step()
```

> **Expected:** Accuracy ~88–94%. DP noise protects privacy at the cost of slight model performance degradation.

---

## Step 5: Measure Privacy Guarantee ($arepsilon$)
TensorFlow Privacy provides accounting tools to compute the privacy budget ($arepsilon$).

```python
from tensorflow_privacy.privacy.analysis import compute_dp_sgd_privacy

compute_dp_sgd_privacy(
    n=60000,
    batch_size=256,
    epochs=3,
    noise_multiplier=1.1,
    delta=1e-5
)
```

> **Expected:** Output displays privacy budget details, e.g., $arepsilon pprox 3.5$ for $\delta = 10^{-5}$.

---

## Step 6: Compare FP vs. DP Results

| Metric | Baseline (No DP) | DP Training (DP-SGD) |
| :--- | :--- | :--- |
| **Accuracy (%)** | ~97–98 | ~88–95 |
| **Privacy Budget ($arepsilon$)** | $\infty$ (No privacy) | ~2–5 (Configurable) |
| **Training Speed** | Faster | Slightly slower |

---

## Step 7: Experiments (Optional)
* Change `noise_multiplier` (e.g., $0.5 ightarrow 2.0$) and compare accuracy impact.
* Try **different batch sizes** (affects the target $arepsilon$).
* Combine with **Federated Learning (TFF)** for multi-node privacy protection.

---

## ✅ Wrap-Up
* **Differential Privacy** ensures one individual's data point does not significantly alter the global outcome.
* Implemented via **gradient clipping + noise injection**.
* **Tradeoff:** Higher privacy protection $ightarrow$ lower accuracy/utility.
* Standard practice in **healthcare, finance, and mobile AI deployments**.
