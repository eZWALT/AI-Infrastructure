# Day 238 Lab – Automate Drift Retraining with Kubeflow

## Learning Goals
* Detect **data drift** in production pipelines
* Automate **retraining & evaluation** with Kubeflow
* Implement **conditional deployment** if retrained model passes gates
* Track artifacts and lineage with Kubeflow Pipelines

---

## 0) Prerequisites
* Running **Kubeflow Pipelines (KFP)** instance
* Access to storage (MinIO/S3/GCS) for datasets & models
* Install Python SDK:
  ```bash
  pip install kfp evidently scikit-learn pandas joblib
  ```

---

## 1) Create Drift Detection Component

**File:** `drift_detector.py`

```python
import pandas as pd
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset

def detect_drift(ref_data_path="/mnt/data/ref.csv", new_data_path="/mnt/data/new.csv", threshold=0.1):
    ref = pd.read_csv(ref_data_path)
    new = pd.read_csv(new_data_path)

    report = Report(metrics=[DataDriftPreset()])
    report.run(reference_data=ref, current_data=new)
    drift_share = report.as_dict()["metrics"][0]["result"]["drift_share"]

    print(f"Drift detected: {drift_share:.2f}")
    if drift_share > threshold:
        exit(0)  # trigger downstream retraining
    else:
        exit(1)  # stop retraining
```

---

## 2) Training Component

**File:** `train.py`

```python
import pandas as pd
from sklearn.linear_model import LogisticRegression
import joblib

def train_model(train_path="/mnt/data/train.csv", model_out="/mnt/data/model.pkl"):
    df = pd.read_csv(train_path)
    X, y = df.drop("label", axis=1), df["label"]

    model = LogisticRegression(max_iter=200)
    model.fit(X, y)
    joblib.dump(model, model_out)
```

---

## 3) Evaluation Component

**File:** `evaluate.py`

```python
import pandas as pd
import joblib
from sklearn.metrics import accuracy_score

def evaluate(model_path="/mnt/data/model.pkl", test_path="/mnt/data/test.csv", threshold=0.85):
    model = joblib.load(model_path)
    test = pd.read_csv(test_path)
    X, y = test.drop("label", axis=1), test["label"]

    preds = model.predict(X)
    acc = accuracy_score(y, preds)
    print(f"Model accuracy: {acc:.3f}")
    if acc >= threshold:
        exit(0)  # pass promotion gate
    else:
        exit(1)  # fail
```

---

## 4) Define Kubeflow Pipeline

**File:** `drift_retrain_pipeline.py`

```python
import kfp
from kfp import dsl

@dsl.pipeline(name="drift-retrain-pipeline", description="Automated drift retraining pipeline")
def pipeline(threshold: float = 0.1, acc_threshold: float = 0.85):

    detect = dsl.ContainerOp(
        name="Detect Drift",
        image="python:3.9",
        command=["python", "drift_detector.py"],
        arguments=["--threshold", str(threshold)]
    )

    train = dsl.ContainerOp(
        name="Train Model",
        image="python:3.9",
        command=["python", "train.py"]
    ).after(detect)

    evaluate = dsl.ContainerOp(
        name="Evaluate Model",
        image="python:3.9",
        command=["python", "evaluate.py", "--threshold", str(acc_threshold)]
    ).after(train)

    # Conditional deployment
    with dsl.Condition(evaluate.output == "0"):
        deploy = dsl.ContainerOp(
            name="Deploy Model",
            image="myregistry/deploy:latest",
            command=["sh", "-c"],
            arguments=["echo Deploying model..."]
        )
```

---

## 5) Compile & Upload Pipeline

Run the following command to compile the pipeline:

```bash
python -m kfp.compiler.cli compile \
    --py drift_retrain_pipeline.py \
    --output drift_retrain_pipeline.yaml
```

Upload `drift_retrain_pipeline.yaml` to the **Kubeflow Pipelines UI**.

---

## 6) Run Pipeline

* Set `threshold=0.1` for drift sensitivity.
* If **drift > 10%**, retraining is triggered.
* If **new accuracy ≥ 0.85**, the pipeline promotes the model to deployment.

---

## 7) Stretch Goals

* Replace dummy deploy step with **KServe model deployment**.
* Store drift reports in **MinIO/S3** as artifacts.
* Add **Katib HPO step** for retraining optimization.
* Trigger pipeline automatically from **Kafka/BigQuery events**.

---

**✅ Outcome:** You built an **automated Kubeflow pipeline** that detects drift, retrains models, evaluates performance, and conditionally deploys only if gates are passed.
