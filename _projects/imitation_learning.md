---
layout: page
title: Learning Locomotion from Demonstrations
description: Behavioral cloning and DAgger across five simulated robots, bringing final imitation returns to 95–100% of expert demonstration means.
img: assets/img/projects/imitation-learning/dagger-humanoid.png
importance: 2
category: robotics
---

I implemented an imitation-learning pipeline that trains continuous-control policies from expert demonstrations, then improves them with **DAgger (Dataset Aggregation)**. Across five simulated locomotion environments, DAgger increased final evaluation return over behavioral cloning and reached approximately **95–100% of the expert demonstration mean**.

## The problem

A policy can match expert actions on a fixed dataset and still fail when controlling a robot. Small action errors move the robot into states that the demonstrations rarely cover, allowing errors to compound. This distribution shift was especially visible in Humanoid: behavioral cloning achieved a mean return of only **295.47**, compared with an expert demonstration mean of **10,344.52**.

I evaluated this problem on **Ant, Humanoid, Walker2d, Hopper, and HalfCheetah** in MuJoCo, and investigated whether collecting expert labels on the learner's own states could recover reliable locomotion.

## My solution

- **Learn a policy from demonstrations.** I implemented a PyTorch multilayer perceptron trained with mean squared error between predicted actions and expert actions, using Adam for optimization.
- **Collect data where the learner actually goes.** After the initial demonstration-based update, DAgger rolls out the current policy, asks the expert for actions at those visited states, and adds the relabeled transitions to a replay buffer. Subsequent updates train on the aggregated dataset.
- **Compare performance across robot morphologies.** I logged evaluation means and standard deviations for behavioral cloning and DAgger in all five environments. The saved DAgger runs contain 10 training iterations per environment, with 20 for Humanoid.
- **Test network capacity.** I also swept Ant policy depth through **2, 4, 8, 16, 32, 64, and 128 hidden layers**, keeping width at 64. This tested whether adding capacity could compensate for limited demonstration coverage.

The main Ant and Humanoid configurations used **two hidden layers of 64 units**, **2,000 initial demonstration transitions**, a **0.005 learning rate**, and **1,000 gradient updates per training iteration**. Evaluation used a transition budget of 10,000 rather than a fixed number of episodes.

## Measured results

The following values come from the saved training logs. Behavioral cloning is its single logged evaluation; DAgger is the **final logged evaluation**, rather than its best intermediate result. Expert means come from two demonstration trajectories per environment. Returns are environment-specific rewards, so their absolute values should not be compared across robots.

| Environment | Expert mean | Behavioral cloning mean | Final DAgger mean | DAgger / expert | Training iterations |
| ----------- | ----------- | ----------------------- | ----------------- | --------------- | ------------------- |
| Ant         | 4,713.65    | 3,855.74                | 4,685.91          | 99.4%           | 10                  |
| Humanoid    | 10,344.52   | 295.47                  | 9,834.80          | 95.1%           | 20                  |
| Walker2d    | 5,566.85    | 2,200.57                | 5,550.04          | 99.7%           | 10                  |
| Hopper      | 3,772.67    | 988.09                  | 3,780.05          | 100.2%          | 10                  |
| HalfCheetah | 4,205.78    | 3,953.38                | 4,099.39          | 97.5%           | 10                  |

**Impact:** Humanoid's final mean return increased by approximately **33.3×** over the saved behavioral-cloning evaluation. Hopper improved by **3.8×**, and Walker2d by **2.5×**. The largest gains came from environments where fixed-data imitation struggled most.

## Visualizations from the experiments

{% include figure.liquid loading="eager" path="assets/img/projects/imitation-learning/dagger-humanoid.png" title="Humanoid imitation learning" alt="Humanoid DAgger evaluation return rises toward the expert demonstration mean over 20 training iterations, with episode standard deviation error bars." class="img-fluid rounded z-depth-1" %}

<div class="caption">
Humanoid: final DAgger mean return was 9,834.80. Error bars show one standard deviation across evaluation episodes within each iteration; they do not measure variation across training seeds.
</div>

{% include figure.liquid path="assets/img/projects/imitation-learning/dagger-ant.png" title="Ant imitation learning" alt="Ant DAgger learning curve compared with behavioral cloning and the expert demonstration mean." class="img-fluid rounded z-depth-1" %}

<div class="caption">
Ant: DAgger finished at 4,685.91, or 99.4% of the expert demonstration mean, after 10 training iterations.
</div>

{% include figure.liquid path="assets/img/projects/imitation-learning/network-depth.png" title="Policy depth experiment" alt="Ant behavioral cloning evaluation mean and standard deviation for networks ranging from 2 to 128 hidden layers." class="img-fluid rounded z-depth-1" %}

<div class="caption">
The separate Ant depth sweep showed a small improvement from two to four hidden layers, followed by declining returns as depth increased. Greater capacity alone did not resolve the imitation problem.
</div>

## What the results establish

These runs show that expert relabeling on learner-visited states can substantially improve imitation performance. They do not establish robustness across random seeds or deployment on physical robots. Expert comparisons use only two demonstration trajectories, and an evaluation exceeding that mean does not establish that the learner is consistently better than the expert. The depth sweep shows a performance trend, but does not by itself distinguish overfitting from optimization difficulty.

**Tools:** Python, PyTorch, NumPy, Gym, MuJoCo, TensorBoard, Matplotlib.
