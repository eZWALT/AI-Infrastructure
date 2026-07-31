# Lab 154 – Optimize a Model with TensorRT

## Goal

Take a pretrained vision model (**ResNet18**) → convert it to TensorRT engines (**FP16 & INT8**) → benchmark performance improvements on Jetson / NVIDIA GPU.

By the end of this lab you should understand:

- PyTorch → ONNX export
- ONNX Runtime baseline benchmarking
- TensorRT engine generation
- FP16 acceleration
- INT8 quantization and calibration
- Inference optimization trade-offs

---

# Step 0 – Prerequisites

## Hardware

Any NVIDIA platform:

- NVIDIA Jetson Nano / Xavier / Orin
- NVIDIA GPU workstation
- Cloud GPU instance

## Software

Required:

- CUDA
- TensorRT
- PyTorch
- torchvision
- ONNX
- ONNX Runtime

Install Python dependencies:

```bash
pip install torch torchvision onnx onnxruntime
```

Verify TensorRT installation:

```bash
trtexec --version
```

---

# Step 1 – Export PyTorch Model to ONNX

Convert ResNet18 from PyTorch format into an interoperable ONNX graph.

```python
import torch
import torchvision.models as models

# Load pretrained ResNet18
model = models.resnet18(pretrained=True).eval()

dummy = torch.randn(1, 3, 224, 224)

torch.onnx.export(
    model,
    dummy,
    "resnet18.onnx",
    input_names=["input"],
    output_names=["output"],
    opset_version=17
)

print("ONNX model saved: resnet18.onnx")
```

Output:

```
resnet18.onnx
```

---

# Step 2 – Baseline ONNX Runtime Inference

Before optimizing, establish a baseline.

```python
import onnxruntime as ort
import numpy as np
import time

session = ort.InferenceSession(
    "resnet18.onnx"
)

input_name = session.get_inputs()[0].name

x = np.random.rand(
    1, 3, 224, 224
).astype(np.float32)


start = time.time()

output = session.run(
    None,
    {input_name: x}
)

latency = (time.time() - start) * 1000

print(f"Latency: {latency:.2f} ms")
```

Record:

- Latency
- Throughput
- GPU utilization

Example:

```
ONNX Runtime FP32: ~50-100ms on Jetson Nano
```

---

# Step 3 – Build TensorRT FP16 Engine

Convert ONNX graph into an optimized TensorRT engine.

```bash
trtexec \
    --onnx=resnet18.onnx \
    --saveEngine=resnet18_fp16.engine \
    --fp16
```

Explanation:

- TensorRT optimizes kernels
- FP16 reduces memory bandwidth
- Tensor cores accelerate computation

Output:

```
resnet18_fp16.engine
```

---

# Step 4 – Benchmark FP16 Engine

Run inference:

```bash
trtexec \
    --loadEngine=resnet18_fp16.engine \
    --shapes=input:1x3x224x224
```

Compare:

```
ONNX Runtime FP32
        vs
TensorRT FP16
```

Expected:

```
~2-3x speedup
```

---

# Step 5 – INT8 Quantization

INT8 reduces model precision from:

```
FP32 → INT8
```

Advantages:

- Lower memory usage
- Faster inference
- Better edge deployment

Create a calibration dataset:

```
calibration/

├── image001.jpg
├── image002.jpg
└── ...
```

Recommended:

```
50-100 representative images
```

Build INT8 engine:

```bash
trtexec \
    --onnx=resnet18.onnx \
    --saveEngine=resnet18_int8.engine \
    --int8 \
    --calib=calib.cache
```

---

# Step 6 – Benchmark INT8 Engine

Run:

```bash
trtexec \
    --loadEngine=resnet18_int8.engine \
    --shapes=input:1x3x224x224
```

Expected:

```
3-5x speedup vs FP32
```

Accuracy impact:

```
<1-2% degradation
```

---

# Step 7 – Compare Results

Example benchmark:

| Engine | Precision | Latency | Speedup | Accuracy Δ |
|---|---|---:|---:|---:|
| ONNX Runtime | FP32 | 50ms | 1x | baseline |
| TensorRT | FP16 | 20ms | 2.5x | ~0% |
| TensorRT | INT8 | 12ms | 4x | <2% |

---

# Step 8 – Integrate with PyTorch (Torch-TensorRT)

TensorRT can also compile directly from PyTorch.

Install:

```bash
pip install torch-tensorrt
```

Compile:

```python
import torch_tensorrt

trt_model = torch_tensorrt.compile(
    model,
    inputs=[
        torch_tensorrt.Input(
            (1,3,224,224)
        )
    ],
    enabled_precisions={
        torch.float16
    }
)

torch.jit.save(
    trt_model,
    "resnet18_trt.ts"
)
```

---

# Step 9 – Cleanup

```bash
rm resnet18.onnx \
   resnet18_fp16.engine \
   resnet18_int8.engine
```

---

# Folder Structure

```
lab154_tensorrt_opt/

├── export_resnet18.py
├── benchmark_onnx.py
├── README.md
│
├── calibration/
│   └── images/
│
└── engines/
    ├── resnet18.onnx
    ├── resnet18_fp16.engine
    └── resnet18_int8.engine
```

---

# Extra High-ROI Exercises

The base lab teaches the basic pipeline. These exercises cover the TensorRT concepts most useful in real deployment.

---

## Exercise 1 – Dynamic Shapes and Batch Optimization

Current engine only supports:

```
batch = 1
```

Build an engine supporting:

```
batch = 1, 8, 16
```

Example:

```bash
trtexec \
 --onnx=resnet18.onnx \
 --minShapes=input:1x3x224x224 \
 --optShapes=input:8x3x224x224 \
 --maxShapes=input:16x3x224x224 \
 --fp16
```

Measure:

- latency
- throughput
- GPU utilization

Learn:

- latency vs throughput trade-offs
- production batching strategies

---

## Exercise 2 – TensorRT Profiling

Profile engine execution:

```bash
trtexec \
 --loadEngine=resnet18_fp16.engine \
 --dumpProfile
```

Analyze:

- slowest layers
- convolution bottlenecks
- GPU kernel efficiency

Goal:

Identify where optimization effort should go.

---

## Exercise 3 – Framework Benchmark Comparison

Compare the same model:

```
PyTorch CUDA
      |
ONNX Runtime CUDA
      |
TensorRT FP32
      |
TensorRT FP16
      |
TensorRT INT8
```

Create a final benchmark table:

| Backend | Precision | Latency | FPS |
|---|---|---:|---:|
| PyTorch | FP32 | | |
| ONNX Runtime | FP32 | | |
| TensorRT | FP16 | | |
| TensorRT | INT8 | | |

---

## Exercise 4 – Deploy Real-Time Camera Inference

Build:

```
Camera
  |
OpenCV
  |
TensorRT Engine
  |
Prediction
  |
FPS counter
```

Measure:

- End-to-end latency
- FPS
- GPU utilization
- Power consumption (Jetson)

This is the closest exercise to a real edge AI deployment.

---

# Final Learning Path

Recommended order:

1. ONNX export
2. TensorRT FP16 optimization
3. INT8 calibration
4. Dynamic shapes
5. TensorRT profiling
6. End-to-end deployment

After completing these, you should have the foundations needed for production TensorRT optimization.
