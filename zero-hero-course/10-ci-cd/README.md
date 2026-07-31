# 🧪 Lab – CI/CD Pipeline for Model Deployment (Enhanced)

## Goal

Create a CI/CD pipeline that:

- Trains a PyTorch model
- Runs basic code quality checks
- Containerizes the application with Docker
- Pushes the image to Docker Hub (or GHCR)
- Deploys it via GitHub Actions (or GitLab CI/Jenkins)
- (Optional) Automatically updates a Kubernetes deployment

**Estimated Time:** 2–3 hours

**Tools**

- GitHub Actions (or GitLab CI / Jenkins)
- Docker & Docker Hub (or GitHub Container Registry)
- Kubernetes (Kind, Minikube, k3s, or managed cluster)
- PyTorch
- Flask or FastAPI

---

# 1️⃣ Prepare Repository

Create a new repository.

Suggested structure:

```text
ml-cicd-lab/
│
├── train.py
├── app.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── README.md
├── tests/
│   └── test_api.py
└── .github/
    └── workflows/
        └── cicd.yml
```

Add:

- `train.py` → trains & saves the model
- `app.py` → inference API
- `requirements.txt`
- `Dockerfile`
- `.github/workflows/cicd.yml`
- `.dockerignore`
- `README.md`

✅ Repository ready for ML + deployment.

---

# 2️⃣ Train a Simple Model (`train.py`)

Train a simple MNIST classifier using PyTorch.

```python
import torch, torchvision, torchvision.transforms as transforms
import torch.nn as nn, torch.optim as optim

trainset = torchvision.datasets.MNIST(
    "./data",
    train=True,
    download=True,
    transform=transforms.ToTensor()
)

trainloader = torch.utils.data.DataLoader(
    trainset,
    batch_size=64,
    shuffle=True
)

class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(28*28,128)
        self.fc2 = nn.Linear(128,64)
        self.fc3 = nn.Linear(64,10)

    def forward(self,x):
        x=x.view(-1,28*28)
        x=torch.relu(self.fc1(x))
        x=torch.relu(self.fc2(x))
        return self.fc3(x)

model = Net()

criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

for epoch in range(1):
    for images, labels in trainloader:
        optimizer.zero_grad()
        outputs=model(images)
        loss=criterion(outputs, labels)
        loss.backward()
        optimizer.step()

torch.save(model.state_dict(),"model.pth")

print("✅ Model trained and saved")
```

### Verify locally

```bash
python train.py

ls model.pth
```

✅ Model successfully generated.

---

# 3️⃣ Create an Inference API (`app.py`)

Serve the trained model with Flask (or FastAPI).

```python
from flask import Flask, request, jsonify
import torch
import torch.nn as nn

app = Flask(__name__)

class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1=nn.Linear(28*28,128)
        self.fc2=nn.Linear(128,64)
        self.fc3=nn.Linear(64,10)

    def forward(self,x):
        x=x.view(-1,28*28)
        return self.fc3(
            torch.relu(
                self.fc2(
                    torch.relu(self.fc1(x))
                )
            )
        )

model = Net()
model.load_state_dict(torch.load("model.pth"))
model.eval()

@app.route("/predict", methods=["POST"])
def predict():

    data = torch.tensor(request.json["input"])

    output = model(data.float())

    _, predicted = torch.max(output,1)

    return jsonify({
        "prediction": predicted.item()
    })

app.run(host="0.0.0.0", port=5000)
```

### Test locally

```bash
python app.py
```

Then send a request using Postman or:

```bash
curl -X POST http://localhost:5000/predict \
-H "Content-Type: application/json" \
-d '{"input":[[0,0,...]]}'
```

✅ API returns a prediction.

---

# 4️⃣ Create `requirements.txt`

Include at least:

```text
torch
torchvision
flask
gunicorn
pytest
```

Install locally.

```bash
pip install -r requirements.txt
```

---

# 5️⃣ Create the Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python","app.py"]
```

### Build

```bash
docker build -t myrepo/ai-model:latest .
```

### Run

```bash
docker run -p 5000:5000 myrepo/ai-model:latest
```

Verify `/predict` still works.

✅ Containerized inference service.

---

# 6️⃣ (Optional) Add a Simple API Test

Create:

```text
tests/test_api.py
```

Write a basic test that verifies:

- `/predict` returns HTTP 200
- Response contains `"prediction"`

Run:

```bash
pytest
```

✅ Basic automated testing.

---

# 7️⃣ Configure GitHub Actions

Create:

```
.github/workflows/cicd.yml
```

Pipeline stages:

1. Checkout repository
2. Setup Python
3. Install dependencies
4. (Optional) Run tests
5. Train model
6. Build Docker image
7. Login to Docker Hub
8. Push Docker image

Example:

```yaml
name: ML CI/CD Pipeline

on:
  push:

jobs:

  build-deploy:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v3

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest

      - name: Train model
        run: python train.py

      - name: Build Docker image
        run: docker build -t myrepo/ai-model:latest .

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASS }}

      - name: Push Docker image
        run: docker push myrepo/ai-model:latest
```

✅ Continuous Integration configured.

---

# 8️⃣ Deploy to Kubernetes (Optional)

Create a Deployment and Service manually.

Deploy once using:

```bash
kubectl apply -f deployment.yaml

kubectl apply -f service.yaml
```

Then extend the workflow with:

```yaml
- name: Deploy to Kubernetes
  run: |
    kubectl set image deployment/ai-app \
      ai-app=myrepo/ai-model:latest
```

✅ Automatic deployment after a successful build.

---

# 9️⃣ Validate the Pipeline

Push a commit.

Verify that GitHub Actions:

- Checks out the code
- Installs dependencies
- Runs tests
- Trains the model
- Builds the Docker image
- Pushes the image
- (Optional) Updates Kubernetes

Open the GitHub Actions tab and inspect every stage.

---

# 🔟 Monitor the Deployment

Verify:

```bash
kubectl get pods

kubectl get deployments

kubectl get services
```

Port-forward if necessary.

```bash
kubectl port-forward deployment/ai-app 5000:5000
```

Call:

```
POST /predict
```

Confirm the deployed service works.

---

# 1️⃣1️⃣ Rollback

Keep previous Docker image tags.

If a deployment fails:

```bash
kubectl rollout undo deployment ai-app
```

Verify:

```bash
kubectl rollout status deployment ai-app
```

✅ Safe rollback completed.

---

# 🎯 Learning Outcomes

By completing this lab you will be able to:

- Build a complete ML CI/CD pipeline
- Train and package a PyTorch model
- Containerize an inference service with Docker
- Configure GitHub Actions for automated builds
- Push Docker images to a registry
- Deploy updates to Kubernetes
- Perform basic API testing
- Monitor pipeline executions
- Roll back failed deployments safely
