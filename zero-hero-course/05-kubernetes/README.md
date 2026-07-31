# Lab 35 — Deploy a PyTorch Model on Kubernetes with Minikube

**Goal:** Deploy a containerized PyTorch inference API to a local Kubernetes cluster using Minikube, expose it as a service, and scale it horizontally.

**Estimated Time:** 60–90 minutes

**Requirements:**

- Docker
- Minikube
- kubectl
- The containerized FastAPI application from **Lab 28**

---

# Learning Objectives

By the end of this lab you will be able to:

- Deploy a containerized AI application on Kubernetes
- Create Kubernetes Deployments and Services
- Expose an API to external users
- Scale an application using multiple replicas
- Inspect running workloads and logs

---

# Project Structure

```
.
├── app.py
├── model.pt
├── requirements.txt
├── Dockerfile
├── deployment.yaml
└── service.yaml
```

---

# Step 1 — Start Minikube

Start a local Kubernetes cluster:

```bash
minikube start --memory=4096 --cpus=4
```

Verify the cluster is running:

```bash
kubectl get nodes
```

Expected output:

```
NAME       STATUS   ROLES
minikube   Ready    control-plane
```

---

# Step 2 — Build the Container Image

Since Minikube uses its own Docker daemon, configure your shell to build images directly inside the cluster.

```bash
eval $(minikube docker-env)
```

Build the image:

```bash
docker build -t ai-model:latest .
```

Verify it exists:

```bash
docker images | grep ai-model
```

---

# Step 3 — Create a Deployment

Create **`deployment.yaml`**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-model-deployment

spec:
  replicas: 2

  selector:
    matchLabels:
      app: ai-model

  template:
    metadata:
      labels:
        app: ai-model

    spec:
      containers:
      - name: ai-model
        image: ai-model:latest

        ports:
        - containerPort: 8000
```

Deploy it:

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get deployments

kubectl get pods
```

Wait until all pods reach the **Running** state.

---

# Step 4 — Expose the Deployment

Create **`service.yaml`**.

```yaml
apiVersion: v1
kind: Service

metadata:
  name: ai-model-service

spec:
  selector:
    app: ai-model

  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000

  type: NodePort
```

Deploy the service:

```bash
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get services
```

Example output:

```
NAME                TYPE       PORT(S)
ai-model-service    NodePort   80:30080/TCP
```

---

# Step 5 — Access the API

Retrieve the service URL:

```bash
minikube service ai-model-service --url
```

Example:

```
http://127.0.0.1:30080
```

Open the FastAPI documentation:

```
http://127.0.0.1:30080/docs
```

Test the `/predict` endpoint by uploading an image.

---

# Step 6 — Scale the Deployment

Increase the number of replicas:

```bash
kubectl scale deployment ai-model-deployment --replicas=4
```

Verify:

```bash
kubectl get pods
```

You should now see four running pods.

---

# Step 7 — Inspect the Cluster

View all resources:

```bash
kubectl get all
```

Inspect a specific pod:

```bash
kubectl describe pod <POD_NAME>
```

View application logs:

```bash
kubectl logs <POD_NAME>
```

These logs are useful for debugging inference requests and container startup issues.

---

# Cleanup

Delete the Kubernetes resources:

```bash
kubectl delete -f service.yaml

kubectl delete -f deployment.yaml
```

Stop Minikube:

```bash
minikube stop
```

---

# Deliverables

Submit:

- Screenshot of `kubectl get pods`
- Screenshot of `kubectl get services`
- Screenshot of the FastAPI Swagger UI (`/docs`)
- Successful prediction using the `/predict` endpoint
- Screenshot after scaling to four replicas

---

# Summary

In this lab you learned how to:

- Build a Docker image for Kubernetes
- Deploy a containerized AI application
- Expose an application using a Kubernetes Service
- Scale a deployment horizontally
- Inspect pods, deployments, services, and logs
- Run an AI inference service on a local Kubernetes cluster