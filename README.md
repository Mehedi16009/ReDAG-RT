# ReDAG$`^{\mathrm{\textbf{RT}}}`$: Global Rate-Priority Scheduling for Real-Time Multi-DAG Execution in ROS 2

> **Official Research Artifact**  
> Submitted to *ACM Transactions on Cyber-Physical Systems (TCPS)* — Under Review  
> arXiv Preprint: [arXiv:2603.18238](https://arxiv.org/pdf/2603.18238)

<!-- BADGES — replace placeholders with live shield.io badges -->
![Build Status](https://img.shields.io/badge/build-passing-brightgreen)
![ROS 2 Distribution](https://img.shields.io/badge/ROS%202-Humble%20Hawksbill-blue)
![License](https://img.shields.io/badge/license-Apache%202.0-blue)
![Platform](https://img.shields.io/badge/platform-x86__64%20%7C%20ARM64-lightgrey)
![arXiv](https://img.shields.io/badge/arXiv-2603.18238-red)
![GitHub Repo](https://img.shields.io/badge/GitHub-ReDAGRT--black)

---

## Overview

Modern robotic systems run multiple computation pipelines at once: perception, localization, planning, and control, all inside a shared middleware runtime. In ROS 2, each pipeline forms a directed acyclic graph (DAG) of callbacks, and a single executor dispatches all of them.

The problem is that the default `SingleThreadedExecutor` and `MultiThreadedExecutor` use plain FIFO dispatch. They do not enforce any rate-based priority across independent DAGs. This causes cross-DAG priority inversion, where low-frequency callbacks block high-frequency, control-critical ones simply because they arrived in the queue first. The result is uncontrolled contention, unbounded blocking, and unstable deadlines once multiple DAGs run concurrently. At moderate to high utilization, this pushes deadline miss rates past 70%, which makes default ROS 2 execution unsuitable for safety-critical cyber-physical systems.

This repository contains **ReDAG-RT**, a user-space global scheduling framework built on top of unmodified ROS 2. It introduces a Rate-Priority (RP) global ready queue that orders every callback in the system by activation rate, enforces per-DAG concurrency limits through configurable `max_active` parameters, and removes cross-graph priority inversion. None of this requires changes to the ROS 2 API, the executor interface, or the Linux kernel scheduler.

The framework also formalizes a multi-DAG task model $\mathcal{G} = \{G_k = (V_k, E_k)\}$, derives worst-case interference bounds $I_i = \sum_{\tau_j \in hp(i)} C_j$, and builds response-time recurrences and schedulability conditions on classical Rate-Monotonic theory.

All experimental configurations, raw runtime traces, analysis scripts, and figures from the manuscript are included here for full reproducibility. The framework is validated under mixed-period, mixed-criticality synthetic workloads in a containerized ROS 2 Humble environment, and every result can be reproduced directly through the automation pipeline in `/scripts`.



---

## Architecture

<img width="800" height="600" alt="Screenshot 2026-06-24 at 4 57 27 PM" src="https://github.com/user-attachments/assets/0ddc41cc-696f-4818-8e0f-038d12e72f6d" />

Figure 1. ReDAG-RT layered user-space scheduling architecture for periodic precedence-constrained DAG workloads. Multiple robotic pipelines modeled as $G_k = (V_k, E_k)$ release callback tasks periodically upon period expiration; a dependency-aware release stage validates both timing and intra-DAG precedence constraints before admitting ready tasks into a unified global Rate-Monotonic priority queue ordered by $\tau^* = \arg\min_{i \in \mathcal{R}} T_i$. The ReDAG-RT global executor performs fixed-priority preemptive arbitration across all concurrent DAGs, triggering preemption when a higher-priority task enters the ready set, and dispatches selected tasks directly to the unmodified Linux execution layer. An integrated Timing Monitor computes response time $R_i = f_i - r_i$, lateness $L_i = f_i - d_i$, and miss rate
$\mathrm{MR}_i$ online, closing the event-driven re-scheduling loop.


---



## Key Features

- **Global Rate-Priority (RP) Dispatching**  
  All callbacks across all concurrent DAG pipelines are
  admitted into a single unified priority queue ordered
  strictly by Rate-Monotonic assignment
  ($T_i < T_j \Rightarrow P_i > P_j$), eliminating
  structural cross-DAG priority inversion.

- **Zero Kernel and API Modifications**  
  ReDAG-RT operates entirely within the ROS 2 user-space
  execution layer. The Linux scheduler, ROS 2 core API,
  and executor interface remain fully unmodified, ensuring
  compatibility with all existing ROS 2 applications.

- **Dependency-Aware Task Admission**  
  A two-condition admission gate validates period
  expiration and intra-DAG precedence constraint
  resolution before inserting tasks into the global
  queue, preserving structural DAG semantics.

- **Asymmetric Per-DAG Concurrency Bounds**  
  Configurable `max_active` parameters per DAG allow
  asymmetric concurrency allocation, reducing mutual
  interference by limiting one pipeline's parallel
  footprint and suppressing cross-DAG contention by
  up to 40.8% relative to symmetric configurations.

- **Formal Schedulability Analysis Integration**  
  Response-time recurrences
  $R_i^{(k+1)} = C_i + \sum_{\tau_j \in hp(i)}
  \lceil R_i^{(k)} / T_j \rceil C_j$
  and the RM utilization bound
  $U \leq N(2^{1/N} - 1)$ are implemented as
  analytical verification hooks, enabling formal
  feasibility testing per task.

- **Comprehensive Runtime Telemetry**  
  Per-job finish times, response times, lateness values,
  maximum lateness $L_i^{\max}$, and miss rate
  $\mathrm{MR}_i$ are logged online and preserved for
  offline analysis and visualization.

- **Fully Open Source and Reproducible**  
  Complete implementation, workload generator,
  experimental sweep scripts, raw logs, and analysis
  pipelines are publicly available under Apache 2.0.

---

## Running Prerequisites

The following host-level software dependencies must be
satisfied prior to environment initialization:

| Requirement | Minimum Version | Notes |
|---|---|---|
| Operating System | Ubuntu 22.04 LTS (Jammy) | Native or WSL2 |
| Docker Engine | 24.x or Docker Desktop 4.x | Required for containerized isolation |
| Git | 2.34+ | For repository cloning and branch management |
| Python | 3.10+ | Host-side analysis scripts |
| curl / wget | Any current stable | For dependency fetching |

---

## Experimental Environment

The experimental framework is validated and confirmed
reproducible on the following hardware architectures
via containerized isolation:

| Architecture | Target Platform | Container Base |
|---|---|---|
| `x86_64` | Multi-core server workstations, CI runners | `osrf/ros:humble-desktop` |
| `ARM64` | Apple Silicon (M1/M2/M3), embedded robotics targets | `osrf/ros:humble-ros-base` |

All experiments are executed within Docker containers to
ensure dependency isolation, eliminate host-side
interference, and guarantee bit-identical reproducibility
across platforms. No host-level ROS 2 installation is
required.

---

## Core Library Versions

| Component | Version / Distribution | Notes |
|---|---|---|
| ROS 2 Distribution | Humble Hawksbill (LTS) | End of support: May 2027 |
| Build System | `colcon` 0.3.x | With `colcon-common-extensions` |
| Compiler | GCC/G++ 11.x | C++17 standard (`-std=c++17`) |
| C++ Client Library | `rclcpp` (Humble) | Core executor extension target |
| Python Client Library | `rclpy` (Humble) | Workload generator and analysis |
| CMake | 3.22+ | Required by `ament_cmake` |
| Python | 3.10.x | Analysis and reporting pipeline |

---

## Deep Learning Backend

**None.**

Omitting external deep learning runtimes is an
intentional architectural design choice. Incorporating
tensor execution frameworks (e.g., TensorRT, ONNX
Runtime, PyTorch) would introduce non-deterministic
execution paths, variable-latency inference calls, and
GPU memory contention — all of which violate the
analytical assumptions required by classical
Rate-Monotonic schedulability proofs and invalidate
the formal interference bounds derived in the
manuscript. ReDAG-RT is designed to operate within
a provably analyzable execution envelope; all
workloads consist of periodic, bounded-WCET callback
tasks with statically known timing parameters.

---

## GPU Configuration

**None / CPU Isolated.**

All execution runs strictly on prioritized, isolated
CPU cores. GPU acceleration is deliberately excluded
to maintain compliance with classical Rate-Monotonic
schedulability analysis and to mitigate un-analyzable
hardware acceleration cross-talk that would corrupt
timing measurements. Future extensions toward
heterogeneous GPU-accelerated perception pipelines
are identified as an open research direction in the
manuscript but are outside the scope of this artifact.

---

## Repository Structure

```text
ReDAG-RT/
│
├── analysis/
│   ├── phase16_generate_report.py   # Final metrics aggregation
│   │                                  and high-resolution figure
│   │                                  generation from Phase 16 logs
│   └── utils/                       # Shared statistical utilities
│                                      (lateness, CDF, heatmap helpers)
│
├── baseline_smoke/
│   ├── single_threaded_baseline/    # Default SingleThreadedExecutor
│   │                                  runs at harmonic periods
│   └── multi_threaded_baseline/     # Default MultiThreadedExecutor
│                                      runs at non-harmonic periods
│
├── figures/
│   ├── fig01_cross_dag_problem.png  # Introduction: priority inversion
│   │                                  problem and ReDAG-RT solution
│   ├── fig02_redagrt_layered_       # Methodology: layered user-space
│   │   architecture.png               scheduling architecture diagram
│   ├── fig03_thread_scaling.pdf     # Thread scalability miss rate curve
│   ├── fig04_deadline_sensitivity.  # Deadline scaling factor sensitivity
│   │   pdf                            plot
│   ├── fig05_concurrency_heatmap.   # Per-DAG concurrency pair heatmap
│   │   pdf
│   └── fig06_response_cdf.pdf       # CDF of callback response times
│
├── logs/
│   ├── phase12_stress_logs/         # Raw traces: heavy concurrent
│   │                                  workload injection (Phase 12)
│   ├── phase13_interference_logs/   # Raw traces: cross-DAG interference
│   │                                  characterization (Phase 13)
│   ├── phase14_profile_logs/        # Execution profiling under standard
│   │                                  ReDAG-RT schedules (Phase 14)
│   ├── phase15_sweep_logs/          # Sensitivity sweep raw outputs
│   │                                  (Phase 15)
│   └── phase16_final_logs/          # Optimal configuration final traces
│                                      used for manuscript tables/figures
│
├── scripts/
│   ├── phase15_sweep.sh             # Parametric sweep: threads
│   │                                  {4,6,8,10} × deadline scale
│   │                                  {0.8,0.9,1.1,1.2}
│   ├── phase16_run_best.sh          # Best configuration validation run
│   │                                  (8 threads, δ=1.1)
│   ├── run_baseline.sh              # Baseline executor comparison runner
│   └── build_workspace.sh           # Full workspace build automation
│
└── src/
    ├── rp_executor/                 # Core ReDAG-RT global executor
    │   ├── include/                   implementation (C++17)
    │   │   └── rp_executor.hpp
    │   └── src/
    │       ├── rp_executor.cpp        # RP global queue + dispatcher
    │       ├── dependency_resolver.   # Intra-DAG precedence validator
    │       │   cpp
    │       └── timing_monitor.cpp     # R_i, L_i, MR_i online logger
    │
    ├── multi_dag_demo/              # Synthetic multi-DAG workload
    │   ├── dag_generator.py           generator and launcher
    │   └── workload_config.yaml       # Period, WCET, criticality config
    │
    └── default_executor_baseline/   # Unmodified ROS 2 executor
        ├── single_threaded_node.py    baseline implementations for
        └── multi_threaded_node.py     direct comparison
```

---

## How to Run the Project on GitHub

No local installation is required to inspect the
research artifact via the GitHub web interface.
The following resources are directly accessible:

**Browsing the implementation:**
Navigate to `src/rp_executor/` to inspect the core
`rp_executor.cpp` dispatcher, `dependency_resolver.cpp`,
and `timing_monitor.cpp` directly in the GitHub
code viewer with full syntax highlighting.

**Reviewing raw experimental logs:**
All runtime traces used to generate the manuscript
figures and tables are preserved in `/logs/` as
plain-text files and can be inspected directly
from the repository browser without any local
dependencies.

**Inspecting analysis scripts:**
The complete reporting pipeline in
`analysis/phase16_generate_report.py` is readable
in full from the GitHub interface, showing every
metric computation and figure generation step.

**Examining sweep configurations:**
The parametric sweep scripts in `/scripts/` document
the exact thread counts, deadline scaling factors,
and concurrency pair configurations used in the
experimental evaluation.

> **Note:** If GitHub Actions CI is attached to
> this repository, navigate to the **Actions** tab
> to view automated build and smoke-test status
> across the supported platforms.

---

## How to Run the Project (Local PC)

### Step 1 — Clone the Repository

```bash
git clone https://github.com/Mehedi16009/ReDAG-RT.git

cd ReDAG-RT
```

### Step 2 — Launch the ROS 2 Humble Container

```bash
docker run -it --rm \
  --name redag_rt_env \
  --volume $(pwd):/workspace \
  --workdir /workspace \
  osrf/ros:humble-desktop \
  bash
```

> For ARM64 targets (Apple Silicon):
> ```bash
> docker run -it --rm \
>   --platform linux/arm64 \
>   --name redag_rt_env_arm \
>   --volume $(pwd):/workspace \
>   --workdir /workspace \
>   osrf/ros:humble-ros-base \
>   bash
> ```

### Step 3 — Source ROS 2 and Check Dependencies

```bash
source /opt/ros/humble/setup.bash

rosdep update

rosdep install \
  --from-paths src \
  --ignore-src \
  --rosdistro humble \
  -y
```

### Step 4 — Build the Workspace

```bash
colcon build \
  --symlink-install \
  --cmake-args -DCMAKE_BUILD_TYPE=Release

source install/setup.bash
```

### Step 5 — Run the Baseline Evaluation

```bash
bash scripts/run_baseline.sh
```

### Step 6 — Run the ReDAG-RT Framework

```bash
ros2 run rp_executor rp_executor_node \
  --ros-args \
  -p max_active_dag1:=2 \
  -p max_active_dag2:=5 \
  -p num_threads:=8
```

### Step 7 — Run the Parametric Sweep (Phase 15)

```bash
bash scripts/phase15_sweep.sh
```

### Step 8 — Run Best Configuration (Phase 16)

```bash
bash scripts/phase16_run_best.sh
```

### Step 9 — Generate Final Report and Figures

```bash
python3 analysis/phase16_generate_report.py \
  --log-dir logs/phase16_final_logs \
  --output-dir figures/
```

---

## Required Python Packages

Install the lightweight analytical dependencies
used by the evaluation pipeline:

```bash
pip install numpy pandas matplotlib scipy
```

| Package | Version | Purpose |
|---|---|---|
| `numpy` | ≥ 1.24 | Numerical trace processing and statistical computation |
| `pandas` | ≥ 2.0 | Tabular log ingestion and metric aggregation |
| `matplotlib` | ≥ 3.7 | CDF plots, bar charts, scalability curves |
| `scipy` | ≥ 1.11 | Statistical analysis and distribution fitting |
| `seaborn` | ≥ 0.12 | Concurrency sensitivity heatmap generation |

---

## Pipeline Stages

The validation pipeline is organized into
chronological phases corresponding to the
experimental methodology described in the
manuscript:

### Baseline Evaluation

**Source:** `src/default_executor_baseline/`

Run unmodified `SingleThreadedExecutor` under
harmonic periods and `MultiThreadedExecutor`
under non-harmonic periods to establish reference
miss rate, lateness, and response time baselines
for cross-system comparison.

```bash
bash scripts/run_baseline.sh
```

---

### Phase 12 — Stress Testing

**Logs:** `logs/phase12_stress_logs/`

Heavy concurrent synthetic workload injection
across both DAG pipelines under maximum
utilization and minimum deadline tightness
($\delta = 0.8$). Designed to expose the
worst-case interference behavior of both the
default executors and ReDAG-RT.

---

### Phase 13 — Interference Characterization

**Logs:** `logs/phase13_interference_logs/`

Systematic measurement of cross-DAG blocking
and interference chain growth under varying
`max_active` concurrency pair configurations.
Generates the raw data for the concurrency
sensitivity heatmap (Figure 5 in the manuscript).

---

### Phase 14 — Execution Profiling

**Logs:** `logs/phase14_profile_logs/`

Execution profiling iterations under standard
ReDAG-RT schedules across the full thread count
range $\{4, 6, 8, 10\}$. Records per-job
$R_i$, $L_i$, and $\mathrm{MR}_i$ for the
thread scalability analysis (Figure 3).

---

### Phase 15 — Parametric Sensitivity Sweep

**Script:** `scripts/phase15_sweep.sh`  
**Logs:** `logs/phase15_sweep_logs/`

```bash
bash scripts/phase15_sweep.sh
```

Sweeps the full parameter space:
- Thread counts: $\{4, 6, 8, 10\}$
- Deadline scaling factors: $\{0.8, 0.9, 1.1, 1.2\}$
- Concurrency pairs: 16 distinct
  $(\texttt{max\_active}_1, \texttt{max\_active}_2)$
  configurations

Produces all raw data for Figures 3, 4, and 5
and Tables II, III, and IV.

---

### Phase 16 — Final Validation and Reporting

**Script:** `scripts/phase16_run_best.sh`  
**Analysis:** `analysis/phase16_generate_report.py`  
**Logs:** `logs/phase16_final_logs/`

```bash
bash scripts/phase16_run_best.sh

python3 analysis/phase16_generate_report.py \
  --log-dir logs/phase16_final_logs \
  --output-dir figures/
```

Executes the optimal validated configuration
(8 threads, $\delta = 1.1$, asymmetric bounds
$(2, 5)$) and generates all final manuscript
metrics, tables, and high-resolution figures.

---

## Experimental Results

The following empirical improvements are reported
in the manuscript over default ROS 2 executors
under controlled mixed-period, mixed-criticality
workloads in a ROS 2 Humble environment:

| Metric | Improvement | Comparison Baseline |
|---|---|---|
| Combined deadline miss rate | **-29.7%** | 4 → 8 threads |
| 99th-percentile response time | **-42.9%** | 49 ms → 28 ms |
| Miss rate vs MultiThreaded | **-13.7%** | U = 0.8 comparable utilization |
| Cross-DAG interference | **-40.8%** | Asymmetric vs symmetric bounds |
| Best vs worst configuration | **-51.2%** | Miss rate 0.975 → 0.476 |
| Median response time | **-33.3%** | 21 ms → 14 ms |

All 64 experimental configurations report
`all_enforced = 1` from the DAG concurrency
validator, confirming that per-DAG concurrency
bounds were never violated during any run.

---

## Reproducibility

All raw runtime traces used to produce the
manuscript figures, tables, and quantitative
claims are preserved verbatim in the `/logs/`
directory under phase-labeled subdirectories
corresponding to each validation stage.

To dynamically reconstruct equivalent raw
traces and regenerate all figures from scratch:

```bash
# Full end-to-end pipeline reconstruction
bash scripts/run_baseline.sh      # Baseline traces
bash scripts/phase15_sweep.sh     # Full sweep traces
bash scripts/phase16_run_best.sh  # Best-config traces

# Regenerate all figures and tables
python3 analysis/phase16_generate_report.py \
  --log-dir logs/phase16_final_logs \
  --output-dir figures/
```

> The automated pipeline is deterministic under
> identical hardware and software configurations.
> Minor numeric variance (< 1%) may arise from
> OS scheduling jitter in non-isolated host
> environments. For maximal reproducibility,
> execute within the provided Docker container
> on a quiescent, lightly loaded host system.

---

## Citation

If this work informs your research, please cite
using the following BibTeX entry:

```bibtex
@article{hasan2026redagrt,
  author    = {Hasan, Md Mehedi and
               Mostafiz, Rafid and
               Paul, Biplob Kumar and
               Hossain, Md Abir and
               Rahman, Ziaur and Ridoy, Mohammad Mehedy Hasan},
  title     = {{ReDAG-RT}: Global Rate-Priority
               Scheduling for Real-Time Multi-{DAG}
               Execution in {ROS}~2},
  journal   = {arXiv preprint arXiv:2603.18238},
  year      = {2026},
  note      = {Submitted to \textit{ACM Transactions
               on Cyber-Physical Systems (TCPS)} ---
               Under Review},
  url       = {https://arxiv.org/pdf/2603.18238}
}
```

---

## License

This project is licensed under the
**Apache License, Version 2.0**.


## Ethical Statement

All performance measurements, runtime execution
traces, deadline miss statistics, response time
distributions, and benchmark behaviors reported
in this manuscript and preserved in this
repository were collected objectively under
controlled experimental conditions. No data
points were selectively omitted, post-hoc
adjusted, cherry-picked across configurations,
or otherwise manipulated to artificially
strengthen reported improvements. The baseline
executor configurations were evaluated under
conditions most favorable to their respective
execution models (harmonic periods for
`SingleThreadedExecutor`; non-harmonic periods
for `MultiThreadedExecutor`) to ensure fair and
conservative comparison. All 64 experimental
configurations were executed and logged in full;
no configurations were excluded from reporting.
The complete raw logs are publicly archived in
this repository to enable independent
verification by any interested researcher.

---

## Contact

For questions regarding the research, the
experimental artifact, or potential research
collaboration, please contact:

**Md Mehedi Hasan**  
Department of Information and Communication
Technology  
Mawlana Bhashani Science and Technology University
(MBSTU), Tangail, Bangladesh

- **Email:** mehedi.hasan.ict@mbstu.ac.bd  
- **arXiv:** [arXiv:2603.18238](https://arxiv.org/pdf/2603.18238)  
- **GitHub:** [Mehedi16009](https://github.com/Mehedi16009)  
- **Repository:** [ReDAG-RT](https://github.com/Mehedi16009/ReDAG-RT)

---

<div align="center">

*ReDAG-RT — Deterministic Multi-DAG Scheduling,*  
*Entirely in ROS 2 User Space.*

**arXiv:2603.18238 · Apache 2.0 · ROS 2 Humble**

</div>

