# Alpamayo-Redux: Reasoning Distillation for Efficient Autonomy

**Status:** Research In-Progress

**Base Model:** [NVIDIA Alpamayo-R1-10B](https://huggingface.co/nvidia/Alpamayo-R1-10B)

**Simulator:** [Alpasim](https://github.com/NVlabs/alpasim)

## 1. Abstract

This project conducts a rigorous ablation study on the Alpamayo Vision-Language-Action (VLA) architecture. The primary objective is to solve the **inference-latency vs. safety trade-off** in autonomous driving.

I hypothesize that the safety benefits of "System 2" explicit reasoning (found in large models like Alpamayo-10B + CF-VLA) can be distilled into a "System 1" efficient architecture (Alpamayo-1B + LCDrive) using closed-loop simulation feedback. The ablation matrix systematically isolates the contributions of inference-time reasoning, closed-loop training, teacher quality, latent reasoning transfer, and post-training RL.

---

## 2. Experimental Design (The Ablation Matrix)

### Core Spine

| ID | Params | Inference Reasoning | Training Data Regime                          | Post-Training | Purpose / What it isolates                              |
| -- | -----: | ------------------- | --------------------------------------------- | ------------- | ------------------------------------------------------- |
| S0 |    10B | None                | Open-loop BC (official)                       | None          | Anchor: baseline capability + latency                   |
| S1 |    10B | **CF-VLA**          | Open-loop BC                                  | None          | Effect of **explicit reasoning at inference**           |
| S2 |    10B | None                | **RoaD**                                      | None          | Effect of **closed-loop SFT alone**                     |
| S3 |    10B | **CF-VLA**          | **RoaD**                                      | None          | "Teacher": best safety (slow)                           |
| S4 |     1B | None                | Distill from S0 (action-only)                 | None          | Small model baseline; isolates size + naive distill     |
| S5 |     1B | None                | Distill from **S3** (action-only)             | None          | Effect of **better teacher** without reasoning transfer |
| S6 |     1B | **LCDrive**         | Distill from **S3** (action + latent targets) | None          | Effect of **latent reasoning** vs action-only           |
| S7 |     1B | **LCDrive**         | Distill from S3                               | **RL (GRPO)** | Effect of **post-training** on latent student           |

### Tiny Slice
**May or may not do dependent on compute resource availability**

| ID | Params | Inference Reasoning | Training        | Post-Training | Purpose                                |
| -- | -----: | ------------------- | --------------- | ------------- | -------------------------------------- |
| T0 |   0.5B | None                | Distill from S0 | None          | Floor baseline                         |
| T1 |   0.5B | None                | Distill from S3 | None          | Teacher quality matters at tiny scale? |
| T2 |   0.5B | **LCDrive**         | Distill from S3 | **RL (GRPO)** | Best shot at "tiny hero"               |

---

## 3. Core Technologies & Methodology

### 3.1. The Backbone: Alpamayo

**Alpamayo-R1** is a Vision-Language-Action (VLA) model that predicts control signals (waypoints/steering) conditioned on visual inputs and natural language instructions.

* **Role:** Serves as the base policy for all experiments.

### 3.2. The Teacher Stack (S3)

We construct a "super-expert" by combining two complementary frameworks:

* **CF-VLA (Counterfactual Vision-Language-Action):**
	* *What it is:* A reasoning module that allows the model to "think before acting." It generates meta-actions, simulates outcomes, and self-corrects if a safety violation is detected.
	* *Role:* Acts as the **Inference-Time Supervisor**. It provides the "Reasoning Trace" (Why did we stop? Because the pedestrian might cross).


* **RoaD (Rollouts as Demonstrations):**
	* *What it is:* A training protocol for Closed-Loop Supervised Fine-Tuning (CL-SFT). Instead of static datasets, it trains on the model's own simulation rollouts.
	* *Role:* **The Training Protocol.** We use CF-VLA to guide the rollouts. If the model drifts, CF-VLA intervenes. RoaD then fine-tunes the model to recover from these drifts.



### 3.3. The Student Stack (S6/S7)

We aim to achieve Teacher-level performance on a 1B parameter budget using latent efficiency:

* **LCDrive (Latent Chain-of-Thought):**
	* *What it is:* A method to compress "reasoning" into high-dimensional latent vectors rather than slow English text tokens.
	* *Role:* Enables the 1B model to "reason" about the scene without the latency penalty of generating full text.


* **GRPO (Group Relative Policy Optimization):**
	* *What it is:* A memory-efficient Reinforcement Learning algorithm (critic-less PPO). It samples groups of outputs and rewards them relative to the group average.
	* *Role:* **The Optimizer.** We sample  latent reasoning paths from LCDrive. The best paths (validated by Alpasim driving scores) are reinforced.
	* *Advantage:* More stable than standard PPO for reasoning models; requires less VRAM.



---

## 4. Evaluation Framework

All models are evaluated in **Alpasim**, a high-fidelity closed-loop simulator.

**Dataset:** PhysicalAI-AV-NuRec (910 test scenes).

### Metrics

1. **Safety (Primary):**
* **Collision Rate:** Percentage of scenarios ending in at-fault collisions.
* **Driving Score:** Normalized score based on distance traveled without infractions.


2. **Reasoning (Secondary):**
* **Reasoning Consistency:** (For S6/S7) Does the latent state decode to a valid textual explanation of the maneuver?


3. **Control (Tertiary):**
* **minADE (Minimum Average Displacement Error):** Evaluated at  horizons.
* **Offroad Rate:** Frequency of lane violations.



---

## 5. Implementation Roadmap

### Phase 1: Baselines (S0, S4)

* [ ] Set up Alpasim environment with NuRec scenes.
* [ ] Evaluate pre-trained **S0 (10B)** to establish baseline metrics.
* [ ] Train **S4 (1B)** via standard behavioral cloning distillation from S0.

### Phase 2: The Teacher (S1, S2, S3)

* [ ] Implement **CF-VLA** wrapper for Alpamayo.
* [ ] Evaluate **S1** (S0 + CF-VLA inference reasoning).
* [ ] Implement **RoaD** data collection pipeline.
* [ ] Train **S2** on closed-loop SFT data (RoaD without CF-VLA guidance).
* [ ] **Data Generation:** Run rollouts where CF-VLA acts as the "expert" interventionist.
* [ ] Train **S3** on CF-VLA-guided rollouts.

### Phase 3: The Students (S5, S6, S7)

* [ ] Train **S5 (1B)** via action-only distillation from S3.
* [ ] Implement **LCDrive** architecture (latent head) for the 1B model.
* [ ] Train **S6** with action + latent targets from S3.
* [ ] Implement **GRPO** trainer.
* [ ] Train **S7** using RL feedback from Alpasim.

### Phase 4: Tiny Slice (T0, T1, T2) — Optional

* [ ] Train **T0 (0.5B)** via distillation from S0.
* [ ] Train **T1 (0.5B)** via distillation from S3.
* [ ] Train **T2 (0.5B)** with LCDrive + GRPO.

---

## 6. References

1. **Alpamayo-R1:** *Bridging Reasoning and Action Prediction for Generalizable Autonomous Driving in the Long Tail.* [arXiv:2511.00088](https://arxiv.org/abs/2511.00088)
2. **RoaD:** *Rollouts as Demonstrations for Closed-Loop Supervised Fine-Tuning of Autonomous Driving Policies.* [arXiv:2512.01993](https://arxiv.org/abs/2512.01993)
3. **CF-VLA:** *Self-Reflective Vision-Language-Action Model with Adaptive Reasoning.* [arXiv:2512.24426](https://arxiv.org/abs/2512.24426)
4. **LCDrive:** *Latent Chain-of-Thought World Modeling for End-to-End Driving.* [arXiv:2512.10226](https://arxiv.org/abs/2512.10226)
5. **GRPO:** *Pushing the Limits of Mathematical Reasoning in Open Language Models.* [arXiv:2402.03300](https://arxiv.org/abs/2402.03300)