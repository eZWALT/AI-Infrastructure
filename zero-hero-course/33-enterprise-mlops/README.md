# Day 231 Lab – Build a Pipeline in Kubeflow

## Learning Goals
* Define a pipeline in **Kubeflow Pipelines (KFP)**
* Build reusable pipeline components (preprocess, train, evaluate)
* Compile and run the pipeline in the Kubeflow UI
* Track artifacts, metrics, and pipeline lineage

---

## 0) Prerequisites
* A running **Kubeflow Pipelines** deployment (on GCP, AWS, Azure, or on-prem)
* Install SDK locally:
  ```bash
  pip install kfp
  ```
* *Optional:* Access to Jupyter Notebook / Kubeflow Notebooks

---

## 1) Create Pipeline Components

### Preprocessing Component – `preprocess.py`
```python
def preprocess_op():
    import pandas as pd
    from sklearn.model_selection import train_test_split
    
    df = pd.read_csv("/mnt/data/iris.csv")
    train, test = train_test_split(df, test_size=0.2, random_state=42)
    train.to_csv("/mnt/data/train.csv", index=False)
    test.to_csv("/mnt/data/test.csv", index=False)
```

### Training Component – `train.py`
```python
def train_op():
    import pandas as pd
    from sklearn.linear_model import LogisticRegression
    import joblib

    train = pd.read_csv("/mnt/data/train.csv")
    X, y = train.drop("species", axis=1), train["species"]

    model = LogisticRegression(max_iter=200)
    model.fit(X, y)
    joblib.dump(model, "/mnt/data/model.pkl")
```

### Evaluation Component – `evaluate.py`
```python
def evaluate_op():
    import pandas as pd
    import joblib
    from sklearn.metrics import accuracy_score

    test = pd.read_csv("/mnt/data/test.csv")
    X, y = test.drop("species", axis=1), test["species"]

    model = joblib.load("/mnt/data/model.pkl")
    preds = model.predict(X)

    acc = accuracy_score(y, preds)
    print(f"Model accuracy: {acc}")
```

---

## 2) Define Pipeline with KFP SDK

**File:** `iris_pipeline.py`

```python
import kfp
from kfp import dsl
from kfp.dsl import pipeline

@pipeline(name="iris-classifier-pipeline", description="Simple Iris ML pipeline")
def iris_pipeline():
    preprocess = dsl.ContainerOp(
        name="Preprocess Data",
        image="python:3.9",
        command=["python", "preprocess.py"]
    )
    train = dsl.ContainerOp(
        name="Train Model",
        image="python:3.9",
        command=["python", "train.py"]
    ).after(preprocess)
    evaluate = dsl.ContainerOp(
        name="Evaluate Model",
        image="python:3.9",
        command=["python", "evaluate.py"]
    ).after(train)
```

---

## 3) Compile the Pipeline

Run the following command to compile the pipeline into a YAML definition:

```bash
python -m kfp.compiler.cli compile \
    --py iris_pipeline.py \
    --output iris_pipeline.yaml
```

*This generates a YAML file to upload to Kubeflow.*

---

## 4) Upload & Run Pipeline

1. Go to **Kubeflow Pipelines UI** → **Upload pipeline** (`iris_pipeline.yaml`).
2. Create a new **Run** with default parameters.
3. Observe DAG execution: `preprocess` → `train` → `evaluate`.

---

## 5) Track Artifacts & Metrics

* **Preprocess step** → outputs `train`/`test` datasets
* **Train step** → model artifact (`model.pkl`)
* **Evaluate step** → logs accuracy to Kubeflow UI
* Explore lineage in **Pipeline Dashboard**

---

## 6) Stretch Goals

* Add **hyperparameter tuning step** with Katib.
* Store model in **MinIO/S3** and register with MLflow/KServe.
* Add conditional step: deploy only if `accuracy > threshold`.
* Convert components into **reusable YAML ops** for team reuse.

---

**✅ Outcome:** You built and executed a Kubeflow pipeline with preprocessing, training, and evaluation stages, and tracked results through the KFP UI.
