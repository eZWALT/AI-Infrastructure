# 🧪 280. Lab – Configure Multi-Tenant Cluster

## 🎯 Objective
- Configure **multi-tenant AI infra** in Kubernetes.
- Isolate teams via **namespaces**.
- Apply **RBAC**, **quotas**, and **GPU limits**.
- Enable **per-tenant monitoring** & **cost tracking**.

---

## Step 1: Create Namespaces per Tenant

Namespaces separate resources for each team.

```bash
kubectl create namespace team-a
kubectl create namespace team-b
```

> **✅ Expected:** Running `kubectl get ns` shows `team-a` and `team-b`.

---

## Step 2: Define Resource Quotas

Restrict CPU, memory, and GPU usage per team.

```yaml
# team-a-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "64Gi"
    requests.nvidia.com/gpu: "2"
    limits.cpu: "40"
    limits.memory: "128Gi"
    limits.nvidia.com/gpu: "4"
```

```bash
kubectl apply -f team-a-quota.yaml
```

> **✅ Expected:** If Team A tries to request >4 GPUs, the job fails.

---

## Step 3: Create RBAC Roles

Define what actions each team can perform.

```yaml
# role-ml-trainer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: ml-trainer
rules:
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["create", "get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
# rolebinding-ml-trainer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ml-trainer-binding
  namespace: team-a
subjects:
- kind: User
  name: alice@company.com
roleRef:
  kind: Role
  name: ml-trainer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f role-ml-trainer.yaml
kubectl apply -f rolebinding-ml-trainer.yaml
```

> **✅ Expected:** Alice can launch jobs **only inside** the `team-a` namespace.

---

## Step 4: Enable GPU Isolation

Install NVIDIA GPU operator and enforce per-tenant GPU allocation.

```bash
kubectl label node <gpu-node-name> team=team-a
```

Use **node selectors** in job specs:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        team: team-a
      containers:
      - name: trainer
        image: pytorch/pytorch:latest
        resources:
          limits:
            nvidia.com/gpu: 2
```

> **✅ Expected:** Team A’s pods only run on assigned GPU nodes.

---

## Step 5: Set Up Monitoring

Deploy **Prometheus + Grafana** for cluster metrics:

```bash
kubectl create ns monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack -n monitoring
```

- Grafana dashboards show **per-namespace usage**.
- Add filters by **namespace** to separate tenants.

---

## Step 6: Enable Cost Allocation with Kubecost

```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer -n kubecost
```

- Kubecost tracks **GPU/CPU/memory costs** per namespace.
- Teams can view usage in dashboards.

> **✅ Expected:** Team A sees their GPU spend separate from Team B.

---

## Step 7: Test Multi-Tenant Setup

- **Team A job submission (allowed):**

  ```bash
  kubectl run job1 -n team-a --image=pytorch/pytorch --limits="nvidia.com/gpu=1"
  ```

- **Team A tries Team B namespace (denied):**

  ```bash
  kubectl run job2 -n team-b --image=pytorch/pytorch --limits="nvidia.com/gpu=1"
  ```

> **✅ Expected:** Access denied due to RBAC restrictions.

---

## Step 8 (Optional Extensions)

- Add **network policies** to prevent cross-tenant traffic.
- Use **per-tenant storage buckets** (S3/GCS).
- Configure **alerts** (Prometheus) when tenant exceeds quota.
- Integrate with **OIDC/SSO** for centralized identity.

---

## 📝 Wrap-Up

- Configured a **multi-tenant AI cluster** with namespaces, quotas, RBAC, and GPU isolation.
- Added **Prometheus/Grafana + Kubecost** for per-tenant monitoring & billing.
- Ensured **fairness, security, and accountability** in shared infra.
