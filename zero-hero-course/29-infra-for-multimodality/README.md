# Day 203 Lab – Deploy CLIP for Image Search

## Learning Goals

* Encode images & text into a **shared embedding space**
* Store image embeddings in **FAISS** (vector database)
* Query with natural language to retrieve similar images
* Deploy as a **FastAPI microservice**

---

## 0) Prerequisites

Make sure you have the following ready:

* **Python 3.9+**

### Install Dependencies

Run these commands in your terminal:

```bash
mkdir clip-search-lab && cd clip-search-lab
python -m venv .venv

# Activate virtual environment
# On macOS/Linux:
source .venv/bin/activate
# On Windows:
# .venv\Scripts\activate

pip install torch torchvision faiss-cpu fastapi uvicorn pillow transformers
```

---

## 1) Load CLIP Model

Create `clip_model.py`:

```python
import torch
from PIL import Image
from transformers import CLIPProcessor, CLIPModel

MODEL_NAME = "openai/clip-vit-base-patch32"
device = "cuda" if torch.cuda.is_available() else "cpu"

model = CLIPModel.from_pretrained(MODEL_NAME).to(device)
processor = CLIPProcessor.from_pretrained(MODEL_NAME)

def embed_image(image_path: str):
    img = Image.open(image_path).convert("RGB")
    inputs = processor(images=img, return_tensors="pt").to(device)
    with torch.no_grad():
        emb = model.get_image_features(**inputs)
    return emb.cpu().numpy()

def embed_text(query: str):
    inputs = processor(text=[query], return_tensors="pt", padding=True).to(device)
    with torch.no_grad():
        emb = model.get_text_features(**inputs)
    return emb.cpu().numpy()
```

---

## 2) Build Image Index with FAISS

Create `build_index.py`:

```python
import os
import faiss
import pickle
import numpy as np
from clip_model import embed_image

image_folder = "images"   # Put your sample JPG/PNG images here
image_paths = [os.path.join(image_folder, f) for f in os.listdir(image_folder)]

embs, metadata = [], []
for path in image_paths:
    embs.append(embed_image(path))
    metadata.append(path)

embs = np.vstack(embs).astype("float32")

index = faiss.IndexFlatL2(embs.shape[1])
index.add(embs)

with open("clip_index.pkl", "wb") as f:
    pickle.dump((index, metadata), f)

print(f"Indexed {len(metadata)} images.")
```

### Run Indexer

```bash
python build_index.py
```

---

## 3) Create Search Function

Create `search.py`:

```python
import faiss
import pickle
import numpy as np
from clip_model import embed_text

with open("clip_index.pkl", "rb") as f:
    index, metadata = pickle.load(f)

def search_images(query: str, k=3):
    qvec = embed_text(query).astype("float32")
    D, I = index.search(qvec, k)
    results = [metadata[i] for i in I[0]]
    return results
```

### Test in Python REPL

```python
from search import search_images
print(search_images("a dog playing in the park"))
```

---

## 4) Deploy FastAPI Service

Create `server.py`:

```python
from fastapi import FastAPI
from pydantic import BaseModel
from search import search_images

app = FastAPI()

class Query(BaseModel):
    text: str

@app.post("/search/")
def search(query: Query):
    results = search_images(query.text, k=5)
    return {"query": query.text, "results": results}
```

### Run API Server

```bash
uvicorn server:app --reload --port 8000
```

---

## 5) Test API with `curl`

In a separate terminal tab, send a POST request with your query:

```bash
curl -X POST "http://127.0.0.1:8000/search/"   -H "Content-Type: application/json"   -d '{"text": "a man riding a bicycle"}'
```

### Sample Response

```json
{
  "query": "a man riding a bicycle",
  "results": ["images/bike1.jpg", "images/bike2.jpg", "images/bike3.jpg"]
}
```

---

## 6) Stretch Goals

* Add an **image upload endpoint** to dynamically insert and index new images.
* Store embeddings in **Weaviate** or **Pinecone** for scale-out vector search.
* Build a **Streamlit UI** to visualize retrieved image results side-by-side.
* Quantize the CLIP model for faster CPU / edge inference.

> ✅ **Outcome:** You built and deployed a CLIP-powered image search API, capable of finding images based on natural language queries.
