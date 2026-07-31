# 🧪 Lab 91 – Configure Load Balancer for AI API

## Goal

Expose an AI inference API through a resilient, scalable load balancer with:

- L4 Load Balancer (Service)
- L7 Load Balancer (Ingress)
- Health checks
- Autoscaling
- Rate limiting
- Canary deployment
- Basic observability

**Outcome:** A production-style Kubernetes deployment for an AI inference API.

**Estimated Time:** 2–3 hours

---

# 0️⃣ Prerequisites

You'll need:

- Kubernetes cluster (Kind, Minikube, k3s, EKS, GKE or AKS)
- `kubectl`
- NGINX Ingress Controller (recommended)
- Optional:
  - DNS name
  - TLS certificate
  - `hey` or `wrk` for load testing

---

# 1️⃣ Architecture

Deploy the following components yourself:

- FastAPI inference API
- Deployment
- ClusterIP Service
- LoadBalancer Service
- Horizontal Pod Autoscaler
- NGINX Ingress
- Canary Ingress
- (Optional) Envoy Proxy

Your architecture should look like:

```text
                Internet
                    │
          LoadBalancer Service
                    │
             NGINX Ingress
             /            \
     Stable Pods      Canary Pods
           │               │
      ClusterIP Service
           │
     FastAPI Deployment
```

---

# 2️⃣ Build the Kubernetes Manifests

Create the following YAML files manually:

```
k8s/
├── namespace.yaml
├── deployment.yaml
├── service-clusterip.yaml
├── service-loadbalancer.yaml
├── ingress.yaml
├── ingress-canary.yaml
├── hpa.yaml
└── configmap.yaml (optional)
```

Deploy them:

```bash
kubectl apply -f k8s/
```

Verify:

```bash
kubectl get all -n ai

kubectl get ingress -n ai

kubectl get hpa -n ai
```

---

# 3️⃣ Test Locally

Before testing the Load Balancer, verify the application.

```bash
kubectl port-forward svc/inference-svc 8080:80
```

Health endpoint:

```bash
curl http://localhost:8080/healthz
```

Inference endpoint:

```bash
curl -X POST \
http://localhost:8080/v1/echo \
-H "Content-Type: application/json" \
-d '{"text":"hello"}'
```

---

# 4️⃣ Configure an L4 Load Balancer

Create a Service of type:

```yaml
type: LoadBalancer
```

Wait until an external IP appears.

```bash
kubectl get svc
```

Test:

```bash
curl http://<EXTERNAL-IP>/healthz
```

---

# 5️⃣ Configure an L7 Load Balancer (Ingress)

Create an Ingress resource that provides:

- Path routing
- TLS termination (optional)
- Rate limiting

Useful annotations include:

- `nginx.ingress.kubernetes.io/limit-rps`
- `nginx.ingress.kubernetes.io/proxy-read-timeout`
- `nginx.ingress.kubernetes.io/proxy-send-timeout`

Test:

```bash
curl https://api.example.com/healthz
```

---

# 6️⃣ Configure a Canary Deployment

Deploy a second version of the application.

Create a Canary Ingress that routes approximately **10%** of traffic to the new version.

Useful annotations:

```text
nginx.ingress.kubernetes.io/canary
nginx.ingress.kubernetes.io/canary-weight
```

Generate requests and verify both versions receive traffic.

---

# 7️⃣ Configure Autoscaling

Create an HPA with:

- Minimum replicas: 2
- Maximum replicas: 50
- Target CPU: 70%

Watch scaling events:

```bash
kubectl get hpa -w
```

Generate load to trigger scaling.

---

# 8️⃣ (Optional) Deploy Envoy

Instead of relying only on NGINX, deploy Envoy as an L7 proxy.

Configure:

- Least Request load balancing
- Retries
- Circuit breaking
- Outlier detection

Compare its behavior with NGINX.

---

# 9️⃣ Validate Resilience

Verify:

- `/healthz`
- `/readyz`

Delete a running pod:

```bash
kubectl delete pod -l app=inference-api --wait=false
```

Confirm that requests continue succeeding.

---

# 🔟 Load Testing

Choose one tool.

### hey

```bash
hey \
-z 60s \
-q 50 \
-c 100 \
-m POST \
-H "Content-Type: application/json" \
-d '{"text":"load"}' \
https://api.example.com/v1/echo
```

### wrk

```bash
wrk \
-t4 \
-c100 \
-d60s \
-s post.lua \
https://api.example.com/v1/echo
```

Record:

- Requests/sec
- p50 latency
- p95 latency
- p99 latency
- Error rate

---

# 1️⃣1️⃣ Observability

Collect basic metrics:

- Requests/sec
- Error rate
- p50 latency
- p95 latency
- p99 latency

(Optional)

Integrate:

- Prometheus
- Grafana
- OpenTelemetry

---

# 1️⃣2️⃣ Cleanup

```bash
kubectl delete -f k8s/
```

---

# 📋 What to Capture

Include in your report:

- Kubernetes architecture
- External LoadBalancer endpoint
- Ingress configuration
- HPA scaling screenshots
- Canary deployment behavior
- Load testing results
- Failure recovery after deleting a pod

---

# 🎯 Learning Outcomes

By completing this lab you will be able to:

- Deploy an AI inference API on Kubernetes
- Configure L4 and L7 load balancing
- Implement health checks and readiness probes
- Configure autoscaling with HPA
- Perform canary deployments
- Apply rate limiting at the Ingress
- Validate resilience during failures
- Benchmark inference latency under load
