# Alpamayo Ablation Study

A research study comparing different training strategies for the Alpamayo vision-language-action model for autonomous driving.

## Models

| Model | Description | Method |
|-------|-------------|--------|
| **A** | Alpamayo-R1-10B | Pre-trained baseline (BC) |
| **B** | Alpamayo-R1-10B + RoaD | Closed-loop SFT with expert-guided rollouts |
| **D** | Alpamayo-R1-1B | Knowledge distillation from A |
| **F** | Alpamayo-R1-1B + GRPO | RL fine-tuning with group relative policy optimization |

## Methods Overview

### RoaD (Rollouts as Demonstrations)
Closed-loop supervised fine-tuning that uses the policy's own expert-guided rollouts as training data:
- Sample K=64 trajectory candidates per step
- Select trajectory closest to expert demonstration
- Recovery mode when deviation exceeds threshold
- Fine-tune with frozen encoders (~4.2k steps)

### GRPO (Group Relative Policy Optimization)
Memory-efficient RL algorithm (PPO variant without critic):
- Sample G trajectories per observation
- Compute group-relative advantages: `A_i = (r_i - mean) / std`
- Policy gradient with clipping
- Reward: safety + progress + comfort

## Evaluation

Using **Alpasim** simulator with 910 test scenes from PhysicalAI-AV-NuRec dataset.

**Metrics:**
- Driving Score (km between incidents)
- Collision Rate (at-fault)
- Offroad Rate
- minADE @ {0.5, 1.0, 2.0, 3.0, 6.4}s

## Resources

- Alpamayo codebase: `~/Documents/alpamayo`
- Alpasim simulator: `~/Documents/alpasim`

## References

1. [Alpamayo-R1 Paper](https://arxiv.org/abs/2511.00088) - VLA with Chain of Causation reasoning
2. [RoaD Paper](https://arxiv.org/abs/2512.01993) - Closed-loop SFT for driving policies
3. [GRPO Paper](https://arxiv.org/abs/2402.03300) - Group Relative Policy Optimization (DeepSeekMath)
