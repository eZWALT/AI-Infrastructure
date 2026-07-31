# 🌍 Lab 140 – Serve a Model Across AWS + GCP

## 🎯 Goal

Deploy the same AI inference service across **two cloud providers**:

- AWS EKS
- GCP GKE

Expose both deployments behind a global DNS layer that routes users to the closest healthy endpoint.

You will learn:

- Multi-cloud Kubernetes deployments
- Container registry management
- Cloud load balancers
- Global DNS routing
- High availability inference architecture
- Failover testing

**Estimated Time:** 3–5 hours

**Tools**

- Docker
- Kubernetes
- AWS EKS
- GCP GKE
- Route53 / Cloudflare
- kubectl
- Helm
- eksctl
- gcloud CLI

---

# 0️⃣ Prerequisites

You need:

## Cloud accounts

- AWS account with permissions to create EKS clusters
- GCP account with permissions to create GKE clusters

## Installed tools

Verify:

```bash
docker --version

kubectl version --client

helm version

eksctl version

gcloud --version
```

Configure AWS:

```bash
aws configure
```

Configure GCP:

```bash
gcloud auth login
```

Set your project:

```bash
gcloud config set project <PROJECT_ID>
```

---

# 1️⃣ Build and Push Docker Image

Use an existing inference API from previous labs.

Example:

```
FastAPI inference service
        |
        v
Docker image
        |
        +---- AWS ECR
        |
        +---- GCP Artifact Registry
```

---

## Build image

```bash
docker build \
-t fastapi-inference:multi .
```

Test locally:

```bash
docker run -p 8000:8000 fastapi-inference:multi
```

Verify:

```bash
curl localhost:8000/healthz
```

---

# 2️⃣ Push Image to AWS ECR

Create repository:

```bash
aws ecr create-repository \
--repository-name fastapi-inference
```

Authenticate Docker:

```bash
aws ecr get-login-password \
--region us-east-1 \
| docker login \
--username AWS \
--password-stdin <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com
```

Tag image:

```bash
AWS_URI=<ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/fastapi-inference:latest


docker tag \
fastapi-inference:multi \
$AWS_URI
```

Push:

```bash
docker push $AWS_URI
```

---

# 3️⃣ Push Image to GCP Artifact Registry

Create repository:

```bash
gcloud artifacts repositories create inference \
--repository-format=docker \
--location=us-central1
```

Authenticate:

```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

Tag:

```bash
GCP_URI=us-central1-docker.pkg.dev/<PROJECT_ID>/inference/fastapi-inference:latest


docker tag \
fastapi-inference:multi \
$GCP_URI
```

Push:

```bash
docker push $GCP_URI
```

---

# 4️⃣ Create Kubernetes Clusters

## AWS EKS

Create cluster:

```bash
eksctl create cluster \
--name ai-eks \
--region us-east-1 \
--nodes 3
```

Verify:

```bash
kubectl get nodes
```

---

## GCP GKE

Create cluster:

```bash
gcloud container clusters create ai-gke \
--region us-central1 \
--num-nodes 3
```

Connect:

```bash
gcloud container clusters get-credentials ai-gke \
--region us-central1
```

Verify:

```bash
kubectl get nodes
```

---

# 5️⃣ Deploy the Inference API

Create:

```
deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: inference-api

spec:
  replicas: 3

  selector:
    matchLabels:
      app: inference-api

  template:

    metadata:
      labels:
        app: inference-api

    spec:

      containers:

      - name: api

        image: <CLOUD_IMAGE_URI>

        ports:
        - containerPort: 8000

        readinessProbe:
          httpGet:
            path: /readyz
            port: 8000

        livenessProbe:
          httpGet:
            path: /healthz
            port: 8000
```

Replace:

```
<CLOUD_IMAGE_URI>
```

with:

AWS:

```
AWS_URI
```

or:

GCP:

```
GCP_URI
```

---

Create:

```
service.yaml
```

```yaml
apiVersion: v1
kind: Service

metadata:
  name: inference-svc

spec:

  type: LoadBalancer

  selector:
    app: inference-api

  ports:

  - port: 80
    targetPort: 8000
```

---

Deploy on AWS:

```bash
kubectl apply -f deployment.yaml

kubectl apply -f service.yaml
```

Deploy on GCP:

Repeat after switching cluster context.

---

# 6️⃣ Obtain Cloud Load Balancers

AWS:

```bash
kubectl get svc inference-svc
```

Example:

```
EXTERNAL-IP:
a1b2c3.amazonaws.com
```

GCP:

```bash
kubectl get svc inference-svc
```

Example:

```
EXTERNAL-IP:
34.12.45.67
```

Save both endpoints.

---

# 7️⃣ Configure Global DNS Routing

Use:

- AWS Route53
- Cloudflare
- Google Cloud DNS

Create:

```
inference.example.com
```

with latency-based routing.

Example:

```
inference.example.com

        |
        |
   Global DNS
        |
  ----------------
  |              |
 AWS EKS       GKE
 us-east-1     us-central1
```

Example records:

```
Record 1:

Name:
inference.example.com

Target:
AWS Load Balancer

Region:
us-east-1
```

```
Record 2:

Name:
inference.example.com

Target:
GCP Load Balancer

Region:
us-central1
```

---

# 8️⃣ Test Multi-Cloud Inference

Health check:

```bash
curl \
http://inference.example.com/healthz
```

Prediction:

```bash
curl \
-X POST \
http://inference.example.com/predict \
-H "Content-Type: application/json" \
-d '{"features":[5.1,3.5,1.4,0.2]}'
```

Verify:

- Response is successful
- Requests reach healthy clusters

---

# 9️⃣ Simulate Failure

Delete AWS pods:

```bash
kubectl delete pods \
-l app=inference-api
```

Check:

```bash
kubectl get pods
```

The Kubernetes deployment should recreate them.

For a stronger test:

Scale AWS deployment to zero:

```bash
kubectl scale deployment inference-api \
--replicas=0
```

Verify DNS routing moves traffic elsewhere.

---

# 🔟 Cleanup

AWS:

```bash
eksctl delete cluster \
--name ai-eks
```

GCP:

```bash
gcloud container clusters delete ai-gke \
--region us-central1
```

Delete:

- Load balancers
- DNS records
- Container registries

---

# 🎯 Learning Outcomes

After completing this lab you will understand:

- Multi-cloud AI serving architectures
- Running inference on EKS and GKE
- Container registry workflows
- Kubernetes service exposure
- Global DNS routing
- High availability inference
- Cloud failover strategies

---

# 🚀 Optional Extensions

## Extension 1 – Add Health-Based Routing

Instead of latency routing:

Route only to healthy endpoints.

Use:

- Route53 health checks
- Cloudflare health monitors

---

## Extension 2 – Add Observability

Deploy:

- Prometheus
- Grafana

Track:

- Requests/sec
- Latency
- Error rate
- CPU usage

---

## Extension 3 – Add Canary Releases

Deploy:

```
v1 inference-api
v2 inference-api
```

Route:

```
90% → stable
10% → canary
```

Validate before full rollout.

---

## Extension 4 – Add GPU Nodes

Modify clusters:

AWS:

```text
g4dn / g5 nodes
```

GCP:

```text
NVIDIA T4 / L4 nodes
```

Deploy GPU inference workloads.

