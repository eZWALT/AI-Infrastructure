# Day 245 Lab: Quantize a Vision Model

## Learning Goals
* Understand **post-training quantization (PTQ)** in PyTorch
* Apply INT8 quantization to a ResNet model
* Compare inference latency and accuracy vs FP32 baseline
* Export quantized model for deployment

---

## 0. Prerequisites
* Python 3.9+, PyTorch ≥ 1.13
* Install dependencies:

```bash
pip install torch torchvision timm
```

> **Note:** GPU is optional. Quantization also runs on CPU.

---

## 1. Load Pretrained Vision Model
Create a script to load the baseline model:

```python
import torch
import torchvision.models as models
import torchvision.transforms as T
from PIL import Image

# Load ResNet-18 pretrained on ImageNet
model_fp32 = models.resnet18(pretrained=True)
model_fp32.eval()
```

---

## 2. Prepare Input Transform & Sample Image
Prepare image pre-processing steps and a sample input:

```python
transform = T.Compose([
    T.Resize(256),
    T.CenterCrop(224),
    T.ToTensor(),
    T.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

img = Image.open("sample.jpg")
x = transform(img).unsqueeze(0)  # Batch size = 1
```

---

## 3. Run Baseline FP32 Inference
Benchmark standard FP32 model latency:

```python
import time

with torch.no_grad():
    start = time.time()
    for _ in range(100):
        _ = model_fp32(x)
    end = time.time()

print("FP32 Avg Latency (ms):", (end - start) / 100 * 1000)
```

---

## 4. Apply Dynamic Quantization (INT8)
Quantize `nn.Linear` layers dynamically to INT8 precision:

```python
import torch.quantization

# Quantize Linear layers to INT8
model_int8 = torch.quantization.quantize_dynamic(
    model_fp32, {torch.nn.Linear}, dtype=torch.qint8
)
model_int8.eval()

with torch.no_grad():
    start = time.time()
    for _ in range(100):
        _ = model_int8(x)
    end = time.time()

print("INT8 Avg Latency (ms):", (end - start) / 100 * 1000)
```

---

## 5. Compare Model Sizes
Save checkpoints to disk and measure memory reduction:

```python
import os

torch.save(model_fp32.state_dict(), "resnet_fp32.pth")
torch.save(model_int8.state_dict(), "resnet_int8.pth")

print("FP32 Size (MB):", os.path.getsize("resnet_fp32.pth") / 1e6)
print("INT8 Size (MB):", os.path.getsize("resnet_int8.pth") / 1e6)
```

---

## 6. Validate Accuracy (Optional – on Dataset)
Run accuracy evaluation on the CIFAR-10 test split:

```python
from torchvision.datasets import CIFAR10
from torch.utils.data import DataLoader

test_data = CIFAR10(root="./data", train=False, transform=transform, download=True)
test_loader = DataLoader(test_data, batch_size=32)

def evaluate(model):
    correct, total = 0, 0
    with torch.no_grad():
        for images, labels in test_loader:
            outputs = model(images)
            preds = outputs.argmax(1)
            correct += (preds == labels).sum().item()
            total += labels.size(0)
    return correct / total

acc_fp32 = evaluate(model_fp32)
acc_int8 = evaluate(model_int8)

print("FP32 Accuracy:", acc_fp32)
print("INT8 Accuracy:", acc_int8)
```

---

## 7. Export Quantized Model for Deployment
Export the quantized PyTorch model to ONNX format:

```python
dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(model_int8, dummy_input, "resnet18_int8.onnx", opset_version=13)
print("Quantized model exported to ONNX")
```

---

## 8. Stretch Goals
* Try **static quantization** with a calibration dataset
* Use **QAT (Quantization-Aware Training)** for better accuracy retention
* Benchmark on GPU vs CPU for throughput differences
* Deploy the ONNX model using **ONNX Runtime / TensorRT**

---

> **Outcome:** You quantized a pretrained ResNet-18 model, measured latency and size improvements, and validated accuracy. The INT8 model runs faster and smaller, making it ready for efficient deployment.
