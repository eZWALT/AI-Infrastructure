
# Lab 63 — Track Experiments with MLflow

## 🧪 Goal
Learn how to use **MLflow Tracking** to manage the full lifecycle of machine learning experiments:
- Track parameters and hyperparameters
- Log metrics during training
- Save artifacts (plots, summaries, ONNX models)
- Store and reload trained models
- Compare multiple runs
- Organize experiments with tags

**Time:** ~2–3 hours

**Tools:** Python, PyTorch, MLflow, torchvision, matplotlib, CUDA (optional)

---

# 1️⃣ Install and Launch MLflow

```bash
pip install "mlflow[extras]" matplotlib onnx
```

Launch the UI:

```bash
mlflow ui
```

Open:

```
http://127.0.0.1:5000
```

---

# 2️⃣ Verify Environment

```python
import mlflow
import torch

print("MLflow:", mlflow.__version__)
print("Torch:", torch.__version__)
print("CUDA:", torch.cuda.is_available())
```

---

# 3️⃣ Create an Experiment

```python
import mlflow

mlflow.set_experiment("mnist-mlflow-lab")
```

---

# 4️⃣ Load MNIST

```python
import torchvision
import torchvision.transforms as T
from torch.utils.data import DataLoader

transform = T.ToTensor()

trainset = torchvision.datasets.MNIST(
    "./data", train=True, download=True, transform=transform
)

testset = torchvision.datasets.MNIST(
    "./data", train=False, download=True, transform=transform
)

trainloader = DataLoader(trainset,batch_size=64,shuffle=True)
testloader = DataLoader(testset,batch_size=256)
```

---

# 5️⃣ Define a Small Model

```python
import torch.nn as nn
import torch.nn.functional as F

class Net(nn.Module):

    def __init__(self):
        super().__init__()

        self.fc1=nn.Linear(784,128)
        self.fc2=nn.Linear(128,64)
        self.fc3=nn.Linear(64,10)

    def forward(self,x):

        x=x.view(-1,784)
        x=F.relu(self.fc1(x))
        x=F.relu(self.fc2(x))

        return self.fc3(x)
```

---

# 6️⃣ Train with MLflow

```python
import time
import torch
import torch.optim as optim
import mlflow.pytorch

device=torch.device("cuda" if torch.cuda.is_available() else "cpu")

model=Net().to(device)

criterion=nn.CrossEntropyLoss()
optimizer=optim.Adam(model.parameters(),lr=1e-3)

with mlflow.start_run(run_name="baseline"):

    mlflow.log_params({
        "optimizer":"Adam",
        "lr":1e-3,
        "batch_size":64,
        "epochs":2
    })

    mlflow.set_tags({
        "framework":"PyTorch",
        "dataset":"MNIST",
        "course":"AI Engineering"
    })

    start=time.time()

    for epoch in range(2):

        model.train()
        running=0

        for images,labels in trainloader:

            images=images.to(device)
            labels=labels.to(device)

            optimizer.zero_grad()

            outputs=model(images)

            loss=criterion(outputs,labels)

            loss.backward()

            optimizer.step()

            running+=loss.item()

        avg=running/len(trainloader)

        mlflow.log_metric("train_loss",avg,step=epoch)

        print(epoch,avg)

    training_time=time.time()-start

    mlflow.log_metric("training_seconds",training_time)

    mlflow.pytorch.log_model(model,"model")
```

---

# 7️⃣ Evaluate and Log Accuracy

```python
correct=0
total=0

model.eval()

with torch.no_grad():

    for images,labels in testloader:

        images=images.to(device)
        labels=labels.to(device)

        pred=model(images).argmax(1)

        total+=labels.size(0)
        correct+=(pred==labels).sum().item()

accuracy=100*correct/total

mlflow.log_metric("accuracy",accuracy)

print(accuracy)
```

---

# 8️⃣ Log System Information

```python
import platform

mlflow.log_params({
    "python":platform.python_version(),
    "torch":torch.__version__,
    "cuda":torch.cuda.is_available()
})

if torch.cuda.is_available():
    mlflow.log_param(
        "gpu",
        torch.cuda.get_device_name(0)
    )
```

---

# 9️⃣ Log Model Statistics

```python
params=sum(p.numel() for p in model.parameters())

mlflow.log_metric(
    "num_parameters",
    params
)
```

---

# 🔟 Log Artifacts

```python
import matplotlib.pyplot as plt

losses=[0.9,0.6,0.3]

plt.plot(losses)
plt.xlabel("Epoch")
plt.ylabel("Loss")

plt.savefig("loss_curve.png")

mlflow.log_artifact("loss_curve.png")
```

Create a model summary:

```python
with open("summary.txt","w") as f:
    f.write(str(model))

mlflow.log_artifact("summary.txt")
```

---

# 1️⃣1️⃣ Export to ONNX

```python
dummy=torch.randn(1,1,28,28).to(device)

torch.onnx.export(
    model,
    dummy,
    "mnist.onnx"
)

mlflow.log_artifact("mnist.onnx")
```

---

# 1️⃣2️⃣ Hyperparameter Sweep

```python
learning_rates=[1e-2,1e-3,1e-4]

for lr in learning_rates:

    with mlflow.start_run(run_name=f"lr={lr}"):

        mlflow.log_param("lr",lr)

        # train model

        # log metrics
```

Compare runs directly in the MLflow UI.

---

# 1️⃣3️⃣ Nested Runs

```python
with mlflow.start_run(run_name="optimizer-study"):

    for lr in [1e-2,1e-3]:

        with mlflow.start_run(
            nested=True,
            run_name=f"lr={lr}"
        ):

            mlflow.log_param("lr",lr)
```

---

# 1️⃣4️⃣ Load a Saved Model

```python
run_id="<RUN_ID>"

uri=f"runs:/{run_id}/model"

loaded=mlflow.pytorch.load_model(uri)
```

---

# 1️⃣5️⃣ Explore the UI

Inspect:

- Parameters
- Metrics
- Tags
- Artifacts
- Models
- Compare Runs

---

# 1️⃣6️⃣ Benchmark Table

| Run | LR | Accuracy | Loss | Time | Params |
|-----|---:|---------:|-----:|-----:|-------:|
|Baseline||||||
|LR=0.01||||||
|LR=0.001||||||
|LR=0.0001||||||

---

# 1️⃣7️⃣ Cleanup

```bash
rm -rf mlruns
```

Stop the server with **CTRL+C**.

---

# 🎯 Learning Outcomes

- Create MLflow experiments
- Track parameters, metrics and tags
- Log PyTorch models
- Save artifacts (plots, summaries, ONNX)
- Compare multiple runs
- Use nested runs
- Reload trained models
- Build reproducible ML experimentation workflows
