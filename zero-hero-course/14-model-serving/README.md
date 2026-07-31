# 🧪 Lab 98 – Serve an Image Classifier with FastAPI

## 🎯 Goal

Deploy a pretrained image classification model (ResNet18 by default) behind a FastAPI REST API. You'll run it locally, send images for prediction, and optionally containerize it with Docker or deploy it on Kubernetes.

**Estimated Time:** 90–120 minutes

**Tools**

- Python 3.10+
- FastAPI
- Uvicorn
- PyTorch
- Torchvision
- Pillow
- Docker (optional)

---

# Step 0 – Prerequisites

- Python 3.10+
- `pip` or `conda`
- `uvicorn`
- GPU optional (CUDA is used automatically if available)

---

# Step 1 – Project Setup

Create the following project structure:

```text
lab98_fastapi_classifier/
├── app/
│   └── main.py
├── scripts/
│   └── client.py
├── requirements.txt
├── Dockerfile
└── README.md
```

---

# Step 2 – Install Dependencies

Create `requirements.txt`

```text
fastapi==0.112.2
uvicorn[standard]==0.30.6
pillow==10.4.0
torch==2.3.1
torchvision==0.18.1
pydantic==2.8.2
python-multipart==0.0.9
```

Install:

```bash
pip install -r requirements.txt
```

---

# Step 3 – Build the FastAPI Application

Create:

```text
app/main.py
```

Implement:

- Load a pretrained ResNet18 model
- Detect CPU/GPU automatically
- Load ImageNet labels
- Preprocess uploaded images
- Perform inference
- Return the Top-K predictions

The API should expose:

```
GET  /healthz

POST /predict
```

---

# Step 4 – Download ImageNet Labels

```bash
wget https://raw.githubusercontent.com/pytorch/hub/master/imagenet_classes.txt
```

---

# Step 5 – Run the API

```bash
uvicorn app.main:app \
    --host 0.0.0.0 \
    --port 8000
```

Verify the health endpoint:

```bash
curl http://localhost:8000/healthz
```

You should receive something similar to:

```json
{
  "status": "ok",
  "device": "cpu"
}
```

(or `"cuda"` if using a GPU)

---

# Step 6 – Build a Client

Create:

```text
scripts/client.py
```

The client should:

- Read an image from disk
- Upload it to `/predict`
- Print the JSON response

Run:

```bash
python scripts/client.py path/to/image.jpg
```

Verify that the API returns the predicted ImageNet classes.

---

# Step 7 – Test the API

Experiment with:

- Different images
- PNG vs JPEG
- Various `top_k` values
- Invalid files (TXT, PDF, etc.)
- Large images

Observe how the API behaves.

---

# Step 8 – Dockerize the Application (Optional)

Create:

```text
Dockerfile
```

Use a lightweight Python image.

Build:

```bash
docker build -t fastapi-cls .
```

Run:

```bash
docker run -p 8000:8000 fastapi-cls
```

Verify that `/healthz` and `/predict` still work.

---

# Step 9 – (Optional) Deploy to Kubernetes

Create:

- Deployment
- Service

Deploy them manually:

```bash
kubectl apply -f deployment.yaml

kubectl apply -f service.yaml
```

Port-forward:

```bash
kubectl port-forward svc/fastapi-cls 8000:8000
```

Verify the API works exactly as before.

---

# Step 10 – Explore the OpenAPI Documentation

FastAPI automatically generates interactive API documentation.

Visit:

```
http://localhost:8000/docs
```

Also inspect:

```
http://localhost:8000/redoc
```

Try invoking the API directly from the browser.

---

# 🎯 Learning Outcomes

By completing this lab you will be able to:

- Build a FastAPI inference server
- Serve pretrained PyTorch models
- Handle image uploads
- Perform image preprocessing
- Return structured prediction results
- Test REST endpoints using Python clients
- Explore automatically generated OpenAPI documentation
- Containerize inference services with Docker
- (Optional) Deploy the API to Kubernetes
