# Day 168 Lab — Build a Streaming Pipeline with Kafka

## Learning Goals

* Stand up a single-broker Kafka cluster on your laptop with a visual UI
* Publish synthetic transactions in real time
* Build a stream processor that validates, aggregates, and emits features
* Observe and troubleshoot using UI and logs
* *(Optional)* Add a Dead-Letter Queue (DLQ) and basic data quality checks

---

## 0) Prerequisites

Make sure the following are installed and running:

* **Docker Desktop** (running)
* **Python 3.9+** (`python --version`)
* Terminal + Code Editor

---

## 1) Project Scaffold

Run the following commands in your terminal to set up the directory layout:

```bash
mkdir kafka-streaming-lab && cd kafka-streaming-lab
mkdir app
```

### Directory Structure

```text
kafka-streaming-lab/
├── docker-compose.yml
└── app/
    ├── requirements.txt
    ├── producer.py
    ├── processor.py
    └── consumer.py
```

---

## 2) Docker Compose (Kafka + Kafka UI)

Create `docker-compose.yml` in the root directory:

```yaml
version: "3.8"

services:
  kafka:
    image: bitnami/kafka:3.7
    container_name: kafka
    ports:
      - "9092:9092"    # external for host apps
      - "29092:29092"  # internal for containers
    environment:
      - KAFKA_ENABLE_KRAFT=yes
      - KAFKA_CFG_NODE_ID=1
      - KAFKA_CFG_PROCESS_ROLES=broker,controller
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT
      - KAFKA_CFG_LISTENERS=INTERNAL://:29092,EXTERNAL://:9092,CONTROLLER://:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=INTERNAL://kafka:29092,EXTERNAL://localhost:9092
      - KAFKA_CFG_INTER_BROKER_LISTENER_NAME=INTERNAL
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true
    volumes:
      - kafka_data:/bitnami/kafka

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      - KAFKA_CLUSTERS_0_NAME=local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:29092
    depends_on:
      - kafka

volumes:
  kafka_data:
```

### Start the Stack

```bash
docker compose up -d
```

> **Kafka UI:** Open [http://localhost:8080](http://localhost:8080) to view the `local` cluster.

---

## 3) Python Dependencies

Create `app/requirements.txt`:

```text
kafka-python==2.0.2
faker==25.9.2
pydantic==2.8.2
python-dateutil==2.9.0.post0
```

### Setup Virtual Environment and Install Dependencies

```bash
cd app
python -m venv .venv

# Activate venv:
# On macOS/Linux:
source .venv/bin/activate 
# On Windows:
# .venv\Scripts\activate

pip install -r requirements.txt
```

---

## 4) Create Topics (Optional)

Auto-creation is enabled, but you can explicitly create topics via the **Kafka UI → Topics → Create**:

* `transactions` (partitions: 3)
* `features` (partitions: 3)
* `dlq-transactions` *(optional)* (partitions: 1)

---

## 5) Producer — Real-Time Synthetic Events

Create `app/producer.py`:

```python
import json
import os
import random
import time
from datetime import datetime, timezone
from faker import Faker
from kafka import KafkaProducer

BOOTSTRAP = os.getenv("BOOTSTRAP", "localhost:9092")
TOPIC = os.getenv("TOPIC", "transactions")
EPS = float(os.getenv("EVENTS_PER_SECOND", "5"))

fake = Faker()
producer = KafkaProducer(
    bootstrap_servers=BOOTSTRAP,
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
    key_serializer=lambda k: str(k).encode("utf-8") if k is not None else None,
    linger_ms=10
)

EVENT_TYPES = ["view", "add_to_cart", "purchase"]
DEVICES = ["web", "ios", "android"]
COUNTRIES = ["US", "IN", "BR", "DE", "GB", "CA"]

def make_event():
    user_id = random.randint(1, 500)
    etype = random.choices(EVENT_TYPES, weights=[0.7, 0.2, 0.1])[0]
    amount = round(random.uniform(5, 200), 2) if etype == "purchase" else 0.0
    return {
        "event_time": datetime.now(timezone.utc).isoformat(),
        "user_id": user_id,
        "event_type": etype,
        "amount": amount,
        "device": random.choice(DEVICES),
        "country": random.choice(COUNTRIES)
    }, user_id

def main():
    print(f"Producing to {TOPIC} @ {EPS} eps")
    interval = 1.0 / EPS
    while True:
        payload, key = make_event()
        producer.send(TOPIC, key=key, value=payload)
        time.sleep(interval)

if __name__ == "__main__":
    main()
```

### Run Producer

```bash
python producer.py
```

> **Tip:** Watch events stream live in **Kafka UI → Topics → transactions → Messages**.

---

## 6) Stream Processor — Validate → Aggregate → Emit Features

Create `app/processor.py`:

```python
import json
import os
import time
from collections import defaultdict
from datetime import datetime, timezone
from dateutil import parser as dtparser
from pydantic import BaseModel, Field, ValidationError
from kafka import KafkaConsumer, KafkaProducer

BOOTSTRAP = os.getenv("BOOTSTRAP", "localhost:9092")
SRC_TOPIC = os.getenv("SRC_TOPIC", "transactions")
SINK_TOPIC = os.getenv("SINK_TOPIC", "features")
DLQ_TOPIC = os.getenv("DLQ_TOPIC", "dlq-transactions")  # optional
WINDOW_SEC = int(os.getenv("WINDOW_SEC", "60"))

class Txn(BaseModel):
    event_time: str
    user_id: int = Field(ge=1)
    event_type: str
    amount: float
    device: str
    country: str

def epoch_minute(ts_iso: str) -> int:
    ts = dtparser.isoparse(ts_iso)
    return int(ts.timestamp() // WINDOW_SEC) * WINDOW_SEC

consumer = KafkaConsumer(
    SRC_TOPIC,
    bootstrap_servers=BOOTSTRAP,
    group_id="processor-1",
    value_deserializer=lambda v: json.loads(v.decode("utf-8")),
    key_deserializer=lambda k: int(k.decode("utf-8")) if k else None,
    auto_offset_reset="earliest",
    enable_auto_commit=False,
    max_poll_records=200
)

producer = KafkaProducer(
    bootstrap_servers=BOOTSTRAP,
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
)

# State format: (user_id, window_start) -> counters
state = defaultdict(lambda: {"event_count": 0, "purchase_count": 0, "revenue": 0.0})
last_flush = time.time()

def flush_ready(now_epoch: int):
    """Flush windows that ended before the current window."""
    to_delete = []
    for (user_id, win_start), agg in state.items():
        if win_start + WINDOW_SEC <= now_epoch - 1:
            out = {
                "user_id": user_id,
                "window_start": win_start,
                "window_end": win_start + WINDOW_SEC,
                "event_count": agg["event_count"],
                "purchase_count": agg["purchase_count"],
                "revenue": round(agg["revenue"], 2),
                "emitted_at": datetime.now(timezone.utc).isoformat()
            }
            producer.send(SINK_TOPIC, value=out)
            to_delete.append((user_id, win_start))
            
    for k in to_delete:
        del state[k]

try:
    while True:
        records = consumer.poll(timeout_ms=1000)
        if not records: 
            flush_ready(int(time.time()))
            continue

        for tp, msgs in records.items():
            for msg in msgs:
                try:
                    txn = Txn(**msg.value)
                    win = epoch_minute(txn.event_time)
                    key = (txn.user_id, win)
                    st = state[key]
                    st["event_count"] += 1
                    if txn.event_type == "purchase":
                        st["purchase_count"] += 1
                        st["revenue"] += float(txn.amount)
                except ValidationError as e:
                    # Optional DLQ
                    producer.send(DLQ_TOPIC, value={
                        "error": "validation_error",
                        "reason": e.errors(),
                        "payload": msg.value
                    })

        consumer.commit()
        now = time.time()
        if now - last_flush > 5:  # flush roughly every 5 seconds
            flush_ready(int(now))
            last_flush = now

except KeyboardInterrupt:
    flush_ready(int(time.time()))
    consumer.commit()
```

### Run Stream Processor

```bash
python processor.py
```

---

## 7) Feature Consumer — View Results / Output to CSV

Create `app/consumer.py`:

```python
import csv
import os
import json
from kafka import KafkaConsumer

BOOTSTRAP = os.getenv("BOOTSTRAP", "localhost:9092")
TOPIC = os.getenv("TOPIC", "features")
WRITE_CSV = os.getenv("WRITE_CSV", "false").lower() == "true"

consumer = KafkaConsumer(
    TOPIC,
    bootstrap_servers=BOOTSTRAP,
    value_deserializer=lambda v: json.loads(v.decode("utf-8")),
    auto_offset_reset="earliest",
    group_id="features-reader"
)

f = None
writer = None
if WRITE_CSV:
    f = open("features.csv", "w", newline="")

print("Consuming features...")
try:
    for msg in consumer:
        rec = msg.value
        print(rec)
        if WRITE_CSV and f:
            if writer is None:
                writer = csv.DictWriter(f, fieldnames=rec.keys())
                writer.writeheader()
            writer.writerow(rec)
finally:
    if f:
        f.close()
```

### Run Consumer Options

```bash
# Print output directly to stdout:
python consumer.py

# Or write directly to a CSV file:
WRITE_CSV=true python consumer.py
```

> **Kafka UI Validation:** Check **Topics → features → Messages** to view aggregated streaming results.

---

## 8) Sanity Checks

* **Throughput:** Check **Consumer Groups → `processor-1`** in the Kafka UI to ensure lag stays near `0`.
* **Window Rolling:** Verify that `window_start` increments every minute per user ID in the emitted features.
* **DLQ Verification:** Edit `producer.py` to periodically emit invalid events (e.g., negative `user_id` or non-numeric amount). Verify that bad records land directly in `dlq-transactions`.

---

## 9) Experiments

1. **Scale Partitions:** Increase `transactions` partitions in Kafka UI. Launch a 2nd processor process (`GROUP_ID=processor-1`). Notice how Kafka automatically balances partition assignment across both instances.
2. **Backpressure:** Bump `EVENTS_PER_SECOND=50` on the producer and monitor how consumer lag behaves in Kafka UI.
3. **Latency:** Decrease `WINDOW_SEC=30` and lower flush intervals to produce higher-frequency windows.
4. **Schema Evolution:** Introduce a new field (e.g., `marketing_channel`) to `producer.py` and verify your stream processor handles it without breaking.
5. **Fault Injection:** Kill `processor.py` mid-execution, restart it, and confirm it resumes exactly from the last committed offsets.

---

## 10) Cleanup

1. Stop Python scripts using `Ctrl+C` in your terminal tabs.
2. Tear down the local Docker infrastructure and clean up Python virtual environments:

```bash
docker compose down -v
deactivate
```

---

## Expected Output Format

* **`transactions` topic:** High-frequency event streams (`view`, `add_to_cart`, `purchase`).
* **`features` topic:** Minute-bucketed aggregates grouped per user:

```json
{
  "user_id": 123,
  "window_start": 1724782380,
  "window_end": 1724782440,
  "event_count": 9,
  "purchase_count": 2,
  "revenue": 153.17,
  "emitted_at": "2025-08-27T01:23:45.678901+00:00"
}
```

---

## Stretch Goals

* [ ] Replace Python window aggregation with **Apache Spark Structured Streaming** or **Apache Flink**.
* [ ] Spin up a **Postgres** container and persist feature streams to database tables via your consumer.
* [ ] Write Dockerfiles for `producer`, `processor`, and `consumer` to fully containerize the application stack.
* [ ] Attach **Prometheus + Grafana** to expose metrics like processing latency and throughput.
* [ ] Migrate in-memory window states to **Redis** or **RocksDB** for stateful recovery across container restarts.
