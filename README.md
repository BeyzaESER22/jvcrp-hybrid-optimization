# JVCRP-PL ULTIMATE — Hybrid Combinatorial Optimization Engine

> **J**oint **V**ehicle **C**ourier **R**outing **P**roblem with **L**ockers  
> *Production-grade C++ solver for cost-optimal route planning in multi-tier delivery networks.*

---

## Table of Contents

1. [Problem Definition and Motivation](#1-problem-definition-and-motivation)
2. [System Architecture](#2-system-architecture)
3. [Solution Pipeline](#3-solution-pipeline)
4. [Analytical Oracle and Decision Mechanism](#4-analytical-oracle-and-decision-mechanism)
5. [Construction Heuristic](#5-construction-heuristic)
6. [24 Neighborhood Operators](#6-24-neighborhood-operators)
7. [Hybrid Optimization Engine — SA + VND + ALNS](#7-hybrid-optimization-engine--sa--vnd--alns)
8. [Cost Function](#8-cost-function)
9. [Configuration Reference](#9-configuration-reference)
10. [Build and Run](#10-build-and-run)
11. [Output Format](#11-output-format)
12. [Engineering Decisions and Design Rationale](#12-engineering-decisions-and-design-rationale)

---

## 1. Problem Definition and Motivation

JVCRP-PL is a three-dimensional extension of the classical VRP (Vehicle Routing Problem):

| Classical VRP | JVCRP-PL |
|---|---|
| Single vehicle fleet | Vehicles **+** crowd-sourced courier network |
| Direct customer delivery | Delivery: **direct home** or **via locker through courier** |
| Single capacity constraint | Vehicle capacity + courier capacity + **time window** penalty |

The real-world motivation is as follows: dispatching large vehicles to every address inside a city is both costly and inefficient. Locker points allow couriers to serve dense areas at lower cost; however, deciding which customer should be served through which channel is an NP-hard assignment problem.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        JVCRP-PL ENGINE                          │
│                                                                 │
│  ┌──────────┐   ┌──────────────────────────────────────────┐   │
│  │ Instance │   │          AnalyticalOracle                 │   │
│  │  · Nodes │──▶│  Feature Engineering → P(vehicle)        │   │
│  │  · Dist  │   │                      → P(courier)        │   │
│  │  · Matrix│   └─────────────────┬────────────────────────┘   │
│  └──────────┘                     │ Threshold = 0.50            │
│                                   ▼                             │
│              ┌────────────────────────────────────┐            │
│              │     ConstructionHeuristic           │            │
│              │  GRASP + Regret Insertion           │            │
│              │  Dynamic Cost Weighting (α·(1-P))  │            │
│              └────────────────┬───────────────────┘            │
│                               │ Initial Solution                │
│                               ▼                                 │
│              ┌────────────────────────────────────┐            │
│              │     Phase 1.5 — Route Smoothing    │            │
│              │     2-Opt Intra-Route              │            │
│              └────────────────┬───────────────────┘            │
│                               │                                 │
│                               ▼                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              AdvancedOptimizer (Phase 2)                │    │
│  │                                                        │    │
│  │   ┌──────────┐   ┌──────────┐   ┌──────────────────┐  │    │
│  │   │    SA    │   │   VND    │   │      ALNS         │  │    │
│  │   │Simulated │◀──│Variable  │◀──│Adaptive Large    │  │    │
│  │   │Annealing │   │Neighbor- │   │Neighborhood      │  │    │
│  │   │Metropolis│   │hood Desc.│   │Search (24 ops)   │  │    │
│  │   └──────────┘   └──────────┘   └──────────────────┘  │    │
│  │                                                        │    │
│  │   Diversification: Ruin & Recreate (Op 21)             │    │
│  │   Intensification: Periodic VND every 2000 iter        │    │
│  └────────────────────────────────────────────────────────┘    │
│                               │                                 │
│                               ▼  × 5 Multi-Start Runs          │
│              ┌────────────────────────────────────┐            │
│              │         Best Overall Solution       │            │
│              │         Final Schedule Report       │            │
│              └────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### Core Data Structures

**`Node`** — A vertex in the graph. The `NodeType` enum takes three values: `DEPOT` (source), `CUSTOMER` (demand point), and `LOCKER` (transfer cabinet). For every customer, `nearest_locker_id` and `dist_to_nearest_locker` are pre-computed; this information is critical in both the Oracle and the construction phase.

**`Route`** — An ordered node list (`path`) along with derived statistics such as `total_load`, `total_distance`, and `pickup_location_id`. Vehicle routes start and end at the depot; courier routes start at a locker and return to it.

**`Solution`** — Composed of a `vehicle_routes` vector, a `courier_routes` vector, and an `assignments` map. Whenever `calculateTotalCost()` is called, all cost components and the feasibility flag are recomputed from scratch.

**`RandomEngine`** — A wrapper around `std::mt19937`. Can be seeded with `time(nullptr)` for non-deterministic runs or with the fixed constant `seed = 42` for reproducible results.

---

## 3. Solution Pipeline

The solver consists of four phases:

```
Phase 0  →  Phase 1  →  Phase 1.5  →  Phase 2
Validation   Construction  Smoothing    Optimization
```

**Phase 0 — System Validation:** Total demand versus fleet capacity is checked. Execution continues even if capacity is exceeded (with a warning message); this design decision preserves flexibility.

**Phase 1 — GRASP-Based Construction:** Oracle scores are computed, customers are partitioned into three pools (vehicle candidates, courier candidates, uncertain), and inserted in priority order.

**Phase 1.5 — Route Smoothing:** 2-Opt is applied to all vehicle routes. This phase is self-contained and can be replaced in the future with VNS or Or-Opt.

**Phase 2 — Hybrid Optimization:** The SA + VND + ALNS trio runs until 50,000 iterations or the 600-second time limit is reached.

The entire phase pipeline is repeated 5 times (**Multi-Start**); the best overall solution is retained.

---

## 4. Analytical Oracle and Decision Mechanism

The Oracle is a feature engineering layer that computes `P(vehicle)` and `P(courier)` scores for each customer. It is **not** a machine learning model; it is a heuristically weighted scoring function — a deliberate design choice: operability without training data is prioritized.

### Feature Vector

| Feature | Definition | Normalization |
|---|---|---|
| `x1` — Locker Distance | Distance to nearest locker | `min(d / 50, 1.0)` |
| `x2` — Depot Distance | Euclidean distance to depot | `min(d / 150, 1.0)` |
| `x3` — Local Density | Number of neighbors within a 15-unit radius | `min(n / 5, 1.0)` |
| `x4` — Load Ratio | `demand / courier_capacity` | Natural `[0,1]` |
| `x6` — Urgency | 1 if slack time < 30, else 0 | Binary |

> `x5` is intentionally omitted; it is a reserved slot for a future categorical feature such as home-delivery preference.

### Score Formulas

```
P(courier) = (1 - norm_x1) × 0.4   # Proximity to locker
           + x3_density    × 0.4    # Clustering
           + (1 - x4)      × 0.2    # Light load

P(vehicle) = (1 - x3_density) × 0.5  # Outlier (sparse area)
           + norm_x1         × 0.3   # Distance to locker
           + x6_urgency      × 0.2   # Urgent delivery
```

For customers where `is_home_delivery_only = true`, `P(courier)` is forced to zero.

---

## 5. Construction Heuristic

### Dynamically Weighted Cost (Step 3)

The insertion cost is weighted not only by physical distance but also by the Oracle probability:

```
Cost(insert) = Δdistance + α × (1 - P)
```

The parameter `α = 100.0` (`alpha_penalty`) controls how strongly the probability influences the cost relative to distance. High `P` → low penalty → the node is pulled toward its preferred channel.

**Why this approach?** Classical insertion heuristics minimize distance alone. This formulation embeds the Oracle's channel decision into the construction phase as a soft constraint. When the optimization phase begins, the starting solution is already much better structured.

### Priority Ordering

1. **Vehicle candidates** are inserted first — these are outlier customers and serve as seeds for the routes.
2. **Courier candidates** are processed second — added to existing courier routes or a new courier is opened.
3. **Uncertain customers** are handled last — the courier channel is attempted first; if that fails, the vehicle is used.

Locker balancing and courier route optimization (greedy nearest-neighbor) are run at the end of construction.

---

## 6. 24 Neighborhood Operators

Operators are organized into four categories:

### Vehicle Operators (1–6)

| ID | Name | Description |
|---|---|---|
| 1 | `intraSwapV` | Swaps two customers within the same vehicle |
| 2 | `intraInsertV` | Removes a customer from its position and reinserts it elsewhere in the same vehicle |
| 3 | `intra2OptV` | Intra-vehicle 2-Opt (reverses a route segment) |
| 4 | `interSwapV` | Swaps customers between two vehicles (capacity-checked) |
| 5 | `interInsertV` | Moves a customer from one vehicle to another |
| 6 | `inter2OptV` | Tail swap between two vehicles |

### Courier Operators (7–12)

Structurally symmetric to operators 1–6; operates on courier routes.

### Transfer Operators (13–18)

Customer transfer between the vehicle channel and the courier channel. Operators in this category are specific to JVCRP and do not exist in classical VRP solvers. By enabling channel crossing, the search space of the solution is dramatically expanded.

| ID | Name | Description |
|---|---|---|
| 13 | `intraSwapT` | Swaps a customer between a vehicle and a courier |
| 14 | `intraInsertVC` | Moves a customer from a vehicle route into a courier route |
| 15 | `intraInsertCV` | Moves a customer from a courier route into a vehicle route |
| 16–18 | `inter*` | Inter-route versions of the above |

### Locker Operators (19–24)

| ID | Name | Description |
|---|---|---|
| 19 | `openL` | Opens a new locker; assigns a customer to the courier channel |
| 20 | `closeL` | Closes a locker; pulls its customers back to a vehicle |
| 21 | `ruinAndRecreate` | Removes 3–7 customers; reinserts them via regret insertion |
| 22 | `stringRelocation` | Moves 2–3 consecutive customers to another vehicle |
| 23 | `lockerSwap` | Swaps locker assignments between two courier routes |
| 24 | `smartLockerRealloc` | Relocates an under-utilized locker to a more active position |

---

## 7. Hybrid Optimization Engine — SA + VND + ALNS

### Simulated Annealing (SA)

```
T_initial    = 150.0
Cooling rate = 0.9975  (applied every 50 iterations)
T_min        = 0.01
```

Metropolis criterion: worsening solutions are accepted with probability `exp(-Δcost / T)`. This prevents the search from becoming trapped in local minima.

**Reheating:** When no improvement is found for 5,000 iterations, the temperature is reset to `100.0` and aggressive Ruin & Recreate is applied. This mechanism serves as a diversification trigger.

### Variable Neighborhood Descent (VND)

VND tries all 24 operators in sequence and adopts the **first improvement** strategy. It is triggered in two distinct contexts:

- When a new global best solution is found → 50 iterations of intensification.
- Periodically every 2,000 iterations → 30 iterations of intensification.

This dual-trigger scheme allows VND to deeply exploit the promising regions that SA has explored.

### Adaptive Large Neighborhood Search (ALNS)

Operator selection is performed via roulette wheel; weights are updated based on performance:

```
σ₁ = 33.0  → New global best
σ₂ = 9.0   → Better than current solution
σ₃ = 3.0   → Accepted (but worse)

w_new = λ × σ + (1 - λ) × w_old    (λ = 0.1)
```

A minimum weight of `0.1` is enforced so that no operator is ever completely disabled.

At the end of the run, `OperatorStatistics` prints a detailed report: number of attempts, success rate, current weight, and average improvement per operator.

---

## 8. Cost Function

```
Total Cost = Distance Cost + Courier Cost + Penalty Cost

Distance Cost = Σ (vehicle route distances) × w_travel

Courier Cost  = Σ (w_courier_base + route_distance × w_courier_dist)
                for each active courier

Penalty Cost  = Σ max(0, arrival_time - due_date) × penalty_late      (time window)
              + Σ max(0, load - capacity)          × penalty_capacity  (capacity violation)
```

**Locker synchronization:** The vehicle records its arrival time at a locker (`locker_availability_time`). A courier can only pick up from a locker after the vehicle has made its delivery there; this constraint is enforced inside `calculateTotalCost()` via a timeline simulation.

**Why soft penalties instead of hard constraints?** Infeasible solutions must remain explorable during optimization. The value `penalty_capacity = 10000` effectively deters violations through a high penalty weight, but does not completely close off the search space.

---

## 9. Configuration Reference

| Parameter | Default | Description |
|---|---|---|
| `vehicle_capacity` | 397 | Vehicle load capacity |
| `courier_capacity` | 198 | Courier load capacity |
| `max_vehicles` | 2 | Maximum number of vehicles |
| `vehicle_speed` | 1.0 | Vehicle speed unit |
| `courier_speed` | 1.0 | Courier speed unit |
| `w_travel` | 1.0 | Distance cost coefficient |
| `w_courier_base` | 20.0 | Fixed cost per courier activation |
| `w_courier_dist` | 0.5 | Courier distance cost coefficient |
| `penalty_late` | 1.0 | Late delivery penalty coefficient |
| `penalty_capacity` | 10000.0 | Capacity violation penalty coefficient |
| `alpha_penalty` | 100.0 | Oracle dynamic weighting coefficient (α) |
| `oracle_threshold` | 0.50 | Channel assignment threshold |
| `alns_decay` | 0.95 | ALNS weight update decay factor |
| `alns_segment_length` | 100 | ALNS segment length |
| `ruin_min_customers` | 3 | Minimum customers removed in Ruin step |
| `ruin_max_customers` | 7 | Maximum customers removed in Ruin step |
| `time_limit_seconds` | 600 | Total time limit (seconds) |

---

## 10. Build and Run

### Requirements

- C++17 or later
- No dependencies beyond the standard library

### Compilation

```bash
# GCC
g++ -O2 -std=c++17 -o jvcrp main.cpp

# Clang
clang++ -O2 -std=c++17 -o jvcrp main.cpp

# MSVC (Visual Studio Developer Command Prompt)
cl /O2 /std:c++17 main.cpp /Fe:jvcrp.exe
```

### Running

```bash
./jvcrp
```

The program writes to standard output; all iteration progress and phase logs are printed to the console.

---

## 11. Output Format

```
========================================
         FINAL SCHEDULE REPORT
========================================
Total Cost:     <float>
  - Distance (km): <float>
  - Courier Pay:   <float>
  - Penalties:     <float>
Feasibility:     VALID | INVALID

--- Professional Vehicle Routes ---
Vehicle 1 (Load: X): [D] -> 3 -> <L14> -> 7 -> [D] | Dist: Y

--- Crowd-Sourced Courier Routes ---
Courier 1 (Base: L14): <L14> -> 5 -> 9 -> <L14> | Dist: Z | Load: W
========================================
```

Below this block, the ALNS Operator Statistics table is printed: number of attempts per operator, success rate, current weight, and average improvement amount.

---

## 12. Engineering Decisions and Design Rationale

**Why a single file (`main.cpp`)?**  
In a prototype and research context, having all components readable in one place makes it easier to follow the algorithmic flow end-to-end. When moving to a production environment, a proper header + source separation should be applied.

**Why `std::mt19937` with a deterministic seed?**  
Reproducibility is the foundation of research work. The fixed value `seed = 42` produces deterministic results; seeding with `time(nullptr)` increases diversification across multi-start runs. The design accommodates both modes simultaneously.

**Why explicit `deepCopy()` instead of relying on copy semantics?**  
Since `Solution` objects contain `std::vector` and `std::map` members, the default copy constructor already produces a deep copy. However, the `deepCopy()` function explicitly marks intentional copy points in the code — this makes future refactoring for move semantics or an object pool optimization significantly easier.

**Why is `penalty_capacity = 10000` so high?**  
Solutions that violate capacity become so expensive that they are practically never accepted; yet the search is not completely restricted. This "soft constraint as high penalty" approach keeps the search space open during large perturbation steps such as Ruin & Recreate, where transient violations may arise.

**What is the value of the Transfer Operators (13–18)?**  
Standard VRP solvers only model intra-route and inter-route moves within a single fleet. Transfer operators move a customer from the vehicle channel to the courier channel or vice versa. Without these transitions, the optimization remains locked into the channel assignments made during construction and cannot access a large portion of the global optimum.

**Rationale for the multi-start strategy:**  
SA and ALNS are sensitive to the initial solution. Running 5 independent trials increases both the diversity of starting points and the breadth of exploration. The theoretical lower bound analysis printed at the start of `main()` provides a reference for evaluating how close each run has come to the optimum.

---

Copyright (c) 2026 Beyza Eser. All rights reserved.
