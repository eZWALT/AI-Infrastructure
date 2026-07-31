84. Lab – Optimize Cloud AI Workload Costs
Lab 84 — Optimize Cloud AI Workload Costs
Goal: Cut end-to-end training/inference costs by 40–80% while maintaining (or improving) throughput and accuracy.
You’ll do: baseline → instrument → optimize (compute, storage, network, scheduling) → measure → report.

Download the Cost Tracking Worksheet (Excel)

0) Prerequisites
A cloud account (AWS or GCP or Azure).

One GPU instance (on-demand) and permission to launch spot/preemptible.

Docker + Kubernetes cluster (managed or self-hosted) or VM-only path.

CLI: aws or gcloud or az; kubectl (if using K8s).

Sample project: PyTorch ResNet-50 on CIFAR-10 (training) + FastAPI/Triton (inference).

Tip: Keep your dataset/model fixed across runs so cost deltas are attributable to infra changes.

1) Design the Experiment
Define the baseline (row 2 in the Excel):

Cloud/Region: (e.g., AWS us-east-1)

Instance: 1× GPU (A10/T4/V100 class) on-demand

Storage: local NVMe + object store (S3/GCS/Blob) in same region

No autoscaling, no spot, FP32, batch size 128

KPIs to record (per run):

Throughput/s, Latency (p50/p95), GPU util %, Time to train, Accuracy, Cost (compute/storage/network), TotalCostUSD, Savings vs baseline %.

Use the worksheet to log each run: one change per row.

2) Instrumentation & Visibility (Baseline + All Runs)
A. VM-level quick check
# GPU + memory + power
nvidia-smi --query-gpu=name,utilization.gpu,utilization.memory,memory.total,pstate,power.draw --format=csv -l 5
# Disk & network
iostat -xz 5
ifstat 5
B. Kubernetes (recommended)
Prometheus + Grafana for metrics, NVIDIA DCGM Exporter for GPU.

# Add DCGM exporter (Helm)
helm repo add nvidia https://nvidia.github.io/dcgm-exporter
helm install dcgm nvidia/dcgm-exporter -n monitoring --create-namespace
Optional: Kubecost for $ visibility.

helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer -n kubecost --create-namespace
C. Tag everything for cost attribution
AWS: --tag-specifications 'ResourceType=instance,Tags=[{Key=Project,Value=Lab84},{Key=Owner,Value=Vivian}]'

GCP: --labels=project=lab84,owner=vivian

Azure: --tags Project=Lab84 Owner=Vivian

3) Baseline Run (On-Demand)
A. Launch (pick your cloud)
AWS (GPU on-demand example)

aws ec2 run-instances \
  --image-id ami-... \
  --instance-type g5.2xlarge \
  --key-name yourkey \
  --count 1 \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":200}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Project,Value=Lab84}]'
GCP (on-demand)

gcloud compute instances create lab84-ondemand \
  --zone=us-central1-a --machine-type=a2-highgpu-1g \
  --accelerator=count=1,type=nvidia-tesla-a100 \
  --boot-disk-size=200GB --labels=project=lab84
Azure (on-demand)

az vm create -g rg-lab84 -n lab84-ondemand \
  --image Ubuntu2204 --size Standard_NC4as_T4_v3 \
  --storage-sku Premium_LRS --tags Project=Lab84
B. Train (FP32, no optimizations yet)
Use your standard ResNet-50 script, e.g., PyTorch CIFAR-10.

Record: Throughput/s, GPU util %, Training time, Accuracy, Costs.

Fill row 2 (baseline) in the Excel.

4) Optimization Levers (Run one lever per row)
4.1 Spot/Preemptible Instances (+ Checkpointing)
Enable resilient training first:

PyTorch (automatic mixed precision optional later):

# train.py (snippet)
from torch.cuda.amp import autocast, GradScaler
scaler = GradScaler()
start_step = load_checkpoint_if_exists(model, optimizer, scaler)  # your function
 
for step, (x, y) in enumerate(loader, start=start_step):
    optimizer.zero_grad(set_to_none=True)
    with autocast(enabled=False):  # keep False for pure baseline; set True in AMP step
        yhat = model(x.cuda(non_blocking=True))
        loss = criterion(yhat, y.cuda(non_blocking=True))
    loss.backward()
    optimizer.step()
    if step % CKPT_EVERY == 0:
        save_checkpoint(model, optimizer, scaler, step)  # lightweight, atomic
Launch Spot/Preemptible:

AWS Spot:

aws ec2 run-instances \
  --instance-market-options 'MarketType=spot' \
  --instance-type g5.2xlarge --image-id ami-... \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Project,Value=Lab84}]'
GCP Spot / Preemptible:

gcloud compute instances create lab84-spot \
  --zone=us-central1-a --machine-type=a2-highgpu-1g \
  --provisioning-model=SPOT --boot-disk-size=200GB \
  --labels=project=lab84
Azure Spot:

az vm create -g rg-lab84 -n lab84-spot \
  --image Ubuntu2204 --size Standard_NC4as_T4_v3 \
  --priority Spot --max-price -1 --eviction-policy Deallocate \
  --tags Project=Lab84
Measure & log a new row (expect large compute $ drop).

4.2 Autoscaling (K8s)
Cluster Autoscaler enabled on your managed cluster.

HPA for inference pods:

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: inference-hpa, namespace: ai }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-api
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 70 }
Log costs before/after a load test.

4.3 Mixed Precision (AMP) for Training
Turn autocast(enabled=True) + GradScaler:

with autocast(True):
    yhat = model(x.cuda(non_blocking=True))
    loss = criterion(yhat, y.cuda(non_blocking=True))
scaler.scale(loss).backward()
scaler.step(optimizer)
scaler.update()
Usually boosts throughput and reduces time → lower compute cost.

4.4 Batch Size & Gradient Accumulation
Increase BatchSize and add GradAccumSteps to keep memory in check.
Record throughput & accuracy; keep the best trade-off.

4.5 Right-Sizing & Utilization
If GPU util < 50%, you’re probably I/O-bound.

Cache dataset on local NVMe.

Pin dataloader workers, enable async I/O.

loader = DataLoader(ds, batch_size=B, shuffle=True, num_workers=8, pin_memory=True, persistent_workers=True)
4.6 Storage Lifecycle Policies
Move old checkpoints/logs to cold storage automatically.

AWS S3 (lifecycle JSON example):

{
  "Rules": [{
    "ID": "MoveOlderArtifacts",
    "Filter": { "Prefix": "experiments/" },
    "Status": "Enabled",
    "Transitions": [{ "Days": 14, "StorageClass": "STANDARD_IA" },
                    { "Days": 45, "StorageClass": "GLACIER" }],
    "NoncurrentVersionTransitions": [{ "NoncurrentDays": 30, "StorageClass": "GLACIER" }]
  }]
}
GCP and Azure offer similar policies—configure equivalent rules.
Log storage and egress cost changes in the sheet.

4.7 Data Locality (Avoid Cross-Region Egress)
Keep compute and object storage in the same region.
If you were cross-region, redo the run same-region and record the network $ drop.

4.8 Container & Runtime Optimizations
Base images: use slim CUDA runtimes, remove build tools from runtime image.

Enable torch.compile() (PyTorch 2) or TensorRT for inference if supported.

# training (safe when model is stable)
model = torch.compile(model)  # often improves throughput
4.9 Inference Cost Cuts (Triton)
Enable dynamic batching and model-level optimization.

config.pbtxt (snippet):

max_batch_size: 64
dynamic_batching { preferred_batch_size: [ 4, 8, 16, 32 ] max_queue_delay_microseconds: 1000 }
instance_group [{ kind: KIND_GPU, count: 1 }]
Add a Redis cache layer for idempotent requests (e.g., embeddings).

4.10 Reservations/Commitments (Analysis Step)
Not a live change for the lab, but simulate:

If a fixed baseline footprint runs 24/7 (e.g., inference), estimate Reserved Instances / Savings Plans (AWS) or Committed Use Discounts (GCP) and log the hypothetical monthly savings in the sheet’s notes for planning.

5) Kubernetes: Spot-Aware Scheduling (Optional but Powerful)
Create node pools: on-demand and spot.

Taint spot nodes: spot=true:NoSchedule.

Tolerate in your training job and add PodDisruptionBudget.

apiVersion: apps/v1
kind: Deployment
metadata: { name: trainer, namespace: ai }
spec:
  replicas: 1
  template:
    metadata: { labels: { app: trainer } }
    spec:
      tolerations:
      - key: "spot" operator: "Equal" value: "true" effect: "NoSchedule"
      nodeSelector: { lifecycle: spot }
      containers:
      - name: trainer
        image: yourrepo/trainer:latest
        resources:
          limits: { nvidia.com/gpu: "1" }
This keeps stateful services on on-demand and ephemeral training on spot.

6) Run, Measure, Record (Repeat)
For each lever:

Apply exactly one change.

Run training/inference.

Capture metrics & costs.

Fill a new row in the Excel (it auto-sums TotalCost; Savings% compares to baseline).

Keep short notes on what changed.

Target sequence: Spot → AMP → Batch/GA → Data Locality → Lifecycle → Autoscale → Runtime/Compile → Inference batching/cache.

7) Analyze & Decide
Plot TotalCostUSD vs Throughput/s to spot Pareto-optimal settings.

Confirm that accuracy remains within your tolerance window.

Promote the top configuration to your default pipeline.

What “Good” Looks Like (Typical Wins)
Spot/Preemptible + checkpointing: 60–80% compute cost cut

AMP + larger batches: 20–40% time cut at same accuracy

Same-region storage + lifecycle: double-digit % storage/network savings

Triton dynamic batching + cache: big inference $ drop with stable latency




