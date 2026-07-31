# Lab 19–21 — GPU Virtual Machines Across AWS, Google Cloud & Azure

**Goal:** Deploy equivalent GPU virtual machines on AWS, Google Cloud, and Azure, verify GPU availability, run the same PyTorch benchmark on each platform, and compare their performance and cost.

**Estimated Time:** 2–3 hours

**Cost:** A few dollars per cloud provider (terminate all resources after completing the lab).

---

# Learning Objectives

By the end of this lab you will be able to:

- Launch GPU-enabled virtual machines on the three major cloud providers
- Connect securely using SSH
- Verify CUDA installation
- Run GPU workloads with PyTorch
- Benchmark GPU performance
- Compare cost vs performance across cloud providers

---

# Benchmark Script

Create a file named `benchmark.py`.

```python
import time
import torch

device = "cuda" if torch.cuda.is_available() else "cpu"

x = torch.rand((10000, 10000), device=device)

# Warmup
_ = x @ x

if device == "cuda":
    torch.cuda.synchronize()

start = time.time()

for _ in range(10):
    y = x @ x

if device == "cuda":
    torch.cuda.synchronize()

end = time.time()

print(f"Device: {torch.cuda.get_device_name(0) if device=='cuda' else 'CPU'}")
print(f"PyTorch: {torch.__version__}")
print(f"Execution time: {end-start:.3f} seconds")
```

Run this exact script on every cloud.

---

# Part A — Amazon Web Services (AWS)

## Recommended Configuration

| Setting | Value |
|---------|-------|
| Instance | `g5.xlarge` |
| GPU | NVIDIA A10G |
| Image | Deep Learning AMI |
| Storage | 100 GB |

### Create an SSH Key

```
EC2 → Key Pairs
```

```bash
chmod 400 ~/.ssh/ai-keypair.pem
```

### Launch the Instance

```
EC2 → Launch Instance
```

Allow SSH from your IP.

### Connect

```bash
ssh -i ~/.ssh/ai-keypair.pem ubuntu@<PUBLIC_IP>
```

### Verify CUDA

```bash
nvidia-smi
```

### Activate PyTorch

```bash
conda activate pytorch
```

### Run Benchmark

```bash
python benchmark.py
```

---

# Part B — Google Cloud Platform (GCP)

## Recommended Configuration

| Setting | Value |
|---------|-------|
| Machine | `g2-standard-8` |
| GPU | NVIDIA L4 |
| Image | Ubuntu 22.04 |
| Storage | 100 GB |

### Enable Compute Engine

```
APIs & Services → Compute Engine API
```

### Create the VM

```
Compute Engine → VM Instances
```

### Connect

```bash
gcloud compute ssh ai-gcp-lab --zone=<ZONE>
```

### Install Drivers

```bash
sudo ubuntu-drivers autoinstall

sudo reboot
```

### Verify GPU

```bash
nvidia-smi
```

### Install PyTorch

```bash
pip install torch torchvision torchaudio \
--index-url https://download.pytorch.org/whl/cu121
```

### Run Benchmark

```bash
python benchmark.py
```

---

# Part C — Microsoft Azure

## Recommended Configuration

| Setting | Value |
|---------|-------|
| VM | Standard_NC4as_T4_v3 |
| GPU | NVIDIA T4 |
| Image | Azure ML DLVM (Ubuntu) |
| Storage | 100 GB |

### Create a Virtual Machine

```
Azure Portal → Virtual Machines
```

Select the **Azure Machine Learning Deep Learning VM** image.

### Connect

```bash
ssh azureuser@<PUBLIC_IP>
```

### Verify GPU

```bash
nvidia-smi
```

The DLVM already includes CUDA and PyTorch.

### Run Benchmark

```bash
python benchmark.py
```

---

# Collect Results

Record the following information for every provider.

| Cloud | GPU | PyTorch Version | Runtime (s) | On-demand Cost/hr |
|--------|-----|-----------------|------------:|------------------:|
| AWS | | | | |
| Google Cloud | | | | |
| Azure | | | | |

Example:

| Cloud | GPU | Runtime | Cost/hr |
|--------|------|---------:|---------:|
| AWS | A10G | 12.4 s | \$1.20 |
| GCP | L4 | 10.8 s | \$0.95 |
| Azure | T4 | 17.6 s | \$0.90 |

---

# Discussion

Answer the following questions:

1. Which cloud delivered the fastest execution time?

2. Which provider offered the best performance per dollar?

3. How do the GPU architectures (T4, L4, A10G) differ?

4. Would Spot/Preemptible instances change your decision?

5. Which cloud would you choose for:
   - Model training# Lab 19–21 — GPU Virtual Machines Across AWS, Google Cloud & Azure

**Goal:** Deploy equivalent GPU virtual machines on AWS, Google Cloud, and Azure, verify GPU availability, run the same PyTorch benchmark on each platform, and compare their performance and cost.

**Estimated Time:** 2–3 hours

**Cost:** A few dollars per cloud provider (terminate all resources after completing the lab).

---

# Learning Objectives

By the end of this lab you will be able to:

- Launch GPU-enabled virtual machines on the three major cloud providers
- Connect securely using SSH
- Verify CUDA installation
- Run GPU workloads with PyTorch
- Benchmark GPU performance
- Compare cost vs performance across cloud providers

---

# Benchmark Script

Create a file named `benchmark.py`.

```python
import time
import torch

device = "cuda" if torch.cuda.is_available() else "cpu"

x = torch.rand((10000, 10000), device=device)

# Warmup
_ = x @ x

if device == "cuda":
    torch.cuda.synchronize()

start = time.time()

for _ in range(10):
    y = x @ x

if device == "cuda":
    torch.cuda.synchronize()

end = time.time()

print(f"Device: {torch.cuda.get_device_name(0) if device=='cuda' else 'CPU'}")
print(f"PyTorch: {torch.__version__}")
print(f"Execution time: {end-start:.3f} seconds")
```

Run this exact script on every cloud.

---

# Part A — Amazon Web Services (AWS)

## Recommended Configuration

| Setting | Value |
|---------|-------|
| Instance | `g5.xlarge` |
| GPU | NVIDIA A10G |
| Image | Deep Learning AMI |
| Storage | 100 GB |

### Create an SSH Key

```
EC2 → Key Pairs
```

```bash
chmod 400 ~/.ssh/ai-keypair.pem
```

### Launch the Instance

```
EC2 → Launch Instance
```

Allow SSH from your IP.

### Connect

```bash
ssh -i ~/.ssh/ai-keypair.pem ubuntu@<PUBLIC_IP>
```

### Verify CUDA

```bash
nvidia-smi
```

### Activate PyTorch

```bash
conda activate pytorch
```

### Run Benchmark

```bash
python benchmark.py
```

---

# Part B — Google Cloud Platform (GCP)

## Recommended Configuration

| Setting | Value |
|---------|-------|
| Machine | `g2-standard-8` |
| GPU | NVIDIA L4 |
| Image | Ubuntu 22.04 |
| Storage | 100 GB |

### Enable Compute Engine

```
APIs & Services → Compute Engine API
```

### Create the VM

```
Compute Engine → VM Instances
```

### Connect

```bash
gcloud compute ssh ai-gcp-lab --zone=<ZONE>
```

### Install Drivers

```bash
sudo ubuntu-drivers autoinstall

sudo reboot
```

### Verify GPU

```bash
nvidia-smi
```

### Install PyTorch

```bash
pip install torch torchvision torchaudio \
--index-url https://download.pytorch.org/whl/cu121
```

### Run Benchmark

```bash
python benchmark.py
```

---

# Part C — Microsoft Azure

## Recommended Configuration

| Setting | Value |
|---------|-------|
| VM | Standard_NC4as_T4_v3 |
| GPU | NVIDIA T4 |
| Image | Azure ML DLVM (Ubuntu) |
| Storage | 100 GB |

### Create a Virtual Machine

```
Azure Portal → Virtual Machines
```

Select the **Azure Machine Learning Deep Learning VM** image.

### Connect

```bash
ssh azureuser@<PUBLIC_IP>
```

### Verify GPU

```bash
nvidia-smi
```

The DLVM already includes CUDA and PyTorch.

### Run Benchmark

```bash
python benchmark.py
```

---

# Collect Results

Record the following information for every provider.

| Cloud | GPU | PyTorch Version | Runtime (s) | On-demand Cost/hr |
|--------|-----|-----------------|------------:|------------------:|
| AWS | | | | |
| Google Cloud | | | | |
| Azure | | | | |

Example:

| Cloud | GPU | Runtime | Cost/hr |
|--------|------|---------:|---------:|
| AWS | A10G | 12.4 s | \$1.20 |
| GCP | L4 | 10.8 s | \$0.95 |
| Azure | T4 | 17.6 s | \$0.90 |

---

# Discussion

Answer the following questions:

1. Which cloud delivered the fastest execution time?

2. Which provider offered the best performance per dollar?

3. How do the GPU architectures (T4, L4, A10G) differ?

4. Would Spot/Preemptible instances change your decision?

5. Which cloud would you choose for:
   - Model training
   - Model serving
   - Large-scale experimentation

---

# Deliverables

Submit:

- Screenshot of `nvidia-smi` from each cloud
- Output of `benchmark.py`
- Completed comparison table
- A short discussion (≈200 words)

---

# Cleanup

After completing the benchmark:

- Stop or terminate all virtual machines.
- Delete unused disks.
- Delete unused public IP addresses.
- Verify there are no active GPU instances in any cloud provider.

---

# Summary

In this lab you learned how to:

- Deploy GPU virtual machines on AWS, Google Cloud, and Azure
- Verify CUDA installation
- Execute identical PyTorch workloads
- Measure GPU performance
- Compare performance against infrastructure cost
- Select the most suitable cloud platform for different AI workloads
   - Model serving
   - Large-scale experimentation

---

# Deliverables

Submit:

- Screenshot of `nvidia-smi` from each cloud
- Output of `benchmark.py`
- Completed comparison table
- A short discussion (≈200 words)

---

# Cleanup

After completing the benchmark:

- Stop or terminate all virtual machines.
- Delete unused disks.
- Delete unused public IP addresses.
- Verify there are no active GPU instances in any cloud provider.

---

# Summary

In this lab you learned how to:

- Deploy GPU virtual machines on AWS, Google Cloud, and Azure
- Verify CUDA installation
- Execute identical PyTorch workloads
- Measure GPU performance
- Compare performance against infrastructure cost
- Select the most suitable cloud platform for different AI workloads