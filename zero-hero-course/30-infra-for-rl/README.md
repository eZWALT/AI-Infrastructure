# Day 210 Lab – Train RL Agent with Ray RLlib

## Learning Goals

* Install and configure **Ray + RLlib**
* Train an RL agent on a standard Gym environment
* Monitor training metrics and visualize results
* Save and reload trained policies for inference

---

## 0) Prerequisites

Make sure you have the following ready:

* **Python 3.9+**

### Install Dependencies

Run these commands in your terminal:

```bash
mkdir rllib-lab && cd rllib-lab
python -m venv .venv

# Activate virtual environment
# On macOS/Linux:
source .venv/bin/activate
# On Windows:
# .venv\Scripts\activate

pip install "ray[rllib]" gymnasium[classic_control] matplotlib
```

---

## 1) Quick Test: Hello RLlib

Create `train_cartpole.py`:

```python
import ray
from ray.rllib.algorithms.ppo import PPOConfig

# Start Ray
ray.init(ignore_reinit_error=True)

# Configure PPO for CartPole
config = (
    PPOConfig()
    .environment("CartPole-v1")
    .rollouts(num_rollout_workers=1)
    .training(train_batch_size=4000)
    .framework("torch")
)

# Build Trainer
algo = config.build()

# Train for N iterations
for i in range(5):
    result = algo.train()
    print(f"Iter: {i}, reward_mean: {result['episode_reward_mean']:.2f}")
```

### Run Training

```bash
python train_cartpole.py
```

> **Observation:** You should see `reward_mean` increase over iterations as the agent learns.

---

## 2) Save & Load Policy

Extend your training script (`train_cartpole.py`) to save and restore checkpoints:

```python
# Save checkpoint
checkpoint = algo.save()
print("Checkpoint saved at:", checkpoint)

# Load checkpoint later
algo.restore(checkpoint)
```

---

## 3) Run Inference with Trained Policy

Create `inference.py`:

```python
import gymnasium as gym
import ray
from ray.rllib.algorithms.ppo import PPOConfig

# Init Ray & load trained policy
ray.init()
config = PPOConfig().environment("CartPole-v1").framework("torch")
algo = config.build()

# Replace with your actual checkpoint path printed in step 2
algo.restore("last_checkpoint_path")

env = gym.make("CartPole-v1", render_mode="human")
obs, _ = env.reset()

done = False
while not done:
    action = algo.compute_single_action(obs)
    obs, reward, terminated, truncated, _ = env.step(action)
    done = terminated or truncated

env.close()
```

### Run Visual Inference

```bash
python inference.py
```

> 🎉 You’ll see a GUI window pop up showing the trained agent balancing the CartPole.

---

## 4) Visualize Training Rewards

Add plotting to your training pipeline using `matplotlib`:

```python
import matplotlib.pyplot as plt

rewards = []
for i in range(20):
    result = algo.train()
    rewards.append(result["episode_reward_mean"])

plt.plot(rewards)
plt.xlabel("Iteration")
plt.ylabel("Mean Episode Reward")
plt.title("PPO Training on CartPole")
plt.show()
```

---

## 5) Stretch Goals

* Swap the environment to **`MountainCar-v0`** or **`LunarLander-v2`**.
* Try a different algorithm, such as **DQN** instead of PPO.
* Run distributed training across multiple rollout workers:
  ```python
  .rollouts(num_rollout_workers=4)
  ```
* Deploy your trained policy as a **FastAPI microservice** for online inference.

> ✅ **Outcome:** You trained and served an RL agent with Ray RLlib, monitored rewards, and deployed a checkpoint for inference.
