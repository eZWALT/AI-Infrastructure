# Lab 112: Monitor GPU Cluster with Prometheus

Here is a practical, end-to-end lab to monitor a Kubernetes GPU cluster with Prometheus, Grafana dashboards, and basic alerting rules. You will deploy NVIDIA's **DCGM Exporter** to expose GPU metrics, scrape them with Prometheus, visualize them in Grafana, and add alert rules.

---

## 0. Mental Model (Architecture)

```text
[ GPU Nodes ] ────────► [ DCGM Exporter ] (DaemonSet)
                             │
                             │ Port 9400
                             ▼
[ Prometheus ] ◄────── [ ServiceMonitor ]
      │
      ├────────────────► [ Grafana Dashboards ]
      │
      └────────────────► [ Alertmanager ] (Alert Rules)
```

* **GPU Nodes** run **DCGM Exporter** as a `DaemonSet`, exposing GPU metrics on port `9400`.
* **Prometheus** scrapes DCGM Exporter metrics alongside standard cluster metrics (`node-exporter`, `kube-state-metrics`).
* **Grafana** visualizes GPU utilization, memory, temperature, and power usage.
* **Alertmanager** routes alert notifications based on predefined rules.

---

## 1. Prerequisites

* A Kubernetes cluster with at least one NVIDIA GPU node (NVIDIA drivers installed).
* `kubectl` and `helm` installed and configured for the cluster.
* NVIDIA Device Plugin installed on GPU nodes.
* Cluster admin privileges.

> 💡 **Tip:** If your cluster is not GPU-ready yet, install the NVIDIA device plugin first:
> ```bash
> helm repo add nvidia https://nvidia.github.io/k8s-device-plugin
> helm repo update
> kubectl create ns gpu-operator --dry-run=client -o yaml | kubectl apply -f -
> helm install nvidia-device-plugin nvidia/k8s-device-plugin -n gpu-operator
> ```

---

## 2. Create a Dedicated Namespace

Keep all monitoring components localized in a single namespace:

```bash
kubectl create namespace monitoring
```

---

## 3. Install Prometheus Stack (`kube-prometheus-stack`)

This Helm chart deploys Prometheus, Grafana, Alertmanager, `node-exporter`, and `kube-state-metrics`.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```

> **Why this stack?** It provides a production-ready Prometheus deployment using CRDs (`ServiceMonitor` and `PrometheusRule`), allowing declarative target scraping and alert definitions.

---

## 4. Label GPU Nodes

Schedule the DCGM exporter exclusively on nodes equipped with GPUs:

```bash
# Replace <node-name> with each GPU node
kubectl label nodes <node-name> gpu=true
```

> **Note:** Labeling ensures the exporter container runs only where physical GPUs exist. If using Node Feature Discovery (NFD), update the node selector in the DaemonSet YAML accordingly.

---

## 5. Deploy NVIDIA DCGM Exporter

The DCGM Exporter collects hardware metrics (GPU utilization, frame-buffer memory, temperature, power usage, ECC errors, clocks) per node/GPU.

### Create `dcgm-exporter.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dcgm-exporter
  namespace: monitoring
  labels:
    app: dcgm-exporter
spec:
  selector:
    matchLabels:
      app: dcgm-exporter
  template:
    metadata:
      labels:
        app: dcgm-exporter
    spec:
      nodeSelector:
        gpu: "true"
      hostPID: false
      hostNetwork: false
      tolerations:
        - effect: NoSchedule
          operator: Exists
        - effect: NoExecute
          operator: Exists
      containers:
        - name: dcgm-exporter
          image: nvcr.io/nvidia/k8s/dcgm-exporter:latest
          imagePullPolicy: IfNotPresent
          ports:
            - name: metrics
              containerPort: 9400
          securityContext:
            privileged: true
          env:
            - name: DCGM_EXPORTER_KUBERNETES
              value: "true"
          volumeMounts:
            - name: pod-resources
              mountPath: /var/lib/kubelet/pod-resources
              readOnly: true
      volumes:
        - name: pod-resources
          hostPath:
            path: /var/lib/kubelet/pod-resources
            type: Directory
---
apiVersion: v1
kind: Service
metadata:
  name: dcgm-exporter
  namespace: monitoring
  labels:
    app: dcgm-exporter
spec:
  clusterIP: None  # Headless service ensures Prometheus scrapes each pod endpoint directly
  selector:
    app: dcgm-exporter
  ports:
    - name: metrics
      port: 9400
      targetPort: metrics
```

### Apply Configuration
```bash
kubectl apply -f dcgm-exporter.yaml
kubectl -n monitoring get pods -l app=dcgm-exporter -o wide
```

---

## 6. Configure Prometheus Scrape Target (`ServiceMonitor`)

Create a custom resource that instructs Prometheus to discover and scrape the DCGM exporter service.

### Create `dcgm-servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dcgm-exporter
  namespace: monitoring
  labels:
    release: monitoring  # Must match the Helm release name of kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: dcgm-exporter
  namespaceSelector:
    matchNames: ["monitoring"]
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

### Apply Configuration
```bash
kubectl apply -f dcgm-servicemonitor.yaml
```

---

## 7. Sanity Check: Inspect Raw Metrics

Verify metric collection by port-forwarding directly to a DCGM Exporter pod:

```bash
POD=$(kubectl -n monitoring get pod -l app=dcgm-exporter -o jsonpath='{.items[0].metadata.name}')
kubectl -n monitoring port-forward pod/$POD 9400:9400
```

In a separate terminal, fetch the raw metrics endpoint:

```bash
curl -s localhost:9400/metrics | head -n 40
```

*Key metrics exposed:*
* `DCGM_FI_DEV_GPU_UTIL` (GPU compute utilization)
* `DCGM_FI_DEV_GPU_TEMP` (GPU temperature)
* `DCGM_FI_DEV_FB_USED` / `DCGM_FI_DEV_FB_TOTAL` (Frame buffer / VRAM usage)
* `DCGM_FI_DEV_POWER_USAGE` (Power draw in Watts)

---

## 8. Query Metrics in Prometheus UI

1. Locate the Prometheus service:
   ```bash
   kubectl -n monitoring get svc | grep prometheus
   ```

2. Port-forward the Prometheus dashboard endpoint:
   ```bash
   kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090
   ```

3. Open `http://localhost:9090` in a browser and test the following PromQL queries:

| Measurement | PromQL Query |
| :--- | :--- |
| **GPU Utilization (%)** | `avg by (instance, gpu) (DCGM_FI_DEV_GPU_UTIL)` |
| **Memory Utilization (%)** | `100 * (DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_TOTAL)` |
| **GPU Temperature (°C)** | `max by (instance, gpu) (DCGM_FI_DEV_GPU_TEMP)` |
| **Power Draw (Watts)** | `avg by (instance, gpu) (DCGM_FI_DEV_POWER_USAGE)` |

---

## 9. Visualize in Grafana

1. **Retrieve Admin Password:**
   ```bash
   kubectl -n monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
   ```

2. **Port-forward Grafana Service:**
   ```bash
   kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
   ```

3. **Access Dashboard:**
   * URL: `http://localhost:3000`
   * Username: `admin`
   * Password: *(Decoded string from step 1)*

4. **Build Dashboards:**
   * Create a new dashboard and add panels using the PromQL expressions listed in Step 8.
   * Add dashboard variables for `instance` and `gpu` to enable per-node and per-GPU filtering.

---

## 10. Configure GPU Alert Rules (`PrometheusRule`)

Create automated alert rules for scrape failures, thermal limits, memory pressure, and hardware memory errors.

### Create `gpu-alerts.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: gpu.rules
      rules:
        - alert: GPUScrapeMissing
          expr: up{job="dcgm-exporter"} == 0
          for: 10m
          labels: 
            severity: warning
          annotations:
            summary: "DCGM exporter scrape failing"
            description: "Prometheus cannot scrape DCGM exporter targets for 10m."

        - alert: GPUHighTemperature
          expr: max by (instance, gpu) (DCGM_FI_DEV_GPU_TEMP) > 80
          for: 5m
          labels: 
            severity: warning
          annotations:
            summary: "GPU temperature high (>80°C)"
            description: "Instance {{ $labels.instance }} GPU {{ $labels.gpu }} too hot."

        - alert: GPUMemoryPressure
          expr: 100 * (DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_TOTAL) > 90
          for: 10m
          labels: 
            severity: warning
          annotations:
            summary: "GPU memory > 90% for 10m"
            description: "Sustained memory pressure on {{ $labels.instance }} GPU {{ $labels.gpu }}."

        - alert: GPUECCErrorsSpike
          expr: rate(DCGM_FI_DEV_ECC_SBE_VOL_TOTAL[5m]) > 0
          for: 5m
          labels: 
            severity: critical
          annotations:
            summary: "ECC single-bit errors increasing"
            description: "ECC SBE rate > 0 on {{ $labels.instance }} GPU {{ $labels.gpu }}."
```

### Apply Configuration
```bash
kubectl apply -f gpu-alerts.yaml
```

---

## 11. (Optional) Pod & Namespace Level Attribution

Because `/var/lib/kubelet/pod-resources` is mounted into the DCGM Exporter DaemonSet, metric samples can automatically inherit Kubernetes context labels (`pod`, `namespace`, `container`). This enables multi-tenant GPU usage tracking and per-pod resource attribution inside Grafana.

---

## 12. Troubleshooting Checklist

| Issue | Potential Cause | Resolution |
| :--- | :--- | :--- |
| **No metrics in Prometheus** | Label mismatch on `ServiceMonitor` | Ensure `metadata.labels.release` matches the Helm release name (`monitoring`). Verify service port name is `metrics`. |
| **Exporter `CrashLoopBackOff`** | Missing drivers or privileges | Run `nvidia-smi` on the host to verify drivers. Ensure `privileged: true` is set in the container security context. |
| **Missing MIG Metrics** | Unconfigured Multi-Instance GPU | Verify MIG mode status and driver support. Ensure hostPath `/var/lib/kubelet/pod-resources` is mounted correctly. |
| **Grafana Panels Empty** | Wrong Datasource or Metric Name | Select default Prometheus data source. Verify raw metric names on `/metrics` endpoint as some builds prefix metrics with `nvidia_dcgm_`. |

---

## 13. Teardown & Clean Up

To remove all created resources and return the cluster to its original state:

```bash
kubectl -n monitoring delete -f gpu-alerts.yaml
kubectl -n monitoring delete -f dcgm-servicemonitor.yaml
kubectl -n monitoring delete -f dcgm-exporter.yaml
helm -n monitoring uninstall monitoring
kubectl delete ns monitoring
```

---

## ✅ Expected Final Verification

* **Prometheus Targets:** All DCGM Exporter instances report status `UP`.
* **Grafana Dashboards:** Visualizing real-time GPU Utilization %, Memory %, Temperature, and Power metrics.
* **Alerting Engine:** Alert rules active and triggering when thresholds (>80°C temperature, >90% memory) are exceeded.
