# 🧪 Lab – Deploy MLflow on Kubernetes

## Goal

Stand up an **MLflow Tracking Server** on Kubernetes with:

- **PostgreSQL** as the backend store (metrics, params, runs)
- **MinIO** (S3-compatible) as the artifact store
- Persistent Volumes, Service, and Ingress for external access

**Estimated time:** 90–120 minutes

**Requirements:**

- Kubernetes cluster (Kind, Minikube, or cloud)
- `kubectl`
- `helm`
- Docker (only if building custom images)

**Namespace:** `mlops`

---

# 1. Create Namespace & Verify StorageClass

```bash
kubectl create namespace mlops

# Verify default StorageClass
kubectl get storageclass
```

> If your cluster has no default StorageClass, create or configure one before continuing.

---

# 2. Install MinIO (Artifact Store)

## Add Helm Repository

```bash
helm repo add minio https://charts.min.io/
helm repo update
```

## Install MinIO

```bash
helm install minio minio/minio \
  --namespace mlops \
  --set rootUser=admin \
  --set rootPassword=admin12345 \
  --set resources.requests.memory=256Mi \
  --set mode=standalone \
  --set replicas=1
```

## Access the Console

```bash
kubectl -n mlops port-forward svc/minio 9000:9000 9001:9001
```

Console:

```
http://localhost:9001
```

Credentials:

- Username: `admin`
- Password: `admin12345`

Create a bucket named:

```
mlflow-artifacts
```

---

# 3. Install PostgreSQL (Backend Store)

## Add Bitnami Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## Install PostgreSQL

```bash
helm install pg bitnami/postgresql \
  --namespace mlops \
  --set global.postgresql.auth.postgresPassword=pgpass \
  --set global.postgresql.auth.username=mlflow \
  --set global.postgresql.auth.password=mlflowpass \
  --set global.postgresql.auth.database=mlflowdb \
  --set primary.persistence.size=5Gi
```

## Verify Service

```bash
PG_HOST=$(kubectl -n mlops get svc pg-postgresql -o jsonpath='{.spec.clusterIP}')
echo $PG_HOST
```

---

# 4. Create Secrets for MLflow

```bash
kubectl -n mlops create secret generic mlflow-secrets \
  --from-literal=BACKEND_URI="postgresql://mlflow:mlflowpass@pg-postgresql.mlops.svc.cluster.local:5432/mlflowdb" \
  --from-literal=ARTIFACT_URI="s3://mlflow-artifacts" \
  --from-literal=AWS_ACCESS_KEY_ID="admin" \
  --from-literal=AWS_SECRET_ACCESS_KEY="admin12345" \
  --from-literal=MLFLOW_TRACKING_USERNAME="admin" \
  --from-literal=MLFLOW_TRACKING_PASSWORD="changeme"
```

---

# 5. (Optional) Create the MinIO Bucket Automatically

Create `mkbucket.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mkbucket
  namespace: mlops

spec:
  template:
    spec:
      restartPolicy: Never

      containers:
      - name: mc
        image: minio/mc:latest

        env:
        - name: MC_HOST_minio
          value: http://admin:admin12345@minio.mlops.svc.cluster.local:9000

        command:
        - sh
        - -c

        args:
        - |
          mc ls minio || true
          mc mb -p minio/mlflow-artifacts || true
```

Apply it:

```bash
kubectl apply -f mkbucket.yaml

kubectl -n mlops logs job/mkbucket
```

---

# 6. Create Persistent Volume Claim

Create `mlflow-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: mlflow-pvc
  namespace: mlops

spec:
  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 5Gi
```

Apply it:

```bash
kubectl apply -f mlflow-pvc.yaml
```

---

# 7. Deploy MLflow Tracking Server

Create `mlflow-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: mlflow
  namespace: mlops

spec:
  replicas: 1

  selector:
    matchLabels:
      app: mlflow

  template:
    metadata:
      labels:
        app: mlflow

    spec:
      containers:
      - name: mlflow
        image: ghcr.io/mlflow/mlflow:v2.14.1
        imagePullPolicy: IfNotPresent

        ports:
        - containerPort: 5000

        envFrom:
        - secretRef:
            name: mlflow-secrets

        env:
        - name: MLFLOW_S3_ENDPOINT_URL
          value: http://minio.mlops.svc.cluster.local:9000

        - name: AWS_DEFAULT_REGION
          value: us-east-1

        - name: MLFLOW_TRACKING_USERNAME
          valueFrom:
            secretKeyRef:
              name: mlflow-secrets
              key: MLFLOW_TRACKING_USERNAME

        - name: MLFLOW_TRACKING_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mlflow-secrets
              key: MLFLOW_TRACKING_PASSWORD

        command:
        - mlflow

        args:
        - server
        - --host=0.0.0.0
        - --port=5000
        - --backend-store-uri=$(BACKEND_URI)
        - --default-artifact-root=$(ARTIFACT_URI)

        volumeMounts:
        - name: mlflow-data
          mountPath: /mlflow

      volumes:
      - name: mlflow-data
        persistentVolumeClaim:
          claimName: mlflow-pvc

---

apiVersion: v1
kind: Service

metadata:
  name: mlflow
  namespace: mlops

spec:
  type: ClusterIP

  selector:
    app: mlflow

  ports:
  - name: http
    port: 5000
    targetPort: 5000
```

Deploy:

```bash
kubectl apply -f mlflow-deploy.yaml

kubectl -n mlops get pods,svc
```

---

# 8. Expose MLflow

## Option A — Port Forward

```bash
kubectl -n mlops port-forward svc/mlflow 5000:5000
```

Open:

```
http://localhost:5000
```

---

## Option B — Ingress

Create `mlflow-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: mlflow
  namespace: mlops

  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: mlflow-basic-auth

spec:
  rules:
  - host: mlflow.localtest.me

    http:
      paths:
      - path: /
        pathType: Prefix

        backend:
          service:
            name: mlflow

            port:
              number: 5000
```

Create the authentication secret:

```bash
htpasswd -bc htpasswd admin changeme

kubectl -n mlops create secret generic mlflow-basic-auth \
  --from-file=auth=htpasswd
```

Apply:

```bash
kubectl apply -f mlflow-ingress.yaml
```

Update `/etc/hosts` if necessary to point:

```
mlflow.localtest.me
```

to your Ingress IP.

---

# 9. Verify End-to-End

Launch the UI.

Run the following client:

```python
import os
import mlflow

os.environ["MLFLOW_TRACKING_USERNAME"] = "admin"
os.environ["MLFLOW_TRACKING_PASSWORD"] = "changeme"

mlflow.set_tracking_uri("http://localhost:5000")

with mlflow.start_run():
    mlflow.log_param("lr", 0.001)
    mlflow.log_metric("loss", 0.42, step=1)

    with open("hello.txt", "w") as f:
        f.write("artifact test")

    mlflow.log_artifact("hello.txt")
```

Verify:

- ✅ Run appears in MLflow UI
- ✅ Parameter stored
- ✅ Metric stored
- ✅ `hello.txt` uploaded to the `mlflow-artifacts` bucket

---

# 10. (Optional) Hardening

- Restrict MinIO with NetworkPolicy
- Rotate database credentials
- Store secrets in an external secrets manager
- Enable TLS (Ingress + MinIO)
- Configure `cert-manager`

---

# Troubleshooting

## Artifacts not saving

Check:

- `MLFLOW_S3_ENDPOINT_URL`
- Bucket exists
- Access keys are correct

---

## Database connection errors

Verify:

- `BACKEND_URI`
- PostgreSQL Service
- Kubernetes DNS resolution

---

## UI not loading

Check logs:

```bash
kubectl -n mlops logs deploy/mlflow
```

Verify:

- Service
- Port-forward
- Ingress

---

## Permission denied

Verify:

- Bucket policies
- MinIO credentials
- Access keys

---

# Cleanup

```bash
helm -n mlops uninstall minio

helm -n mlops uninstall pg

kubectl delete namespace mlops
```

---

# 🎯 Learning Outcomes

By completing this lab you will be able to:

- Deploy MLflow Tracking Server on Kubernetes
- Configure PostgreSQL as the backend store
- Configure MinIO as the S3 artifact store
- Use Persistent Volumes for storage
- Expose MLflow via Service and Ingress
- Secure access with Basic Authentication
- Validate end-to-end experiment tracking
