# Lab 133: Build a High-Availability AI Inference Cluster

## 🎯 Goal
Deploy a **multi-replica, fault-tolerant AI inference service** on Kubernetes featuring:
* Load balancing
* Auto-healing pods
* Readiness & Liveness health probes
* Horizontal Pod Autoscaling (HPA)
* Pod Disruption Budgets (PDB)
* Multi-zone/multi-node scheduling via Pod Anti-Affinity (optional)

---

## 📂 Folder Structure

```text
lab133_ha_cluster/
 ├── k8s/
 │   ├── deployment.yaml
 │   ├── service.yaml
 │   └── pdb.yaml
 └── README.md
```

---

## Step 0: Prerequisites

Before starting this lab, ensure you have:
* A running **Kubernetes cluster** (≥ 2 worker nodes, ideally across multiple availability zones).
* `kubectl` and `helm` installed and configured with cluster admin permissions.
* A containerized inference application image (e.g., FastAPI/Triton from Lab 126 or a standard inference service).

---

## Step 1: Create a Dedicated Namespace

Isolate all cluster resources within a dedicated namespace:

```bash
kubectl create namespace ai-ha
```

---

## Step 2: Create the Deployment Manifest (Multiple Replicas)

Define a Deployment configuration with **3 replicas**, explicit resource limits, and health probes (`/readyz` and `/healthz`).

### File: `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-inference
  namespace: ai-ha
  labels:
    app: ha-inference
spec:
  replicas: 3                # High availability base redundancy
  selector:
    matchLabels:
      app: ha-inference
  template:
    metadata:
      labels:
        app: ha-inference
    spec:
      # Step 6 (Optional): Multi-Zone / Anti-Affinity Scheduling
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - "ha-inference"
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: model-api
        image: YOUR_REGISTRY/secure-ml:latest   # Replace with your image repository
        ports:
        - containerPort: 8000
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 10
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
```

### Apply the Deployment:

```bash
kubectl -n ai-ha apply -f k8s/deployment.yaml
```

---

## Step 3: Configure the Load Balancer Service

Expose the inference deployment across pods using a Kubernetes Service of type `LoadBalancer`.

### File: `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ha-inference-svc
  namespace: ai-ha
spec:
  selector:
    app: ha-inference
  ports:
  - name: http
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

### Apply the Service:

```bash
kubectl -n ai-ha apply -f k8s/service.yaml
```

---

## Step 4: Configure Horizontal Pod Autoscaler (HPA)

Set up dynamic autoscaling to expand replica count automatically during traffic surges:

```bash
kubectl -n ai-ha autoscale deployment ha-inference \
  --cpu-percent=70 --min=3 --max=10
```

### Key Benefits:
* Automatically scales up to **10 pods** when average CPU utilization exceeds 70%.
* Guarantees a baseline minimum of **3 active pods** under low or zero load.

---

## Step 5: Define a Pod Disruption Budget (PDB)

Protect the inference service against voluntary disruptions (e.g., node drains, cluster upgrades) by enforcing minimum availability.

### File: `k8s/pdb.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ha-inference-pdb
  namespace: ai-ha
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: ha-inference
```

### Apply the PDB:

```bash
kubectl -n ai-ha apply -f k8s/pdb.yaml
```

> 🛡️ **Protection Guarantee:** Ensures that at least **2 inference pods** remain operational at all times during planned maintenance activities.

---

## Step 6: Multi-Zone & Anti-Affinity Scheduling (Overview)

By incorporating `podAntiAffinity` inside the Deployment spec (Step 2), Kubernetes distributes pods across separate physical host nodes or availability zones rather than placing them on a single node.

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - "ha-inference"
      topologyKey: "kubernetes.io/hostname"
```

---

## Step 7: Conduct a Chaos Test (Resilience Verification)

Simulate node/pod failures to verify cluster auto-healing capabilities:

1. **Delete an active pod manually:**
   ```bash
   POD_NAME=$(kubectl -n ai-ha get pods -l app=ha-inference -o jsonpath='{.items[0].metadata.name}')
   kubectl -n ai-ha delete pod $POD_NAME
   ```

2. **Watch pod recovery in real-time:**
   ```bash
   kubectl -n ai-ha get pods -w
   ```

### Observations:
* The **Deployment controller** instantly spawns a replacement pod to restore the target replica count.
* The **Readiness probe** keeps the new pod out of the Service endpoint pool until it is fully initialized, preventing 502/503 HTTP errors.

---

## Step 8: Validation and Load Testing

### 1. Obtain Service Endpoint
```bash
kubectl -n ai-ha get svc ha-inference-svc
```

### 2. Verify Continuous Load Balancing
Execute continuous health requests against the external IP / service endpoint:

```bash
while true; do
  curl -s http://<svc-ip>/healthz
  echo ""
  sleep 1
done
```

### 3. Load Testing HPA
Trigger HPA autoscaling by running a load generation tool (e.g., `hey`, `ab`, or `locust`):

```bash
# Example load testing with 'hey'
hey -z 2m -q 50 -c 20 http://<svc-ip>/readyz

# Monitor HPA status and pod scaling in another terminal:
kubectl -n ai-ha get hpa -w
```

---

## Step 9: Cleanup

Teardown all lab resources by deleting the namespace:

```bash
kubectl delete namespace ai-ha
```

---

## ✅ Summary Checklist

| Component | HA Strategy Implemented |
| :--- | :--- |
| **Redundancy** | Minimum 3 active pod replicas running across worker nodes. |
| **Traffic Routing** | LoadBalancer Service distributing incoming inference requests evenly. |
| **Auto-Healing** | Liveness probes restarting stuck pods; Readiness probes filtering unready pods. |
| **Auto-Scaling** | Horizontal Pod Autoscaler dynamically scaling between 3 and 10 replicas based on CPU usage. |
| **Maintenance Safety** | Pod Disruption Budget guaranteeing minimum 2 operational pods during drains. |
| **Zone Fault Isolation** | Pod anti-affinity rules preventing single-point-of-failure node co-location. |
