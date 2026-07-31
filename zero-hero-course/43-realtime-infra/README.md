# 🧪 Lab 301: Build Real-Time Fraud Detection Pipeline

## 🎯 Objective
* Build a **fraud detection pipeline** that scores transactions in real time.
* Use **Kafka (streaming)** + **Python model API (FastAPI or Triton)**.
* Achieve **sub-100ms decision latency**.

---

## Step 1: Set Up Environment

### Requirements:
* **Docker + Docker Compose** (for Kafka, Zookeeper)
* **Python 3.9+**
* **Libraries:** `fastapi`, `uvicorn`, `scikit-learn`, `pandas`, `confluent-kafka`, `requests`

```bash
pip install fastapi uvicorn scikit-learn pandas confluent-kafka requests
```

✅ **Expected Result:** Python environment and required dependencies successfully installed.

---

## Step 2: Start Kafka Locally

Create a `docker-compose.yml` file:

```yaml
version: '3'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```

Start the service container:

```bash
docker-compose up -d
```

✅ **Expected Result:** Kafka cluster up and running at `localhost:9092`.

---

## Step 3: Train a Simple Fraud Detection Model

Create and run `train_model.py`:

```python
from sklearn.ensemble import RandomForestClassifier
import pandas as pd
import joblib

# Example: synthetic or Kaggle fraud dataset
# data = pd.read_csv("creditcard.csv") 
# X, y = data.drop("Class", axis=1), data["Class"]

# Synthetic data placeholder for standalone run
import numpy as np
X = np.random.rand(100, 3)
y = np.random.choice([0, 1], size=100, p=[0.9, 0.1])

model = RandomForestClassifier(n_estimators=50)
model.fit(X, y)
joblib.dump(model, "fraud_model.pkl")
```

✅ **Expected Result:** Trained model saved locally as `fraud_model.pkl`.

---

## Step 4: Build Real-Time Fraud API

Create `fraud_api.py`:

```python
from fastapi import FastAPI
import joblib

app = FastAPI()
model = joblib.load("fraud_model.pkl")

@app.post("/predict")
def predict(features: list[float]):
    prob = model.predict_proba([features])[0][1]
    return {"fraud_score": float(prob), "fraud": bool(prob > 0.7)}
```

Run the server using Uvicorn:

```bash
uvicorn fraud_api:app --host 0.0.0.0 --port 8000
```

✅ **Expected Result:** Real-time fraud detection API active at `http://localhost:8000/predict`.

---

## Step 5: Kafka Producer (Transaction Stream)

Create `producer.py`:

```python
from confluent_kafka import Producer
import json
import random
import time

p = Producer({'bootstrap.servers': 'localhost:9092'})

while True:
    txn = {
        "amount": random.randint(1, 1000), 
        "location": random.choice(["US", "EU", "ASIA"]), 
        "device": random.randint(1000, 9999)
    }
    p.produce("transactions", json.dumps(txn).encode("utf-8"))
    p.flush()
    time.sleep(0.5)
```

✅ **Expected Result:** Live transactions continuously streamed to Kafka topic `transactions`.

---

## Step 6: Kafka Consumer + API Scoring

Create `consumer.py`:

```python
from confluent_kafka import Consumer
import requests
import json

c = Consumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'fraud-detector',
    'auto.offset.reset': 'earliest'
})
c.subscribe(['transactions'])

while True:
    msg = c.poll(1.0)
    if msg is None:
        continue
    
    txn = json.loads(msg.value().decode("utf-8"))
    
    # Convert transaction dictionary into feature vector
    features = [float(txn["amount"]), float(len(txn["location"])), float(txn["device"])]
    
    r = requests.post("http://localhost:8000/predict", json=features)
    print("Transaction:", txn, "| Prediction:", r.json())
```

✅ **Expected Result:** Streaming Kafka messages consumed and scored in real-time.

---

## Step 7: Add Monitoring (Latency Metrics)

Enhance consumer logic to record round-trip latency:

```python
import time

start = time.time()
r = requests.post("http://localhost:8000/predict", json=features)
latency = (time.time() - start) * 1000

print(f"Latency: {latency:.2f} ms | Prediction: {r.json()}")
```

✅ **Expected Result:** Measured average decision latency between **30–80 ms** locally.

---

## Step 8: Optional Enhancements

1. **Prometheus Metrics:** Add latency histograms and total fraud prediction counters.
2. **Triton Inference Server:** Deploy model via Triton for GPU-accelerated concurrent execution.
3. **Model Optimization:** Convert models to **TensorRT / ONNX Runtime** to minimize inference latency.
4. **Horizontal Scaling:** Scale Kafka consumer groups across multiple workers for high-throughput stream processing.

---

## ✅ Wrap-Up
* Built an end-to-end **real-time fraud detection pipeline** using Kafka & FastAPI.
* Trained and served a machine learning model via REST endpoint.
* Streamed transaction feeds into Kafka topics and evaluated them with **sub-100ms latency**.
* Configured real-time performance instrumentation and monitoring hooks.
