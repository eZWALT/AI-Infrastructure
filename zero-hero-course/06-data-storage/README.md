# Lab 36 — Build an AI Data Ingestion Pipeline

**Goal:** Build an end-to-end data ingestion pipeline that stores raw data in object storage, streams updates through Kafka, and loads curated data into PostgreSQL.

**Estimated Time:** 90–120 minutes

**Requirements:**

- Docker & Docker Compose
- Python 3.10+
- PostgreSQL
- Kafka
- MinIO (S3-compatible object storage)

---

# Learning Objectives

By the end of this lab you will be able to:

- Store raw datasets in object storage
- Stream events using Apache Kafka
- Consume streaming data with Python
- Load processed data into PostgreSQL
- Understand the architecture of a modern AI data ingestion pipeline

---

# Architecture

```
                 +------------------+
                 | transactions.csv |
                 +--------+---------+
                          |
                          v
                   +-------------+
                   |   MinIO/S3  |
                   +-------------+
                          |
                          |
                (new transactions)
                          |
                          v
                   +-------------+
                   |    Kafka    |
                   +-------------+
                          |
                          v
                 +----------------+
                 | Kafka Consumer |
                 +----------------+
                          |
                          v
                  +---------------+
                  | PostgreSQL DB |
                  +---------------+
```

---

# Project Structure

```
.
├── docker-compose.yml
├── transactions.csv
├── ingest_to_minio.py
├── kafka_producer.py
└── kafka_consumer.py
```

---

# Step 1 — Create the Project

Create a working directory:

```bash
mkdir ai-ingestion-lab

cd ai-ingestion-lab
```

---

# Step 2 — Start the Infrastructure

Create a file named **`docker-compose.yml`**.

```yaml
version: "3.9"

services:

  minio:
    image: minio/minio
    command: server /data

    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password

    ports:
      - "9000:9000"
      - "9001:9001"

  kafka:
    image: bitnami/kafka:latest

    environment:
      KAFKA_ENABLE_KRAFT: yes
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE: "true"

    ports:
      - "9092:9092"

  postgres:
    image: postgres:14

    environment:
      POSTGRES_USER: aiuser
      POSTGRES_PASSWORD: aipass
      POSTGRES_DB: aidb

    ports:
      - "5432:5432"
```

Start all services:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

You should see:

- MinIO
- Kafka
- PostgreSQL

---

# Step 3 — Upload Data to Object Storage

Create **`ingest_to_minio.py`**.

```python
from minio import Minio

client = Minio(
    "localhost:9000",
    access_key="admin",
    secret_key="password",
    secure=False
)

bucket = "rawdata"

if not client.bucket_exists(bucket):
    client.make_bucket(bucket)

client.fput_object(
    bucket,
    "transactions.csv",
    "transactions.csv"
)

print("Uploaded transactions.csv")
```

Install dependencies:

```bash
pip install minio
```

Run:

```bash
python ingest_to_minio.py
```

Open the MinIO Console:

```
http://localhost:9001
```

You should see the uploaded CSV file.

---

# Step 4 — Produce Streaming Events

Create **`kafka_producer.py`**.

```python
import json
import random
import time

from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers="localhost:9092",
    value_serializer=lambda x: json.dumps(x).encode()
)

for i in range(10):

    record = {
        "user_id": i,
        "amount": random.randint(10, 1000)
    }

    producer.send("transactions", record)

    print(record)

    time.sleep(1)

producer.flush()
```

Install Kafka client:

```bash
pip install kafka-python
```

Run:

```bash
python kafka_producer.py
```

Ten transaction events should be published to Kafka.

---

# Step 5 — Consume Events and Store Them

Create **`kafka_consumer.py`**.

```python
import json

import psycopg2

from kafka import KafkaConsumer

consumer = KafkaConsumer(
    "transactions",
    bootstrap_servers="localhost:9092",
    value_deserializer=lambda m: json.loads(m.decode())
)

conn = psycopg2.connect(
    dbname="aidb",
    user="aiuser",
    password="aipass",
    host="localhost",
    port=5432
)

cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS transactions(
    user_id INT,
    amount INT
)
""")

for message in consumer:

    record = message.value

    cur.execute(
        "INSERT INTO transactions VALUES (%s,%s)",
        (record["user_id"], record["amount"])
    )

    conn.commit()

    print(record)
```

Install dependencies:

```bash
pip install kafka-python psycopg2
```

Run:

```bash
python kafka_consumer.py
```

Incoming Kafka messages should now be inserted into PostgreSQL.

---

# Step 6 — Verify the Pipeline

## Verify Object Storage

Open:

```
http://localhost:9001
```

Confirm that `transactions.csv` exists.

---

## Verify PostgreSQL

Find the PostgreSQL container:

```bash
docker ps
```

Run:

```bash
docker exec -it <POSTGRES_CONTAINER> \
psql -U aiuser -d aidb
```

Query the table:

```sql
SELECT * FROM transactions;
```

You should see all streamed records.

---

## Verify the Complete Pipeline

The complete data flow should look like this:

```
CSV File
    │
    ▼
 MinIO (Raw Storage)
    │
    ▼
 Kafka Producer
    │
    ▼
 Kafka Topic
    │
    ▼
 Kafka Consumer
    │
    ▼
 PostgreSQL
```

---

# Cleanup

Stop and remove all containers and volumes:

```bash
docker compose down -v
```

---

# Deliverables

Submit:

- Screenshot of the MinIO bucket
- Screenshot of the Kafka producer output
- Screenshot of the Kafka consumer output
- Screenshot of the PostgreSQL table
- A brief explanation (≈150 words) describing how data flows through the pipeline

---

# Summary

In this lab you learned how to:

- Store raw datasets in object storage
- Build an event-driven ingestion pipeline
- Publish streaming events with Kafka
- Consume events using Python
- Load curated data into PostgreSQL
- Understand the architecture behind modern AI data pipelines# Lab 36 — Build an AI Data Ingestion Pipeline

**Goal:** Build an end-to-end data ingestion pipeline that stores raw data in object storage, streams updates through Kafka, and loads curated data into PostgreSQL.

**Estimated Time:** 90–120 minutes

**Requirements:**

- Docker & Docker Compose
- Python 3.10+
- PostgreSQL
- Kafka
- MinIO (S3-compatible object storage)

---

# Learning Objectives

By the end of this lab you will be able to:

- Store raw datasets in object storage
- Stream events using Apache Kafka
- Consume streaming data with Python
- Load processed data into PostgreSQL
- Understand the architecture of a modern AI data ingestion pipeline

---

# Architecture

```
                 +------------------+
                 | transactions.csv |
                 +--------+---------+
                          |
                          v
                   +-------------+
                   |   MinIO/S3  |
                   +-------------+
                          |
                          |
                (new transactions)
                          |
                          v
                   +-------------+
                   |    Kafka    |
                   +-------------+
                          |
                          v
                 +----------------+
                 | Kafka Consumer |
                 +----------------+
                          |
                          v
                  +---------------+
                  | PostgreSQL DB |
                  +---------------+
```

---

# Project Structure

```
.
├── docker-compose.yml
├── transactions.csv
├── ingest_to_minio.py
├── kafka_producer.py
└── kafka_consumer.py
```

---

# Step 1 — Create the Project

Create a working directory:

```bash
mkdir ai-ingestion-lab

cd ai-ingestion-lab
```

---

# Step 2 — Start the Infrastructure

Create a file named **`docker-compose.yml`**.

```yaml
version: "3.9"

services:

  minio:
    image: minio/minio
    command: server /data

    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password

    ports:
      - "9000:9000"
      - "9001:9001"

  kafka:
    image: bitnami/kafka:latest

    environment:
      KAFKA_ENABLE_KRAFT: yes
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE: "true"

    ports:
      - "9092:9092"

  postgres:
    image: postgres:14

    environment:
      POSTGRES_USER: aiuser
      POSTGRES_PASSWORD: aipass
      POSTGRES_DB: aidb

    ports:
      - "5432:5432"
```

Start all services:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

You should see:

- MinIO
- Kafka
- PostgreSQL

---

# Step 3 — Upload Data to Object Storage

Create **`ingest_to_minio.py`**.

```python
from minio import Minio

client = Minio(
    "localhost:9000",
    access_key="admin",
    secret_key="password",
    secure=False
)

bucket = "rawdata"

if not client.bucket_exists(bucket):
    client.make_bucket(bucket)

client.fput_object(
    bucket,
    "transactions.csv",
    "transactions.csv"
)

print("Uploaded transactions.csv")
```

Install dependencies:

```bash
pip install minio
```

Run:

```bash
python ingest_to_minio.py
```

Open the MinIO Console:

```
http://localhost:9001
```

You should see the uploaded CSV file.

---

# Step 4 — Produce Streaming Events

Create **`kafka_producer.py`**.

```python
import json
import random
import time

from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers="localhost:9092",
    value_serializer=lambda x: json.dumps(x).encode()
)

for i in range(10):

    record = {
        "user_id": i,
        "amount": random.randint(10, 1000)
    }

    producer.send("transactions", record)

    print(record)

    time.sleep(1)

producer.flush()
```

Install Kafka client:

```bash
pip install kafka-python
```

Run:

```bash
python kafka_producer.py
```

Ten transaction events should be published to Kafka.

---

# Step 5 — Consume Events and Store Them

Create **`kafka_consumer.py`**.

```python
import json

import psycopg2

from kafka import KafkaConsumer

consumer = KafkaConsumer(
    "transactions",
    bootstrap_servers="localhost:9092",
    value_deserializer=lambda m: json.loads(m.decode())
)

conn = psycopg2.connect(
    dbname="aidb",
    user="aiuser",
    password="aipass",
    host="localhost",
    port=5432
)

cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS transactions(
    user_id INT,
    amount INT
)
""")

for message in consumer:

    record = message.value

    cur.execute(
        "INSERT INTO transactions VALUES (%s,%s)",
        (record["user_id"], record["amount"])
    )

    conn.commit()

    print(record)
```

Install dependencies:

```bash
pip install kafka-python psycopg2
```

Run:

```bash
python kafka_consumer.py
```

Incoming Kafka messages should now be inserted into PostgreSQL.

---

# Step 6 — Verify the Pipeline

## Verify Object Storage

Open:

```
http://localhost:9001
```

Confirm that `transactions.csv` exists.

---

## Verify PostgreSQL

Find the PostgreSQL container:

```bash
docker ps
```

Run:

```bash
docker exec -it <POSTGRES_CONTAINER> \
psql -U aiuser -d aidb
```

Query the table:

```sql
SELECT * FROM transactions;
```

You should see all streamed records.

---

## Verify the Complete Pipeline

The complete data flow should look like this:

```
CSV File
    │
    ▼
 MinIO (Raw Storage)
    │
    ▼
 Kafka Producer
    │
    ▼
 Kafka Topic
    │
    ▼
 Kafka Consumer
    │
    ▼
 PostgreSQL
```

---

# Cleanup

Stop and remove all containers and volumes:

```bash
docker compose down -v
```

---

# Deliverables

Submit:

- Screenshot of the MinIO bucket
- Screenshot of the Kafka producer output
- Screenshot of the Kafka consumer output
- Screenshot of the PostgreSQL table
- A brief explanation (≈150 words) describing how data flows through the pipeline

---

# Summary

In this lab you learned how to:

- Store raw datasets in object storage
- Build an event-driven ingestion pipeline
- Publish streaming events with Kafka
- Consume events using Python
- Load curated data into PostgreSQL
- Understand the architecture behind modern AI data pipelines