# Lab 147: Deploy YOLOv5 on Jetson Nano

## 🎯 Goal
Deploy a real-time object detection model (YOLOv5) on an edge device (Jetson Nano or local Linux/GPU host), optimized with NVIDIA TensorRT for high-performance inference.

---

## 0. Prerequisites

* **Target Device:** Jetson Nano Developer Kit (4GB preferred) OR standard Linux machine / Google Colab / WSL2 with an NVIDIA GPU.
* **OS / Environment:** JetPack SDK installed (for Jetson) or Ubuntu Linux 20.04/22.04 with CUDA support.
* **Storage:** 16–32 GB microSD card / eMMC or available disk space.
* **Input Source:** USB webcam, CSI camera (e.g., Raspberry Pi Camera v2), or sample video/image files.
* Internet connectivity for cloning repositories and downloading pretrained weights.

---

## 1. Environment Setup

Update your host system and install baseline dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip git
```

Set up an isolated Python virtual environment:

```bash
python3 -m venv yolov5-env
source yolov5-env/bin/activate
```

Upgrade package management utilities and install primary ML libraries:

```bash
pip install --upgrade pip setuptools wheel
pip install numpy torch torchvision matplotlib onnx onnxruntime
```

---

## 2. Clone YOLOv5 Repository

Retrieve the official Ultralytics YOLOv5 codebase and install required Python dependencies:

```bash
git clone https://github.com/ultralytics/yolov5.git
cd yolov5
pip install -r requirements.txt
```

---

## 3. Test Inference (PyTorch Engine)

Execute a baseline detection test using PyTorch and the lightweight `yolov5s.pt` model weights:

```bash
python detect.py --source 0 --weights yolov5s.pt --conf 0.4
```

* `--source 0`: Captures input from a connected webcam (Replace `0` with a path to an image or video file if running headlessly or without a webcam, e.g., `--source data/images`).
* `--weights yolov5s.pt`: Uses the small YOLOv5 model variant for speed.
* `--conf 0.4`: Sets the detection confidence threshold to 40%.

---

## 4. Export Model to ONNX Format

Convert the PyTorch model (`.pt`) into an open hardware-agnostic intermediate format (**ONNX**):

```bash
python export.py --weights yolov5s.pt --include onnx
```

* **Output file:** `yolov5s.onnx`

---

## 5. Optimize Model with TensorRT

Convert and compile the model into an optimized NVIDIA **TensorRT** runtime engine (`.engine`):

```bash
python export.py --weights yolov5s.pt --include engine
```

* **Output file:** `yolov5s.engine`

---

## 6. Run Inference with TensorRT Engine

Execute object detection utilizing the compiled TensorRT engine:

```bash
python detect.py --source 0 --weights yolov5s.engine --conf 0.4
```

> ⚡ **Expected Speedup:** Up to 2×–4× performance increase compared to standard PyTorch inference on Jetson edge devices.

---

## 7. Performance Benchmarking

Measure and compare execution throughput across models:

```bash
# Test TensorRT Engine FPS
python detect.py --source data/images --weights yolov5s.engine --conf 0.4 --save-txt

# Test Baseline PyTorch FPS
python detect.py --source data/images --weights yolov5s.pt --conf 0.4 --save-txt
```

### Benchmark Comparison Example (Jetson Nano)
| Backend | Average FPS | Optimization Level |
| :--- | :--- | :--- |
| **PyTorch (`.pt`)** | ~5 FPS | Unoptimized Baseline |
| **TensorRT (`.engine`)** | ~15 FPS | Quantized & Layer-Fused |

---

## 8. (Optional) Expose Detections via FastAPI Endpoint

Install lightweight web server components:

```bash
pip install fastapi uvicorn
```

Create an `app.py` web service script:

```python
from fastapi import FastAPI, UploadFile
import torch

app = FastAPI()

# Load compiled TensorRT or PyTorch model via torch.hub
model = torch.hub.load('ultralytics/yolov5', 'custom', path='yolov5s.engine')

@app.post("/predict")
async def predict(file: UploadFile):
    img = await file.read()
    results = model(img)
    # Return bounding box coordinates [xmin, ymin, xmax, ymax, confidence, class]
    return results.pandas().xyxy[0].to_dict(orient="records")
```

Launch the REST API server:

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

---

## 9. Cleanup

Deactivate the environment and release GPU memory resources:

```bash
deactivate
```
*(Press `Ctrl+C` to terminate active camera streams or uvicorn web servers).*

---

## ✅ Key Learning Outcomes

- [x] Configured YOLOv5 object detection model pipelines on an edge device profile.
- [x] Converted standard PyTorch models (`.pt`) to intermediate **ONNX** & hardware-optimized **TensorRT** formats.
- [x] Quantified and benchmarked inference speed gains (FPS).
- [x] Built a production-ready **FastAPI** web endpoint for remote image predictions.
