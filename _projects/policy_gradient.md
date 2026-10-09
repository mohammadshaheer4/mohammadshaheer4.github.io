---
layout: page
title: Training Control Policies with Policy Gradients
description: A PyTorch policy-gradient pipeline exploring reward-to-go, learned baselines, and advantage estimation across five control environments.
img: assets/img/projects/policy-gradient/halfcheetah-comparison.png
importance: 3
category: robotics
---

I implemented a **policy-gradient reinforcement-learning pipeline** for discrete and continuous control, then evaluated how return estimation, batch size, learned baselines, and **generalized advantage estimation (GAE)** affect learning. The experiments span CartPole, InvertedPendulum, LunarLander, HalfCheetah, and Hopper.

## The problem

Learning directly from interaction requires assigning credit to actions whose consequences may arrive much later. Policy-gradient estimates can vary sharply between batches, making training slow or unstable. The challenge was to improve that learning signal while handling both discrete actions and continuous robot controls.

I compared alternative estimators and training configurations, using evaluation return over time to distinguish faster learning from a brief performance peak.

## My solution

- **Implement stochastic control policies.** I used categorical action distributions for discrete control and Gaussian policies with learned log standard deviations for continuous control. PyTorch and Adam optimize the negative mean of action log probability multiplied by estimated advantage.
- **Compare full-trajectory returns with reward-to-go.** The pipeline can assign each action the whole trajectory's discounted return or only the discounted rewards from that action onward. I also implemented advantage standardization.
- **Learn a value baseline.** A separate neural network predicts standardized return targets. Its predictions are converted back to the current batch's return scale before computing advantages.
- **Explore temporal credit assignment with GAE.** I implemented a backward recursion over temporal-difference residuals and compared **λ = 0, 0.95, 0.99, and 1** on Hopper with action noise of standard deviation 0.5.
- **Run controlled comparisons and parameter sweeps.** CartPole compares three estimator configurations at batch sizes of **1,500 and 6,000**. InvertedPendulum explores **25 batch-size/learning-rate combinations**. HalfCheetah explores batches of **10,000, 30,000, and 50,000**, learning rates of **0.005, 0.01, and 0.02**, and four combinations of reward-to-go and a learned baseline.

## Measured results

These values are **raw average evaluation returns from the saved logs**. Final return means the last logged training iteration; peak return is the highest logged evaluation mean within the same run. They are not averages across independent training seeds.

| Environment / configuration                                            | Training iterations | Final mean return | Peak mean return |
| ---------------------------------------------------------------------- | ------------------- | ----------------- | ---------------- |
| CartPole: reward-to-go + standardization, batch 6,000                  | 150                 | 200.00            | 200.00           |
| InvertedPendulum: batch 5,000, learning rate 0.009                     | 100                 | 1,000.00          | 1,000.00         |
| LunarLander: reward-to-go + learned baseline                           | 100                 | 209.94            | 218.51           |
| HalfCheetah: reward-to-go + baseline, batch 30,000, learning rate 0.02 | 100                 | 212.97            | 342.32           |
| Hopper: GAE λ = 0.95                                                   | 300                 | 371.78            | 1,019.03         |
| Hopper: GAE λ = 0.99                                                   | 300                 | 456.68            | 474.39           |

**CartPole:** reward-to-go configurations finished at the environment's maximum return of **200** for both batch sizes. With the smaller batch, the full-trajectory estimator without advantage standardization finished at **72.83**, despite reaching 200 earlier. The learning curves show why a peak alone is insufficient to judge stability.

**HalfCheetah:** at batch 30,000 and learning rate 0.02, the combined reward-to-go and baseline configuration finished at **212.97**. Reward-to-go alone finished at **128.44**, the baseline alone at **−25.22**, and neither technique at **−26.98**. Among the nine combined-estimator sweep runs, batch 30,000 and learning rate 0.02 produced the highest final and peak means.

**Hopper:** λ = 0.95 produced the highest intermediate mean, but λ = 0.99 finished with the highest mean among the four tested settings. This distinguishes the best observed peak from the best final evaluation.

## Visualizations from the experiments

{% include figure.liquid loading="eager" path="assets/img/projects/policy-gradient/halfcheetah-comparison.png" title="HalfCheetah estimator comparison" alt="HalfCheetah learning curves compare full-trajectory returns, a learned baseline, reward-to-go, and reward-to-go with a learned baseline." class="img-fluid rounded z-depth-1" %}

<div class="caption">
HalfCheetah: reward-to-go with a learned baseline outperformed the other three configurations in the recorded comparison. All use batch 30,000, learning rate 0.02, two hidden layers of 32 units, discount 0.95, and a 150-step episode limit.
</div>

{% include figure.liquid path="assets/img/projects/policy-gradient/cartpole-small-batch.png" title="CartPole with smaller batches" alt="CartPole average evaluation returns for three policy-gradient estimators using a batch size of 1500." class="img-fluid rounded z-depth-1" %}

{% include figure.liquid path="assets/img/projects/policy-gradient/cartpole-large-batch.png" title="CartPole with larger batches" alt="CartPole average evaluation returns for the same estimators using a batch size of 6000." class="img-fluid rounded z-depth-1" %}

<div class="caption">
CartPole: reward-to-go and advantage standardization improved convergence in these runs. The larger batch reduced the instability visible with the smaller full-trajectory configuration.
</div>

{% include figure.liquid path="assets/img/projects/policy-gradient/lunar-lander.png" title="Continuous landing control" alt="LunarLander continuous-control evaluation return improves during 100 policy-gradient training iterations." class="img-fluid rounded z-depth-1" %}

<div class="caption">
LunarLander: a two-layer, 64-unit policy used reward-to-go, a learned baseline, batch 10,000, learning rate 0.005, and discount 0.99. Final mean return was 209.94.
</div>

{% include figure.liquid path="assets/img/projects/policy-gradient/hopper-advantage-estimation.png" title="Hopper advantage estimation" alt="Hopper evaluation returns compare generalized advantage estimation with lambda values of 0, 0.95, 0.99, and 1." class="img-fluid rounded z-depth-1" %}

<div class="caption">
Hopper: intermediate λ values improved the observed returns over λ = 0. Training used two hidden layers of 32 units, batch 2,000, learning rate 0.001, discount 0.99, and noisy actions.
</div>

## What the results establish

These experiments demonstrate the implemented pipeline and show how estimator choices affect the recorded learning curves. They do not establish performance across random seeds, physical robots, or differently configured episode horizons. The saved GAE implementation resets the terminal value contribution but does not reset the recursive advantage contribution at episode boundaries; the Hopper comparisons therefore describe this implementation and need a corrected rerun before attributing the differences solely to λ.

**Tools:** Python, PyTorch, NumPy, Gym, MuJoCo, Box2D, TensorBoard, Matplotlib.
