# Day 196 Lab – Deploy a BERT Model with FastAPI

## Learning Goals

* Load a pretrained **BERT model** for text classification
* Expose it as a **FastAPI endpoint**
* Send queries and receive predictions in JSON format
* Understand how to containerize and deploy in production

---

## 0) Prerequisites

Make sure you have the following ready:

* **Python 3.9+**

### Install Dependencies

Run these commands in your terminal:

```bash
mkdir bert-fastapi-lab && cd bert-fastapi-lab
python -m venv .venv

# Activate virtual environment
# On macOS/Linux:
source .venv/bin/activate
# On Windows:
# .venv\Scripts\activate

pip install torch transformers fastapi uvicorn
```

---

## 1) Load Pretrained BERT Model

We’ll use `distilbert-base-uncased-finetuned-sst-2-english` for sentiment classification.

Create `model.py`:

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

MODEL_NAME = "distilbert-base-uncased-finetuned-sst-2-english"

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
model = AutoModelForSequenceClassification.from_pretrained(MODEL_NAME)

labels = ["negative", "positive"]

def predict(text: str):
    inputs = tokenizer(text, return_tensors="pt", truncation=True, padding=True)
    with torch.no_grad():
        outputs = model(**inputs)
        probs = torch.nn.functional.softmax(outputs.logits, dim=-1)[0]
    return {labels[i]: float(probs[i]) for i in range(len(labels))}
```

---

## 2) Build FastAPI App

Create `server.py`:

```python
from fastapi import FastAPI
from pydantic import BaseModel
from model import predict

app = FastAPI()

class Query(BaseModel):
    text: str

@app.post("/classify/")
def classify(query: Query):
    result = predict(query.text)
    return {"input": query.text, "prediction": result}
```

---

## 3) Run the API Server

```bash
uvicorn server:app --reload --port 8000
```

> **API Endpoint:** `http://127.0.0.1:8000`  
> **Interactive Docs:** `http://127.0.0.1:8000/docs` (auto-generated Swagger UI)

---

## 4) Test the Endpoint

In a separate terminal tab, send a request using `curl`:

```bash
curl -X POST "http://127.0.0.1:8000/classify/"   -H "Content-Type: application/json"   -d '{"text": "I really enjoyed this movie"}'
```

### Expected Output (Sample)

```json
{
  "input": "I really enjoyed this movie",
  "prediction": {
    "negative": 0.02,
    "positive": 0.98
  }
}
```

---

## 5) Optional – Add Batch Inference

Modify `predict_batch()` in `model.py` to accept a list of texts:

```python
def predict_batch(texts: list[str]):
    inputs = tokenizer(texts, return_tensors="pt", truncation=True, padding=True)
    with torch.no_grad():
        outputs = model(**inputs)
        probs = torch.nn.functional.softmax(outputs.logits, dim=-1)
    return [{labels[i]: float(p[i]) for i in range(len(labels))} for p in probs]
```

Add a `/batch` endpoint in `server.py`:

```python
from model import predict_batch

class BatchQuery(BaseModel):
    texts: list[str]

@app.post("/batch/")
def classify_batch(query: BatchQuery):
    results = predict_batch(query.texts)
    return {"predictions": results}
```

---

## 6) Optional – Dockerize for Deployment

Create a `Dockerfile` in your root directory:

```dockerfile
FROM python:3.10-slim

WORKDIR /app
COPY . .

RUN pip install --no-cache-dir torch transformers fastapi uvicorn

CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Build & Run Docker Container

```bash
docker build -t bert-fastapi .
docker run -p 8000:8000 bert-fastapi
```

---

## 7) Stretch Goals

* Add a `/health` endpoint for monitoring.
* Connect with **Prometheus / Grafana** for latency and request metrics.
* Deploy on **Kubernetes / Ray Serve** for horizontal scaling.
* Extend to multilingual models (e.g., `bert-base-multilingual-cased`).

> ✅ **Outcome:** You deployed a BERT text classification API using FastAPI, complete with options for batch processing, monitoring, and containerization.
