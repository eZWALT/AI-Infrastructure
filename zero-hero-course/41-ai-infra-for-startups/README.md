# 🧪 Lab 287: Deploy AI Infra on Low Budget

## 🎯 Objective
* Deploy a budget-conscious AI infrastructure using open-source tools + cloud spot GPUs.
* Train + serve a simple ML model.
* Apply cost-saving strategies (spot/preemptible instances, OSS tools, resource quotas, auto-shutdown).

---

## Step 1: Choose Environment
We will use AWS or GCP with spot/preemptible GPU instances combined with an open-source (OSS) stack.

* **Compute:** Spot GPU (AWS `g4dn` / `p3`, GCP preemptible `NVIDIA T4`)
* **Storage:** AWS S3 / GCP GCS (cheaper object storage tier)
* **Orchestration:** Prefect (OSS)
* **Model Tracking:** MLflow (OSS)
* **Serving:** BentoML (OSS)

> **Note:** Lean stack = zero software licensing costs.

---

## Step 2: Launch a Spot GPU Instance

### AWS Example
```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxx \
  --count 1 \
  --instance-type g4dn.xlarge \
  --instance-market-options 'MarketType=spot' \
  --key-name my-key \
  --security-groups my-sg
```

### GCP Example
```bash
gcloud compute instances create gpu-trainer \
  --machine-type=n1-standard-4 \
  --accelerator=type=nvidia-tesla-t4,count=1 \
  --maintenance-policy=TERMINATE \
  --preemptible
```

✅ **Expected Result:** GPU VM running at **~70–80% lower cost** than on-demand instances.

---

## Step 3: Install Core Tools

Install the core machine learning and orchestration stack:

```bash
# ML / orchestration stack
pip install torch torchvision
pip install mlflow
pip install prefect
pip install bentoml
pip install prometheus-client
```

---

## Step 4: Train a Simple Model (MNIST)

Create and execute the training script:

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms

# Data loader
train_loader = torch.utils.data.DataLoader(
    datasets.MNIST('./data', train=True, download=True, transform=transforms.ToTensor()),
    batch_size=64, 
    shuffle=True
)

# Network definition
class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(28 * 28, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        return self.fc2(torch.relu(self.fc1(x.view(-1, 28 * 28))))

model = Net()
opt = optim.Adam(params=model.parameters())

# Training loop
for epoch in range(1):
    for data, target in train_loader:
        opt.zero_grad()
        loss = nn.CrossEntropyLoss()(model(data), target)
        loss.backward()
        opt.step()
```

✅ **Expected Result:** Model trains in under 1 minute and reaches **~97% accuracy**.

---

## Step 5: Log Model with MLflow

Log metrics, parameters, and model artifacts using MLflow:

```python
import mlflow.pytorch

mlflow.set_tracking_uri("file:./mlruns")

with mlflow.start_run():
    mlflow.log_param("epochs", 1)
    mlflow.log_metric("loss", loss.item())
    mlflow.pytorch.log_model(model, "mnist-model")
```

✅ **Expected Result:** Model artifacts and loss metrics logged into local MLflow run folder (`./mlruns`).

---

## Step 6: Package & Serve with BentoML

Save and define a BentoML service for model inference:

```python
import bentoml
import torch

# Save model to BentoML store
bentoml.pytorch.save_model("mnist_classifier", model)

# Define service
@bentoml.service(models=["mnist_classifier:latest"])
class MnistService:
    @bentoml.api
    def predict(self, arr: torch.Tensor):
        return model(arr).argmax(dim=1).tolist()
```

Run the server:

```bash
bentoml serve service:MnistService
```

✅ **Expected Result:** Local REST API serving MNIST predictions on designated port.

---

## Step 7: Add Monitoring (Prometheus)

Expose Prometheus metrics for production monitoring:

```python
from prometheus_client import Counter, start_http_server
import bentoml
import torch

pred_counter = Counter("predictions_total", "Number of predictions served")

# Start Prometheus metrics endpoint
start_http_server(8000)

@bentoml.service(models=["mnist_classifier:latest"])
class MnistService:
    @bentoml.api
    def predict(self, arr: torch.Tensor):
        pred_counter.inc()
        return model(arr).argmax(dim=1).tolist()
```

✅ **Expected Result:** Prometheus metrics available and scrapable at `http://localhost:8000/metrics`.

---

## Step 8: Apply Budget Guardrails

1. **Spot/Preemptible GPUs only:** Never run on-demand for training workloads.
2. **Auto-shutdown cron:** Schedule automatic termination to avoid idle costs.
   ```bash
   sudo shutdown -h +360  # Auto-shutdown after 6 hours
   ```
3. **Experiment hygiene:** Run short experiments and regularly prune old checkpoints.
4. **Cold Tier Storage:** Move raw datasets and archived models to AWS S3 Glacier / GCP Coldline storage.

---

## Step 9: Test & Validate

* Query REST API endpoint with test sample tensor inputs.
* Monitor real-time GPU memory and usage via `nvidia-smi`.
* Verify Prometheus metrics endpoint updates request counter (`predictions_total`).
* Review cloud cost report: Spot GPU vs. On-Demand (~3× savings).

---

## ✅ Wrap-Up
* Deployed lean AI infrastructure on spot/preemptible GPU instances.
* Leveraged 100% open-source stack (**MLflow, Prefect, BentoML, Prometheus**).
* Implemented strict budget guardrails (auto-shutdown, checkpoint pruning, object storage tiers).
* Demonstrated how startups can maintain production-ready AI infra for **<$200/month**.
