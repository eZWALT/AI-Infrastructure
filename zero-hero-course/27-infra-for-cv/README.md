# Day 189 Lab – Deploy Real-Time Object Detection

## Learning Goals

* Run a pretrained object detection model (**YOLOv8**)
* Perform inference on live video or webcam stream
* Deploy a **FastAPI inference server** for real-time detection
* Visualize detections with bounding boxes

---

## 0) Prerequisites

Make sure you have the following ready:

* **Python 3.9+**
* **GPU recommended** (CUDA device or Google Colab)

### Install Dependencies

Run these commands in your terminal:

```bash
mkdir object-detection-lab && cd object-detection-lab
python -m venv .venv

# Activate virtual environment
# On macOS/Linux:
source .venv/bin/activate
# On Windows:
# .venv\Scripts\activate

pip install ultralytics fastapi uvicorn opencv-python-headless python-multipart
```

---

## 1) Run YOLOv8 Locally (Quick Test)

Create a test script or run in Python REPL:

```python
from ultralytics import YOLO

# Load pretrained COCO model
model = YOLO("yolov8n.pt")  # n = nano, fast & lightweight

# Run inference on an image
results = model.predict("https://ultralytics.com/images/bus.jpg", show=True)

for r in results:
    print(r.boxes.xyxy)  # print bounding boxes
```

> ✅ **Outcome:** You should see a bus, people, and objects detected.

---

## 2) Real-Time Webcam Detection

Create `webcam.py`:

```python
import cv2
from ultralytics import YOLO

model = YOLO("yolov8n.pt")
cap = cv2.VideoCapture(0)  # 0 = default webcam

while True:
    ret, frame = cap.read()
    if not ret:
        break
    results = model(frame)
    annotated = results[0].plot()
    cv2.imshow("YOLOv8 Real-Time", annotated)
    if cv2.waitKey(1) & 0xFF == ord("q"):
        break

cap.release()
cv2.destroyAllWindows()
```

### Run Webcam Script

```bash
python webcam.py
```

> **Note:** Press `q` in the video window to quit.

---

## 3) Deploy FastAPI Inference Server

Create `server.py`:

```python
from fastapi import FastAPI, UploadFile, File
from ultralytics import YOLO
import cv2
import numpy as np

app = FastAPI()
model = YOLO("yolov8n.pt")

@app.post("/detect/")
async def detect(file: UploadFile = File(...)):
    contents = await file.read()
    nparr = np.frombuffer(contents, np.uint8)
    img = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
    results = model(img)
    detections = []
    for r in results:
        for box in r.boxes:
            detections.append({
                "class": model.names[int(box.cls)],
                "confidence": float(box.conf),
                "bbox": box.xyxy[0].tolist()
            })
    return {"detections": detections}
```

### Start the FastAPI Server

```bash
uvicorn server:app --reload --port 8000
```

---

## 4) Test API with `curl`

In a separate terminal tab, send a POST request with an image file:

```bash
curl -X POST "http://127.0.0.1:8000/detect/"   -F "file=@bus.jpg"
```

### Expected Output (Sample)

```json
{
  "detections": [
    {"class": "bus", "confidence": 0.89, "bbox": [34, 56, 280, 190]},
    {"class": "person", "confidence": 0.77, "bbox": [310, 80, 400, 250]}
  ]
}
```

---

## 5) Optional — Streamlit UI for Easy Testing

Install Streamlit:

```bash
pip install streamlit
```

Create `app.py`:

```python
import streamlit as st
from ultralytics import YOLO
import cv2
import numpy as np

model = YOLO("yolov8n.pt")
st.title("Real-Time Object Detection")

uploaded = st.file_uploader("Upload an image", type=["jpg", "png", "jpeg"])
if uploaded:
    file_bytes = np.asarray(bytearray(uploaded.read()), dtype=np.uint8)
    img = cv2.imdecode(file_bytes, 1)
    results = model(img)
    st.image(results[0].plot(), channels="BGR")
```

### Run Streamlit Web App

```bash
streamlit run app.py
```

---

## 6) Stretch Goals

* Switch to **YOLOv8m** or **YOLOv8l** for higher detection accuracy.
* Connect FastAPI with **Kafka / MQTT** for edge-to-cloud video streaming.
* Deploy with **Triton Inference Server** for high-throughput GPU optimization.
* Quantize model weights to **INT8** to run efficiently on Jetson or edge devices.

> ✅ **Final Outcome:** You deployed a real-time object detection pipeline accessible via webcam, REST API, or web UI.
