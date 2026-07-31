# 🧪 273. Lab – Defend Against Adversarial Attacks

## 🎯 Objective
- Understand how adversarial examples affect models.
- Generate attacks (**FGSM**, **PGD**).
- Apply **adversarial training** to improve robustness.
- Compare accuracy on clean vs. adversarial test sets.

---

## Step 1: Environment Setup

```bash
pip install torch torchvision
```

> **✅ Expected:** PyTorch installed and working with CUDA (if available).

---

## Step 2: Load Dataset & Model

We’ll use **MNIST** with a simple CNN.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms

# Dataset
transform = transforms.Compose([transforms.ToTensor()])
trainset = torchvision.datasets.MNIST(root='./data', train=True, download=True, transform=transform)
testset = torchvision.datasets.MNIST(root='./data', train=False, download=True, transform=transform)

trainloader = torch.utils.data.DataLoader(trainset, batch_size=64, shuffle=True)
testloader = torch.utils.data.DataLoader(testset, batch_size=1000, shuffle=False)

# Model
class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(1, 32, 3, 1)
        self.conv2 = nn.Conv2d(32, 64, 3, 1)
        self.fc1 = nn.Linear(9216, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        x = torch.relu(self.conv1(x))
        x = torch.relu(self.conv2(x))
        x = torch.flatten(x, 1)
        x = torch.relu(self.fc1(x))
        return self.fc2(x)

device = "cuda" if torch.cuda.is_available() else "cpu"
model = Net().to(device)
```

---

## Step 3: Train Baseline Model

```python
optimizer = optim.Adam(model.parameters(), lr=0.001)
criterion = nn.CrossEntropyLoss()

for epoch in range(1):  # short training for demo
    for x, y in trainloader:
        x, y = x.to(device), y.to(device)
        optimizer.zero_grad()
        output = model(x)
        loss = criterion(output, y)
        loss.backward()
        optimizer.step()
```

> **✅ Expected:** Model reaches ~97% accuracy on clean MNIST.

---

## Step 4: Implement FGSM Attack

```python
def fgsm_attack(model, data, target, epsilon):
    data.requires_grad = True
    output = model(data)
    loss = criterion(output, target)
    model.zero_grad()
    loss.backward()
    data_grad = data.grad.data
    perturbed_data = data + epsilon * data_grad.sign()
    return torch.clamp(perturbed_data, 0, 1)
```

> **✅ Expected:** Perturbed images look nearly identical but fool the model.

---

## Step 5: Test on Adversarial Examples

```python
def test_attack(model, loader, epsilon):
    correct, total = 0, 0
    for data, target in loader:
        data, target = data.to(device), target.to(device)
        adv_data = fgsm_attack(model, data, target, epsilon)
        output = model(adv_data)
        pred = output.argmax(dim=1, keepdim=True)
        correct += pred.eq(target.view_as(pred)).sum().item()
        total += target.size(0)
    return 100. * correct / total

print("Accuracy on clean test set:", test_attack(model, testloader, 0.0))
print("Accuracy on FGSM adversarial examples (ε=0.2):", test_attack(model, testloader, 0.2))
```

> **✅ Expected:** Clean accuracy ~97%, adversarial accuracy drops sharply (~10–20%).

---

## Step 6: Adversarial Training Defense

```python
adv_model = Net().to(device)
optimizer = optim.Adam(adv_model.parameters(), lr=0.001)

for epoch in range(1):  
    for data, target in trainloader:
        data, target = data.to(device), target.to(device)
        optimizer.zero_grad()
        
        # Generate adversarial samples
        adv_data = fgsm_attack(adv_model, data, target, epsilon=0.2)
        
        # Train on clean + adversarial
        output_clean = adv_model(data)
        output_adv = adv_model(adv_data)
        loss = criterion(output_clean, target) + criterion(output_adv, target)
        
        loss.backward()
        optimizer.step()
```

> **✅ Expected:** Model learns to resist FGSM perturbations.

---

## Step 7: Evaluate Robustness

```python
print("Baseline Model (ε=0.2 FGSM):", test_attack(model, testloader, 0.2))
print("Adversarially Trained Model (ε=0.2 FGSM):", test_attack(adv_model, testloader, 0.2))
```

> **✅ Expected:**
> - **Baseline model collapses** (~10–20% accuracy).
> - **Adversarially trained model performs much better** (~70–80% accuracy).

---

## Step 8 (Optional Extensions)

- Try **PGD attack** (stronger than FGSM).
- Test on **different $\epsilon$ values** (0.05, 0.1, 0.3).
- Use **preprocessing defenses** (JPEG compression, Gaussian noise).

---

## 📝 Wrap-Up

- Adversarial examples = real threat to deployed AI models.
- FGSM showed how small perturbations fool a baseline model.
- Adversarial training improves robustness significantly.
- Defense is not perfect — it’s an **arms race** between attackers & defenders.
