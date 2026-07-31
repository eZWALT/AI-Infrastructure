# Lab 28 – Containerize a PyTorch Model

## 🎯 Goal

Package a simple PyTorch model inside a Docker container, expose it through a FastAPI REST API, and run image predictions.

**Estimated Time:** 60–90 minutes  
**Cost:** Free (local Docker) or minimal (cloud VM with Docker)

---

# Learning Outcomes

By the end of this lab, you will be able to:

- Train and save a PyTorch model
- Build a FastAPI inference service
- Containerize an AI application with Docker
- Run reproducible inference anywhere
- (Optional) Enable GPU acceleration using NVIDIA Docker

---

# Prerequisites

Before starting, ensure you have:

- Docker installed

```bash
docker --version
```

- Python 3.10+
- PyTorch installed locally (only needed to generate the model)
- *(Optional)* NVIDIA Container Toolkit if running on a GPU

---

# Step 1 — Train and Save a Model

Create a file named **`train_model.py`**.

```python
import torch
import torch.nn as nn
from torchvision import models

# Load pretrained ResNet18
model = models.resnet18(pretrained=True)

# Replace classification head
model.fc = nn.Linear(model.fc.in_features, 10)

# Fake training for demonstration
torch.save(model, "model.pt")

print("✅ Model saved as model.pt")
```

Run the script once:

```bash
python train_model.py
```

Expected output:

```
✅ Model saved as model.pt
```

A file named **`model.pt`** should now exist.

---

# Step 2 — Create the FastAPI Inference Service

Create **`app.py`**.

```python
from fastapi import FastAPI, UploadFile
from PIL import Image
import torch
import torchvision.transforms as T

app = FastAPI()

# Load model
model = torch.load("model.pt", map_location="cpu")
model.eval()

transform = T.Compose([
    T.Resize((224, 224)),
    T.ToTensor()
])

@app.post("/predict")
async def predict(file: UploadFile):
    img = Image.open(file.file).convert("RGB")
    x = transform(img).unsqueeze(0)

    with torch.no_grad():
        output = model(x)

    prediction = output.argmax().item()

    return {
        "prediction": int(prediction)
    }
```

This creates a minimal REST API with a single endpoint:

```
POST /predict
```

---

# Step 3 — Define Python Dependencies

Create **`requirements.txt`**.

```text
fastapi==0.103.0
uvicorn==0.23.2
torch==2.1.0
torchvision==0.16.0
pillow==10.0.0
```

---

# Step 4 — Create the Dockerfile

Create a file named **`Dockerfile`**.

```dockerfile
FROM python:3.10-slim

# Working directory
WORKDIR /app

# Install system packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    libgl1 \
    libglib2.0-0 && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY model.pt app.py ./

# Expose API port
EXPOSE 8000

# Launch FastAPI
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

# Step 5 — Build the Docker Image

From the project directory, build the container.

```bash
docker build -t pytorch-api:latest .
```

Verify that the image exists:

```bash
docker images
```

You should see something similar to:

```
REPOSITORY      TAG       IMAGE ID
pytorch-api     latest    ...
```

---

# Step 6 — Run the Container

Start the API:

```bash
docker run -it -p 8000:8000 pytorch-api:latest
```

The FastAPI server will now be available at:

```
http://localhost:8000
```

Interactive API documentation:

```
http://localhost:8000/docs
```

---

# Step 7 — Test the API

## Option A — Swagger UI

Open:

```
http://localhost:8000/docs
```

1. Open the **`POST /predict`** endpoint.
2. Click **Try it out**.
3. Upload an image (e.g., `cat.jpg`).
4. Click **Execute**.

Example response:

```json
{
  "prediction": 3
}
```

---

## Option B — Using curl

```bash
curl -X POST http://localhost:8000/predict \
    -F "file=@cat.jpg"
```

Example response:

```json
{
  "prediction": 3
}
```

---

# Optional — GPU Support

If you have the NVIDIA Container Toolkit installed, you can run the container using GPU acceleration.

```bash
docker run --gpus all -it -p 8000:8000 pytorch-api:latest
```

Verify PyTorch detects the GPU:

```python
import torch

print(torch.cuda.is_available())
```

Expected output:

```
True
```

---

# Project Structure

```
.
├── app.py
├── train_model.py
├── model.pt
├── requirements.txt
└── Dockerfile
```

---

# Summary

In this lab you:

- ✅ Created a PyTorch model
- ✅ Saved it for inference
- ✅ Built a FastAPI prediction service
- ✅ Packaged everything inside a Docker container
- ✅ Exposed a REST API
- ✅ (Optional) Enabled GPU inference

This workflow is the foundation for deploying deep learning models in production environments.