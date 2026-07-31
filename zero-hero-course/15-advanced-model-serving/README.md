# Lab 105: Deploy a Model on Triton Inference Server

## 🎯 Goal
Export a PyTorch model to ONNX format, build a Triton model repository layout, run Triton Inference Server via Docker, and send real HTTP inference requests.

* **Optional Extensions:** TensorRT conversion, dynamic batching tuning, Kubernetes deployment, and Prometheus metrics.

---

## 0. Prerequisites

* **Container Runtime:** Docker installed
* **Hardware Support (Optional):** NVIDIA GPU driver + `nvidia-container-toolkit`
* **Python Environment:** Python 3.10+
  ```bash
  pip install torch torchvision pillow numpy requests
  ```
* **Network Ports:** `8000` (HTTP), `8001` (gRPC), and `8002` (Metrics) available

---

## 1. Create a Triton Model Repository Layout

First, set up the directory structure. Triton expects model repositories to follow a strict naming and versioning convention:

```bash
mkdir -p ~/lab105/models/resnet50_onnx/1
cd ~/lab105
```

### Expected Repository Structure
```text
models/
 └─ resnet50_onnx/
     ├─ config.pbtxt
     └─ 1/
        └─ model.onnx      # Model binary (exported in Step 2)
```

### Configuration (`models/resnet50_onnx/config.pbtxt`)
Create the protobuf configuration file defining the model inputs, outputs, execution targets, and dynamic batching:

```protobuf
name: "resnet50_onnx"
platform: "onnxruntime_onnx"
max_batch_size: 16

input [
  { 
    name: "input"  
    data_type: TYPE_FP32
    dims: [3, 224, 224] 
  }
]
output [
  { 
    name: "logits"
    data_type: TYPE_FP32
    dims: [1000] 
  }
]

instance_group [
  { 
    kind: KIND_GPU
    count: 1 
  }
]

dynamic_batching {
  preferred_batch_size: [4, 8, 16]
  max_queue_delay_microseconds: 1000
}
```

---

## 2. Export a Pretrained ResNet50 to ONNX

Create the script `export_onnx.py` in `~/lab105`:

```python
import torch
import torchvision as tv

def main():
    # Load pretrained ResNet50 in evaluation mode
    model = tv.models.resnet50(weights="DEFAULT").eval()
    
    # Create dummy tensor matching input specification
    dummy = torch.randn(1, 3, 224, 224)
    
    # Export model to ONNX format with dynamic batch sizes
    torch.onnx.export(
        model, 
        dummy, 
        "models/resnet50_onnx/1/model.onnx",
        input_names=["input"], 
        output_names=["logits"],
        dynamic_axes={"input": {0: "batch"}, "logits": {0: "batch"}},
        opset_version=17
    )
    print("Exported to models/resnet50_onnx/1/model.onnx")

if __name__ == "__main__":
    main()
```

Run the export script:

```bash
python export_onnx.py
```

---

## 3. Start Triton Server (Docker)

Set the release tag and start the server container.

```bash
export TRITON_TAG=latest

docker run --rm -it \
  --gpus all \
  -p8000:8000 -p8001:8001 -p8002:8002 \
  -v $PWD/models:/models \
  nvcr.io/nvidia/tritonserver:${TRITON_TAG} tritonserver \
    --model-repository=/models \
    --strict-model-config=false \
    --exit-on-error=false
```

> **Note:** If running on a CPU-only host, omit `--gpus all`.

### Health & Endpoint Verification
Open a separate terminal to run checks:

* **Check Readiness:**
  ```bash
  curl -s http://localhost:8000/v2/health/ready && echo
  ```
* **Inspect Model Metadata:**
  ```bash
  curl -s http://localhost:8000/v2/models/resnet50_onnx | jq
  ```
* **Inspect Prometheus Metrics:**
  ```bash
  curl -s http://localhost:8002/metrics | head
  ```

---

## 4. Send an Inference Request (HTTP/JSON)

Create `client_http.py` in `~/lab105`:

```python
import argparse
import numpy as np
import requests
from PIL import Image

def preprocess(img):
    # Resize and center crop image to 224x224
    img = img.convert("RGB").resize((256, 256))
    o = (256 - 224) // 2
    img = img.crop((o, o, o + 224, o + 224))
    
    # Normalize RGB values according to ImageNet standard parameters
    x = np.asarray(img).astype("float32") / 255.0
    mean = np.array([0.485, 0.456, 0.406], dtype=np.float32)
    std = np.array([0.229, 0.224, 0.225], dtype=np.float32)
    x = (x - mean) / std
    
    # Reshape array layout to NCHW format
    x = np.transpose(x, (2, 0, 1))[None, ...]
    return x

if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("image_path")
    ap.add_argument("--url", default="http://localhost:8000/v2/models/resnet50_onnx/infer")
    args = ap.parse_args()

    x = preprocess(Image.open(args.image_path))
    
    # Construct KServe v2 JSON payload
    payload = {
      "inputs": [{
        "name": "input",
        "shape": list(x.shape),
        "datatype": "FP32",
        "data": x.flatten().tolist()
      }],
      "outputs": [{"name": "logits"}]
    }

    r = requests.post(args.url, json=payload, timeout=60)
    r.raise_for_status()
    
    # Parse output logits and compute Softmax probabilities
    out = r.json()["outputs"][0]["data"]
    logits = np.array(out, dtype=np.float32).reshape((x.shape[0], 1000))
    exps = np.exp(logits - logits.max(axis=1, keepdims=True))
    probs = exps / exps.sum(axis=1, keepdims=True)
    
    top5 = probs[0].argsort()[-5:][::-1]
    print("Top-5 indices:", top5.tolist())
    print("Top-5 probs:", probs[0][top5].tolist())
```

Execute the inference script:

```bash
pip install pillow requests numpy
python client_http.py path/to/image.jpg
```

---

## 5. (Optional) TensorRT Acceleration

Build an optimized FP16 TensorRT engine:

```bash
trtexec --onnx=models/resnet50_onnx/1/model.onnx \
        --saveEngine=models/resnet50_trt/1/model.plan \
        --explicitBatch --fp16
```

Create configuration file `models/resnet50_trt/config.pbtxt`:

```protobuf
name: "resnet50_trt"
platform: "tensorrt_plan"
max_batch_size: 16

input:  { name: "input",  data_type: TYPE_FP32, dims: [3, 224, 224] }
output: { name: "logits", data_type: TYPE_FP32, dims: [1000] }

instance_group [ { kind: KIND_GPU, count: 1 } ]

dynamic_batching { 
  preferred_batch_size: [4, 8, 16]
  max_queue_delay_microseconds: 1000 
}
```

Restart Triton (or enable repository polling) and route requests to `resnet50_trt`.

---

## 6. Tune Throughput & Latency

1. **Scale Model Instances:** Increase instance counts per GPU within `config.pbtxt` to leverage parallel execution:
   ```protobuf
   instance_group [ { kind: KIND_GPU, count: 2 } ]
   ```
2. **Optimize Queue Delays:** Tune `preferred_batch_size` and `max_queue_delay_microseconds` to balance overall batch throughput against maximum latency targets (p95 / p99).
3. **Load Testing:** Benchmark under dynamic load using tools like `hey` or `Locust`.

---

## 7. (Optional) Kubernetes Deployment

Create deployment definition `k8s-triton.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triton
spec:
  replicas: 1
  selector:
    matchLabels:
      app: triton
  template:
    metadata:
      labels:
        app: triton
    spec:
      containers:
      - name: triton
        image: nvcr.io/nvidia/tritonserver:latest
        args: ["tritonserver", "--model-repository=/models"]
        ports:
        - containerPort: 8000
        - containerPort: 8001
        - containerPort: 8002
        volumeMounts:
        - name: model-repo
          mountPath: /models
        resources:
          limits:
            nvidia.com/gpu: 1
      volumes:
      - name: model-repo
        hostPath:
          path: /path/on/node/models  # Replace with actual host path or PersistentVolumeClaim
---
apiVersion: v1
kind: Service
metadata:
  name: triton-svc
spec:
  selector:
    app: triton
  type: NodePort
  ports:
  - name: http
    port: 8000
    targetPort: 8000
  - name: grpc
    port: 8001
    targetPort: 8001
  - name: metrics
    port: 8002
    targetPort: 8002
```

Deploy to Kubernetes cluster:

```bash
kubectl apply -f k8s-triton.yaml
```

---

## 8. Troubleshooting

| Symptom | Cause | Solution |
| :--- | :--- | :--- |
| **Model fails to load** | Configuration mismatch | Inspect Triton logs for name/shape/type discrepancies; align `config.pbtxt` with the ONNX layer names. |
| **HTTP 400 Bad Request** | Invalid payload layout | Confirm request body strictly matches target name `"input"`, tensor shape `[N,3,224,224]`, and `datatype: "FP32"`. |
| **Low Throughput** | Suboptimal batch configuration | Enable dynamic batching, scale `instance_group` count, or convert model to TensorRT engine. |
| **Slow CPU Execution** | Expected performance profile | Utilize GPU execution instances or optimized hardware backends for production workloads. |
| **Numeric Outputs** | Unmapped predictions | Result indices can be mapped to readable label datasets via an optional `labels.txt` mapping file inside the model directory. |

---

## ✅ Summary of Achievements

- [x] Exported a deep learning model to standard **ONNX format**.
- [x] Created an organized **Triton Model Repository structure** and configuration.
- [x] Served inferencing over **HTTP/gRPC endpoints** and exposed **Prometheus metrics**.
- [x] Preprocessed images and submitted real-time inference requests.
- [x] Explored advanced tuning pathways including **dynamic batching, TensorRT compilation, and Kubernetes deployment**.

---

# 🚀 Beyond the Lab (High ROI Extensions)

Complete these only after the original lab is working.

Each extension should take approximately **15–30 minutes**.

---

# Extension 1 – Use Triton's Official Python Client (HTTP)

Until now you've manually constructed HTTP requests using `requests`.

In production you'll almost always use Triton's SDK instead.

## Install

```bash
pip install tritonclient[http]
```

Create:

```text
client_sdk_http.py
```

Load an image exactly as before, but replace the raw JSON request with the Triton client library.

Use:

- `tritonclient.http.InferenceServerClient`
- `InferInput`
- `InferRequestedOutput`

Verify that predictions match those from `client_http.py`.

✅ You learned the standard Triton HTTP client.

---

# Extension 2 – Try gRPC Inference

One of Triton's biggest advantages is native gRPC support.

Install the client:

```bash
pip install tritonclient[grpc]
```

Create:

```text
client_grpc.py
```

Connect to

```text
localhost:8001
```

instead of

```text
localhost:8000
```

Run inference on the same image.

Compare:

- HTTP latency
- gRPC latency
- Client code complexity

You should now understand why most production Triton deployments expose gRPC.

✅ Triton running over both HTTP and gRPC.

---

# Extension 3 – Observe Dynamic Batching

Edit

```text
models/resnet50_onnx/config.pbtxt
```

Change

```text
preferred_batch_size

[4,8,16]
```

to

```text
[1]
```

Restart Triton.

Run several inference requests.

Now restore

```text
preferred_batch_size

[4,8,16]
```

Restart Triton again.

Generate concurrent requests:

```bash
hey -n 200 -c 20 \
-m POST \
-T application/json \
-D payload.json \
http://localhost:8000/v2/models/resnet50_onnx/infer
```

Watch throughput increase once Triton starts batching requests.

*(You don't need huge gains on CPU—the goal is understanding how batching works.)*

✅ First experience with dynamic batching.

---

# Extension 4 – Explore Triton Metrics

Open:

```text
http://localhost:8002/metrics
```

Search for:

```
nv_inference_request_success

nv_inference_execution_count

nv_inference_request_duration_us

nv_inference_queue_duration_us
```

While repeatedly running inference:

```bash
python client_http.py image.jpg
```

Refresh the metrics page.

Observe how the counters increase.

Think about what each metric represents:

- Number of requests
- Execution time
- Queueing delay
- Successful inferences

These metrics are what Prometheus and Grafana collect in production.

✅ Learned how Triton exposes operational metrics.

---

# 🎯 Additional Learning Outcomes

After completing these extensions you will also know how to:

- Use Triton's official HTTP SDK
- Serve models over gRPC
- Understand and observe dynamic batching
- Inspect Triton's built-in Prometheus metrics
