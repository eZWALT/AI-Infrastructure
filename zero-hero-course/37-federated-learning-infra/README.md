# Day 259 Lab – Train a Federated Model with TFF

## 🧪 Lab — Train a Federated Model with TensorFlow Federated (TFF)

### 🎯 Objective
* Learn how to use **TensorFlow Federated (TFF)** to simulate federated training.
* Train a model on **decentralized data** (EMNIST dataset).
* Observe how **local training + aggregation** builds a global model.

---

### Step 1: Environment Setup

Install TensorFlow & TFF:
```bash
pip install tensorflow tensorflow_federated
```

Verify installation:
```python
import tensorflow as tf
import tensorflow_federated as tff

print("TF version:", tf.__version__)
print("TFF version:", tff.__version__)
```

✅ **Expected:** Versions print without errors.

---

### Step 2: Load a Federated Dataset
We’ll use **EMNIST (Federated)** — handwritten digits/letters split by writer → mimics client data.

```python
emnist_train, emnist_test = tff.simulation.datasets.emnist.load_data()

# Pick sample clients (e.g., first 5)
sample_clients = emnist_train.client_ids[:5]

# Create federated datasets
federated_train_data = [emnist_train.create_tf_dataset_for_client(c) 
                        for c in sample_clients]
```

✅ **Expected:** You now have a list of datasets, one per client.

---

### Step 3: Define the Model
We’ll wrap a simple **Keras model** for TFF.

```python
def create_keras_model():
    return tf.keras.Sequential([
        tf.keras.layers.Reshape((28,28,1), input_shape=(28,28)),
        tf.keras.layers.Conv2D(32, 3, activation='relu'),
        tf.keras.layers.MaxPooling2D(),
        tf.keras.layers.Flatten(),
        tf.keras.layers.Dense(64, activation='relu'),
        tf.keras.layers.Dense(62, activation='softmax')  # 62 EMNIST classes
    ])
```

✅ **Expected:** Model summary shows Conv2D + Dense layers.

---

### Step 4: Wrap Model for TFF
TFF requires a `model_fn` wrapper:

```python
def model_fn():
    return tff.learning.models.from_keras_model(
        keras_model=create_keras_model(),
        input_spec=federated_train_data[0].element_spec,
        loss=tf.keras.losses.SparseCategoricalCrossentropy(),
        metrics=[tf.keras.metrics.SparseCategoricalAccuracy()])
```

✅ **Expected:** No errors → TFF now knows how to use the model.

---

### Step 5: Build Federated Training Algorithm
We’ll use **Federated Averaging (FedAvg)**.

```python
trainer = tff.learning.algorithms.build_weighted_fed_avg(model_fn)
state = trainer.initialize()
```

`state` holds the global model + optimizer state.

---

### Step 6: Run Training Rounds

```python
NUM_ROUNDS = 5
for round_num in range(1, NUM_ROUNDS + 1):
    state, metrics = trainer.next(state, federated_train_data)
    print(f"Round {round_num}, Metrics: {metrics}")
```

✅ **Expected:** Accuracy increases slightly with each round.

---

### Step 7: Evaluate the Global Model

```python
evaluator = tff.learning.algorithms.build_fed_eval(model_fn)
state, eval_metrics = evaluator.next(state, [emnist_test])
print("Final evaluation:", eval_metrics)
```

✅ **Expected:** You’ll see test accuracy (not very high at 5 rounds, but > random).

---

### Step 8: Observe Federated Learning Dynamics
* Each round = local client updates + server aggregation.
* Accuracy improves **without centralizing data**.
* Try different numbers of clients, rounds, and batch sizes.

---

### Step 9 (Optional Extensions)
* Increase `NUM_ROUNDS = 20` → better accuracy.
* Simulate **non-IID clients** by selecting users with skewed data.
* Add **differential privacy optimizers**:
  ```python
  from tensorflow_privacy.privacy.optimizers.dp_optimizer_keras import DPKerasSGDOptimizer
  ```
* Try **custom aggregation functions** instead of FedAvg.

---

### ✅ Wrap-Up
* You trained a model across multiple clients with **TensorFlow Federated**.
* Key takeaway: only **model updates** travel, not raw data.
* You saw how FL maintains privacy while still improving a global model.
