---
layout: page
title: Catching a Moving Target
description: A C++ interception planner that captured a moving target on all 9 test maps, including grids with 4.26 million cells.
img: assets/img/projects/moving-target/map4.png
importance: 1
category: robotics
---

I implemented a robot motion planner that intercepts a moving target across obstacle-filled, variable-cost grids. The planner combines **space-time weighted A\*** with **spatial breadth-first search (BFS)**, choosing its search strategy based on the problem size. In the experiments, it **caught the target on all nine test maps**, with **eight runs completing in under one second of wall time**.

## The problem

Reaching a moving target requires arriving at the right place **at the right time**. Given a map, the robot's starting position, and the target's known future trajectory, I needed to find a collision-free interception while accounting for terrain cost. The robot could move to eight neighboring cells, and computation time affected the simulator's clock.

Adding time to the search state makes the problem much larger: a grid with millions of cells can produce hundreds of millions of possible position-time combinations. The implementation needed to balance capture feasibility, path cost, and computation.

## My solution

- **Search for an interception along the entire target trajectory.** Weighted A* searches states of the form `(x, y, time)`, treating each future target position and its timestamp as a possible goal. A reachability heuristic uses Chebyshev distance to discard states that cannot reach any future target visit in time.
- **Adapt search aggressiveness to terrain variation.** I set the heuristic weight from the mean and standard deviation of traversable-cell costs: `w = 1 + 4 / (1 + σ/μ)`. Uniform positive-cost terrain gives a weight of 5; greater relative variation lowers the weight toward 1, encouraging more exploration. The observed A* weights ranged from **2.263 to 5.000**.
- **Use a smaller search model for large maps.** When `width × height × trajectory length` reaches **200 million**, the planner switches to spatial BFS. It finds a reachable cell the target will visit later, then lets the robot wait there. Each spatial cell is visited at most once, avoiding a full search over position and time.
- **Keep execution and search efficient.** I used 64-bit state keys, a priority queue with lazy replacement, hash-based state lookup, and a cached path deque. A Python results script checks legal moves, traversable positions, capture status, and timestamp-based path cost, then generates the trajectory plots.

## Measured results

The measurements cover one Release-build run per map on Linux using GNU C++ 15.2.0. **Wall time includes startup, map parsing, planning, execution, and trajectory output**; simulated capture time is a separate measurement.

| Map | Grid size     | Search      | Wall time (s) | Simulated capture time (s) | Robot moves | Path cost |
| --- | ------------- | ----------- | ------------- | -------------------------- | ----------- | --------- |
| 1   | 1,825 × 2,332 | BFS         | 0.660         | 5,344                      | 421         | 5,344     |
| 2   | 1,825 × 2,332 | BFS         | 0.729         | 952                        | 952         | 4,577,351 |
| 3   | 473 × 436     | Weighted A* | 0.049         | 241                        | 241         | 241       |
| 4   | 473 × 436     | Weighted A* | 0.650         | 375                        | 375         | 375       |
| 5   | 200 × 200     | Weighted A* | 0.290         | 181                        | 181         | 1,063     |
| 6   | 200 × 200     | Weighted A* | 0.898         | 140                        | 140         | 539       |
| 7   | 400 × 400     | Weighted A* | 0.029         | 250                        | 250         | 250       |
| 8   | 400 × 400     | Weighted A* | 0.961         | 451                        | 451         | 451       |
| 9   | 400 × 400     | Weighted A* | 2.356         | 379                        | 378         | 379       |

**Impact:** the planner achieved **9/9 successful captures** across grids ranging from **40,000 to 4,255,900 cells**. Both largest-map runs finished in **under 0.73 seconds** of wall time. Map 1 illustrates interception by waiting: the robot made **421 moves** before capture at simulated time **5,344 seconds**, rather than following the target's entire route.

## Trajectories from the experiments

{% include figure.liquid loading="eager" path="assets/img/projects/moving-target/map4.png" title="Weighted A* interception on map 4" alt="Map 4 terrain with the blue robot path avoiding obstacles and meeting the orange target path at a magenta capture star." class="img-fluid rounded z-depth-1" %}

<div class="caption">
Map 4: weighted A* captured the target in 375 moves with path cost 375 and 0.650 seconds of wall time. The terrain-dependent heuristic weight was 2.263.
</div>

{% include figure.liquid path="assets/img/projects/moving-target/map1.png" title="BFS interception on map 1" alt="Map 1 obstacle grid showing a short blue robot route to a waiting location where the orange target trajectory reaches the magenta capture star." class="img-fluid rounded z-depth-1" %}

<div class="caption">
Map 1: BFS found an interception location on a 4.26-million-cell grid. Blue shows the executed robot path, orange the target trajectory, and the magenta star the capture location. Waiting is not separately visible in the spatial trace.
</div>

## What the results establish

These experiments demonstrate successful interception on the benchmark maps. Weighted A* does not guarantee minimum path cost, and the BFS branch prioritizes a feasible waiting location over terrain cost—visible in map 2's high cost. The A* implementation omits explicit waiting successors and does not reopen expanded states. The cached plan also assumes one simulated time step per transition, so timing changes on slower machines can affect capture. The experiments include no baseline comparison from which to claim a speedup.

**Tools:** C++, CMake, Python, NumPy, Matplotlib.
