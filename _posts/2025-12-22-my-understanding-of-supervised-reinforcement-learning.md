# Supervised Reinforcement Learning (SRL): Fine-Tuning Reasoning Models Step by Step

Training language models for **complex, multi-step reasoning** remains a fundamental challenge. In such tasks, a single incorrect intermediate step can derail the entire reasoning chain—even if many parts of the solution are correct. Unfortunately, most standard fine-tuning approaches fail to handle this gracefully.

Here I share my understanding of **Supervised Reinforcement Learning (SRL)**, a fine-tuning framework proposed by Google, which reframes reasoning as a **step-wise decision-making problem** and provides dense, stable supervision for learning structured reasoning.


## Why Standard Fine-Tuning Struggles with Reasoning

### Limitations of Supervised Fine-Tuning (SFT)

Supervised Fine-Tuning (SFT) trains models using next-token prediction over full expert solutions. While effective for surface-level imitation, SFT enforces **rigid token-level copying**:

- The model must reproduce the *entire* expert trajectory exactly  
- Any deviation—even if logically valid—is penalized  
- Generalization beyond the training data is limited  

### Limitations of Outcome-Based RL (RLVR / GRPO)

Outcome-based reinforcement learning methods, such as RL with verifiable rewards (RLVR), optimize only for the **final answer**:

- Intermediate reasoning is ignored  
- Partially correct solutions still receive zero or negative reward  
- Sparse rewards lead to unstable training  
- Long-horizon reasoning problems become difficult to explore  

As a result, many complex reasoning tasks become effectively **intractable** under purely outcome-based objectives.


## Core Idea of Supervised Reinforcement Learning (SRL)

SRL reframes problem solving as a **sequential decision-making process**.

Instead of:
- Imitating the entire expert trajectory (SFT), or
- Optimizing only the final outcome (RLVR),

**SRL trains the model to reproduce the sequence of key reasoning actions** underlying expert solutions, using an RL-style objective with **step-wise supervision**.

Key ideas:
- Expert demonstrations are decomposed into **meaningful reasoning steps**
- Each step is treated as an **action**
- The model receives a **reward at every step**, not just at the end


## How SRL Works (High-Level Overview)

The figure below illustrates SRL's step-wise training procedure, where partial trajectories are used to supervise each reasoning step independently.

![SRL Step-Wise Training](/assets/images/SRL%20step-wise.png)

*Image generated using Google's Nano Banana*

At each step *k*, the policy model generates two components:

1. An internal monologue, explicitly encapsulated within `<think>...</think>` tags.
2. A step action *âₖ*, representing the model's external reasoning decision.

While the internal monologue is not supervised or compared to the teacher, the `<think>` tag enforces a clear separation between private reasoning and externally evaluated actions. Only the step action contributes to the reward signal.


## Comparing Training Paradigms

| Method | Needs Teacher Demonstration | Reward Type | What It Trains |
|--------|----------------------------|-------------|----------------|
| **SFT** | Yes | None | Full trajectory imitation |
| **SRL** | Yes | Step-wise similarity | Reasoning steps |
| **RLVR / GRPO** | No | Final verifiable reward | Outcome correctness |


## Step-Wise Training Data Construction

SRL uses a powerful **teacher model** θ_expert to generate expert solution trajectories.

Given a solution with **N reasoning steps**, SRL constructs **N − 1 partial trajectories**.  
For each step *k* ∈ {1, ..., N-1}:

```
x_step_k = [x, y_step_1, ..., y_step_{k-1}]
```

The model is trained to predict: **y_step_k**

This transforms a single expert solution into many training examples that teach the model how to proceed correctly from different intermediate states.


## Supervising Reasoning Steps (Not Final Answers)

The teacher trajectory is converted into a sequence of **actions**:

```
a₁, a₂, a₃, ..., aₖ
```

Each *aₖ* represents a meaningful reasoning decision.

At step *k*, the policy model is given:
- The original query
- The context of all previous steps *a₁ ... aₖ₋₁*

The model generates:
1. An internal reasoning trace (kept private)
2. A **step action** *âₖ*

Only the action is compared with the teacher's action.  
The internal reasoning remains free and unsupervised.


## Step-Wise Reward via Sequence Similarity

For each step *k*:

- Model action: *âₖ*
- Teacher action: *aₖ*

The reward is computed using a **sequence similarity function**:

```
rₖ = R(ŷ_step_k, y_step_k) = 2M / T
```

Where:
- **T** = total number of elements in both sequences combined  
- **M** = total number of elements in all non-overlapping matching blocks  

The similarity is computed by first identifying the longest contiguous matching subsequence and then recursively searching the remaining segments.

Although SRL provides supervised, step-wise rewards, optimization is performed using a reinforcement learning objective. Specifically, the paper employs Group Relative Policy Optimization (GRPO) with a KL-divergence penalty that constrains the policy from drifting too far from a reference model.

This combination allows SRL to benefit from dense supervision while retaining the stability guarantees of modern RL fine-tuning methods.

### Format Enforcement

If the model output fails to follow the required format, a negative reward is assigned:

```
r(ŷ, y) = R(ŷ, y)   if output follows format
        = -1        otherwise
```


## Dynamic Sampling for SRL

Not all training samples provide useful learning signals.

If all rollouts for a sample yield rewards with near-zero variance, the resulting advantage is weak. SRL therefore filters such samples and retains a sample only if:

```
std({r₁, r₂, ...}) > ε
```

This ensures training focuses on examples that meaningfully distinguish correct and incorrect reasoning behavior.


## Practical Insights from the Authors

### Out-of-Order Reasoning Steps

What if the model uses a different but correct reasoning order?

In some tasks, multiple reasoning orders may be logically valid (e.g., A → B → C vs. B → A → C).

**Insights from the authors:**
- Providing **step titles or step-level prompts** from the teacher helps align student behavior
- Using **answer augmentation**, where multiple teacher trajectories are generated per step, further improves robustness
- The reward is taken as the **maximum similarity score across teacher variants**, allowing flexibility in reasoning order


### Combining Intermediate and Final Rewards

The authors explored combining step-wise SRL rewards with final outcome rewards (RLVR). However, this introduced the challenge of **reward balancing**.

**Final approach used in the paper:**
1. Train the model using **SRL** to learn structured reasoning
2. Fine-tune using **RLVR** to optimize final outcomes

This staged strategy avoids instability while preserving both step-wise correctness and goal achievement.


## Key Takeaways

- SRL reframes reasoning as **sequential decision-making**
- It provides **dense, stable supervision** at every reasoning step
- Internal reasoning is free and not directly supervised
- Training scales efficiently and generalizes better than SFT
- A practical pipeline is **SRL → RLVR**

SRL bridges the gap between imitation learning and outcome-based reinforcement learning, making it a powerful framework for training reasoning-capable language models.

## References
[Supervised Reinforcement Learning: From Expert Trajectories to Step-wise Reasoning.](https://doi.org/10.48550/arXiv.2510.25992)