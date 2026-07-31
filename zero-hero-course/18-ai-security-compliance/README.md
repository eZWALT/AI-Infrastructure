# Lab 126 – Secure a Model Endpoint with Authentication

## 🎯 Goal

Take a plain FastAPI inference API and secure it with:

- JWT-based user authentication
- API Key authentication (for service-to-service calls)
- Rate limiting
- Deployment to Kubernetes with TLS support

---

# Step 0 — Prerequisites

- Python 3.10+
- (Optional) Kubernetes cluster with an Ingress controller (e.g., NGINX)
- Docker (if you want to containerize)

---

# Step 1 — Create the FastAPI App

Create the following file:

```
app/main.py
```

```python
from fastapi import FastAPI, Depends, HTTPException, status, Security
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm, APIKeyHeader
from pydantic import BaseModel
from jose import JWTError, jwt
from passlib.context import CryptContext
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi.responses import JSONResponse
import time

# FastAPI app + rate limiter
limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter

@app.exception_handler(RateLimitExceeded)
def rate_limit_handler(request, exc):
    return JSONResponse(status_code=429, content={"detail": "Rate limit exceeded"})

# Demo secrets
SECRET_KEY = "change-me"
ALGORITHM = "HS256"
API_KEY = "super-secret"

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Demo users
fake_users = {
    "alice": {"username": "alice", "hashed_pw": pwd_context.hash("alicepass")},
    "admin": {"username": "admin", "hashed_pw": pwd_context.hash("adminpass")}
}

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

class Token(BaseModel):
    access_token: str
    token_type: str

class Features(BaseModel):
    features: list[float]

def authenticate_user(username, password):
    user = fake_users.get(username)
    if not user or not pwd_context.verify(password, user["hashed_pw"]):
        return None
    return user

def create_access_token(data: dict, expires_in: int = 3600):
    payload = data.copy()
    payload.update({"exp": time.time() + expires_in})
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload.get("sub")
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

async def get_api_key(api_key: str = Security(api_key_header)):
    if api_key == API_KEY:
        return api_key
    raise HTTPException(status_code=403, detail="Invalid API key")

# Routes
@app.post("/token", response_model=Token)
async def login(form: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form.username, form.password)
    if not user:
        raise HTTPException(status_code=401, detail="Bad credentials")
    token = create_access_token({"sub": form.username})
    return {"access_token": token, "token_type": "bearer"}

@app.get("/healthz")
def health():
    return {"status": "ok"}

@app.get("/readyz")
def ready():
    return {"status": "ready"}

@app.post("/predict")
@limiter.limit("30/minute")
async def predict(
    data: Features,
    user: str = Depends(get_current_user),
    api_key: str = Depends(get_api_key)
):
    # Demo "model": sum of features
    score = sum(data.features)
    return {"prediction": "class_A" if score > 5 else "class_B"}
```

---

# Step 2 — Install Dependencies

```bash
pip install fastapi uvicorn python-jose passlib[bcrypt] slowapi
```

---

# Step 3 — Run Locally

```bash
export JWT_SECRET='change-me'
export API_KEY='super-secret'

uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Test the API:

```bash
curl -s http://localhost:8000/healthz
```

---

# Step 4 — Get a JWT

```bash
curl -s -X POST http://localhost:8000/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=alice&password=alicepass"
```

The response will contain a JWT access token.

---

# Step 5 — Call the Protected Endpoint

## Option A — Bearer Token

```bash
TOKEN="<paste JWT>"

curl -s -X POST http://localhost:8000/predict \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"features":[5.1,3.5,1.4,0.2]}'
```

---

## Option B — API Key

```bash
curl -s -X POST http://localhost:8000/predict \
  -H "X-API-Key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"features":[6.0,2.2,4.0,1.0]}'
```

---

# Step 6 — Rate Limiting

Configuration:

- Global default: **60 requests/minute**
- `/predict`: **30 requests/minute**

If exceeded, the API returns:

```
429 Too Many Requests
```

---

# Step 7 — Containerize

Create a `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY app/ app/

RUN pip install fastapi uvicorn python-jose passlib[bcrypt] slowapi

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Build the image:

```bash
docker build -t secure-endpoint .
```

Run the container:

```bash
docker run \
  -p 8000:8000 \
  -e JWT_SECRET=change-me \
  -e API_KEY=super-secret \
  secure-endpoint
```

---

# Step 8 — Deploy to Kubernetes

Create:

```
k8s/deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-ml
  namespace: ai-sec

spec:
  replicas: 1

  selector:
    matchLabels:
      app: secure-ml

  template:
    metadata:
      labels:
        app: secure-ml

    spec:
      containers:
      - name: api
        image: YOUR_REGISTRY/secure-endpoint:latest

        ports:
        - containerPort: 8000

        envFrom:
        - secretRef:
            name: jwt-secret

        - secretRef:
            name: api-key

---

apiVersion: v1
kind: Service

metadata:
  name: secure-ml-svc
  namespace: ai-sec

spec:
  selector:
    app: secure-ml

  ports:
  - port: 80
    targetPort: 8000
```

Create the namespace and secrets:

```bash
kubectl create namespace ai-sec

kubectl -n ai-sec create secret generic jwt-secret \
  --from-literal=JWT_SECRET='change-me'

kubectl -n ai-sec create secret generic api-key \
  --from-literal=API_KEY='super-secret'
```

Deploy:

```bash
kubectl -n ai-sec apply -f k8s/deployment.yaml
```

Optionally, expose the service through an Ingress with TLS.

---

# Step 9 — Hardening Checklist

Before using this architecture in production:

- Use OIDC (Auth0, AWS Cognito, Microsoft Entra ID, Keycloak, etc.) instead of the demo `/token` endpoint.
- Enforce HTTPS at the Ingress (consider mutual TLS internally).
- Store secrets in Kubernetes Secrets or Vault rather than hardcoding them.
- Apply Kubernetes RBAC and namespace isolation.
- Protect against DoS attacks with WAF rules and appropriate rate limits.
- Log request IDs, authenticated users, and model versions, then forward logs to a SIEM platform.

---

# ✅ End Result

By completing this lab you have:

- Secured a FastAPI model endpoint with JWT authentication
- Added API Key authentication for service-to-service communication
- Implemented request rate limiting
- Containerized the application with Docker
- Prepared a Kubernetes deployment with Secrets
- Learned the basic security building blocks for deploying ML inference APIs

# 🚀 Optional Extension Exercises (High ROI)

These exercises build directly on the lab above without requiring any additional infrastructure.

---

# Extension 1 — Verify Authentication Failures

Understand how the API behaves when credentials are missing or invalid.

### No JWT

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"features":[1,2,3,4]}'
```

Expected:

```
401 Unauthorized
```

---

### Invalid JWT

```bash
curl -X POST http://localhost:8000/predict \
  -H "Authorization: Bearer invalid.token.here" \
  -H "X-API-Key: super-secret" \
  -H "Content-Type: application/json" \
  -d '{"features":[1,2,3,4]}'
```

Expected:

```
401 Unauthorized
```

---

### Invalid API Key

```bash
curl -X POST http://localhost:8000/predict \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-API-Key: wrong-key" \
  -H "Content-Type: application/json" \
  -d '{"features":[1,2,3,4]}'
```

Expected:

```
403 Forbidden
```

---

## What you learned

- Difference between authentication and authorization
- JWT validation
- API key validation

---

# Extension 2 — Test Rate Limiting

Verify that SlowAPI protects your endpoint.

Install ApacheBench if needed.

Ubuntu:

```bash
sudo apt install apache2-utils
```

macOS:

```bash
brew install httpd
```

Run:

```bash
ab -n 100 -c 20 \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-API-Key: super-secret" \
  -p payload.json \
  -T application/json \
  http://localhost:8000/predict
```

where `payload.json` contains

```json
{
    "features":[1,2,3,4]
}
```

Eventually you'll receive

```
429 Too Many Requests
```

---

## What you learned

- API protection against abuse
- Basic load testing
- HTTP 429 behavior

---

# Extension 3 — Use Environment Variables Properly

Instead of hardcoding secrets:

```python
SECRET_KEY = "change-me"
API_KEY = "super-secret"
```

replace them with

```python
import os

SECRET_KEY = os.getenv("JWT_SECRET")
API_KEY = os.getenv("API_KEY")
```

Run:

```bash
export JWT_SECRET=my-secret
export API_KEY=my-api-key

uvicorn app.main:app
```

Verify everything still works.

---

## What you learned

- Twelve-Factor App principles
- Production-ready secret management
- Environment configuration

---

# Extension 4 — Add Interactive API Documentation

FastAPI automatically generates Swagger UI.

Open

```
http://localhost:8000/docs
```

and

```
http://localhost:8000/redoc
```

Log in to obtain a JWT.

Authorize inside Swagger.

Call `/predict` directly from the browser.

---

## What you learned

- Interactive API documentation
- Testing secured endpoints without curl
- OpenAPI support

---

# Extension 5 — Inspect the JWT

Copy the JWT returned by

```
POST /token
```

Decode it locally.

Option 1:

```
https://jwt.io
```

Option 2:

```python
from jose import jwt

decoded = jwt.decode(
    TOKEN,
    "change-me",
    algorithms=["HS256"]
)

print(decoded)
```

Observe fields such as

- subject (`sub`)
- expiration (`exp`)

Modify the expiration time in

```python
create_access_token(...)
```

Generate another token and compare.

---

## What you learned

- JWT structure
- Claims
- Token expiration
- Signature verification
