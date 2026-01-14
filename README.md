# Cluster-Capacity-Optimization
Generic Cluster Optimization Algorithm using MILP
Very generic and early draft of what could be, this is Work in progress
I'm still learning 
Author: Pablo Rivera 
A lightweight prototype that optimizes pilot-cluster capacity during PIVT. It
uses daily node/vCPU usage signals plus test-team demand to compute allocations
that maximize utilization, reduce conflicts, and minimize disruptive moves.

---

## Problem
Testing teams request nodes (or vCPU) for weeks. Daily usage often drifts from
plan, creating idle pockets in some clusters while others are quota-bound.
Manual rebalancing is slow and inconsistent. CCO ingests telemetry and demand,
then proposes optimal allocations across clusters.

## Key Inputs
- **Telemetry (daily):** node/cluster occupancy and basic health signals
- **Testing demand:** required nodes/vCPU, duration, earliest start, deadline
- **Constraints:** blackouts, destructive windows, zone/rack placement, policy/LKG versions
- **Costs & priorities:** move (evacuation) cost, team priority/fairness weights

## Outputs
- **Daily optimized plan** per cluster (nodes/vCPU per test) with conflicts resolved
- **Evacuation drafts** (pre-filled intent + node lists) when churn is justified
- **Targeted CT actions** when headroom is insufficient

## UI Views
- **Real vs Projected vs Optimized** utilization by cluster (bar chart)
- **Allocation table** per test × cluster with capacity lines
- **Recommendations panel** with move/CT suggestions

---

## Quick Start (Minimal MILP Example with PuLP)
This toy model assigns nodes to tests across two clusters to maximize usage
without exceeding capacities. Replace toy data with real telemetry and demands.

```python
import pulp

# Clusters and capacities (nodes)
clusters = {"clusterA": 10, "clusterB": 8}

# Tests and required nodes
tests = {"ping": 6, "network": 4, "stress": 5}

# Decision variables: x[(test, cluster)] = nodes assigned (integer)
x = pulp.LpVariable.dicts("x", [(t, c) for t in tests for c in clusters],
                          lowBound=0, cat=pulp.LpInteger)

# Model
model = pulp.LpProblem("CCO_Minimal", pulp.LpMaximize)

# Objective: maximize total assigned nodes
model += pulp.lpSum(x[(t, c)] for t in tests for c in clusters)

# Cluster capacity constraints
for c, cap in clusters.items():
    model += pulp.lpSum(x[(t, c)] for t in tests) <= cap

# Each test must receive exactly its demand
for t, demand in tests.items():
    model += pulp.lpSum(x[(t, c)] for c in clusters) == demand

# Solve
model.solve(pulp.PULP_CBC_CMD(msg=False))

# Print solution
print("Status:", pulp.LpStatus[model.status])
for (t, c), var in x.items():
    val = int(var.value())
    if val > 0:
        print(f"{t:8s} -> {c:8s}: {val} nodes")
print("Total allocated:", int(pulp.value(model.objective)))
```

---

## Simple Visualization (Matplotlib)
Sketch a side-by-side bar chart for Real vs Projected vs Optimized.

```python
import matplotlib.pyplot as plt

clusters = ["clusterA", "clusterB"]
real = [8, 4]        # from telemetry
projected = [10, 7]  # planned demand
optimized = [10, 5]  # solver result
capacity = [10, 8]

x = range(len(clusters))
width = 0.25

plt.figure(figsize=(8,4))
plt.bar([i - width for i in x], real, width, label="Real", color="#4C78A8")
plt.bar(x, projected, width, label="Projected", color="#F58518")
plt.bar([i + width for i in x], optimized, width, label="Optimized", color="#54A24B")

for i, cap in enumerate(capacity):
    plt.hlines(cap, i-0.45, i+0.45, colors="gray", linestyles="dashed")

plt.title("Cluster Usage: Real vs Projected vs Optimized")
plt.xticks(list(x), clusters)
plt.ylabel("Nodes")
plt.legend()
plt.tight_layout()
plt.show()
```

---

## Modeling Options
- **MILP (Mixed Integer Linear Programming):** linear constraints & objectives,
  global optimum; add time-indexed variables for multi-day planning and movement costs.
- **CP-SAT (OR-Tools):** powerful for discrete scheduling with complex logical constraints,
  time windows, and soft penalties.

## Roadmap (optional)
- **MVP:** single-day allocation, movement cost, basic fairness caps
- **Phase 2:** multi-day horizon, blackouts/destructive windows, rolling updates
- **Phase 3:** predictive demand, quota pipeline integration, automated evacuation drafts

## Repository Layout
```
README.md
cco_milp_minimal.py
viz_example.py
sample_data/
  telemetry.csv
  demand.csv
models/
  advanced_milp.ipynb
  cpsat_formulation.ipynb
```

## Usage
1. Replace toy data with daily telemetry and team demands.
2. Add constraints (LKG/recipe versions, zone/rack placement, blackouts).
3. Introduce movement costs and deadlines; balance utilization vs churn.
4. Wire outputs to Evacuate Clusters and CT workflows.

## License
Internal use only (or MIT if open-sourcing). Ensure no confidential data is committed.

## Acknowledgments
This is Work in progress, I haven't bother anybody yet :-)   

