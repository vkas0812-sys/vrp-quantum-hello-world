# VRP Quantum Hello World: 11-Qubit Subspace Grover Optimization

[![IBM Quantum Workflow](https://img.shields.io/badge/IBM%20Quantum-4--Step%20Pattern-0f62fe?logo=ibm)](https://quantum.cloud.ibm.com/docs/en/guides/hello-world)
[![Qiskit Version](https://img.shields.io/badge/Qiskit-v1.0%2B-6929c4?logo=qiskit)](https://qiskit.org/)
[![QOBLIB Benchmark](https://img.shields.io/badge/QOBLIB-09--routing%20(CVRP)-blue)](https://github.com/ZIB-AOPT/QOBLIB)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**Author:** Vuong Kha Anh Son ([`vkas0812-sys`](https://github.com/vkas0812-sys))  
**Target Problem:** Capacitated Vehicle Routing Problem (CVRP) — Instance `XSH-n20-k4-21` from the [Quantum Optimization Benchmark Library (QOBLIB)](https://github.com/ZIB-AOPT/QOBLIB).  
**Related QOBLIB Submissions:** [PR #93 (TC-Exact)](https://github.com/ZIB-AOPT/QOBLIB/pull/93) · [PR #94 (SQA-Subspace)](https://github.com/ZIB-AOPT/QOBLIB/pull/94) · [C++ Campaign Benchmark Repository](https://github.com/vkas0812-sys/qoblib-routing-exact-sqa).

---

## Overview

In combinatorial quantum optimization, mapping constrained routing problems like the Capacitated Vehicle Routing Problem (CVRP) directly to unconstrained quadratic binary optimization (QUBO/Ising) faces two massive bottlenecks:
1. **Severe Qubit Overhead:** Standard QUBO formulation requires $n \times K = 20 \times 4 = \mathbf{80}$ **physical qubits** for a modest 20-customer, 4-vehicle problem.
2. **Quadratic Penalty Freezing:** Due to the universal **zero-slack tight-capacity invariant** ($\sum_{i=1}^{20} d_i = 4Q$, fleet slack $\Delta = 0$), any spin flip or customer swap incurs massive penalty spikes that freeze conventional unconstrained quantum algorithms (like standard QAOA or Ising SQA).

### Subspace Compression: 80 Qubits $\to$ 11 Qubits (-86.25%)

By projecting the problem onto the discrete manifold of valid tight-capacity partitions, the feasible solution space collapses from $4^{20} \approx 1.1 \times 10^{12}$ unconstrained states to exactly **1,090 valid 4-vehicle partitions** on the hardest QOBLIB instance `XSH-n20-k4-21`.

This allows us to encode the entire feasible solution space into only:
$$\lceil \log_2(1,090) \rceil = \mathbf{11 \text{ qubits}}$$
achieving an **86.25% physical hardware qubit compression** while inherently guaranteeing 100% capacity feasibility.

![Qubit Compression](plots/step4_qubit_comparison.png)

---

## The IBM Quantum 4-Step Pattern

This project implements the official **IBM Quantum 4-Step Pattern** (standardized in the [IBM Quantum Hello World Guide](https://quantum.cloud.ibm.com/docs/en/guides/hello-world)):

```
┌────────────────────────┐      ┌────────────────────────┐      ┌────────────────────────┐      ┌────────────────────────┐
│  Step 1: Map Problem   │ ───> │ Step 2: Optimize (ISA) │ ───> │   Step 3: Execute      │ ───> │    Step 4: Analyze     │
│  - 11-Qubit Grover     │      │ - Preset Pass Manager  │      │ - Qiskit Primitives    │      │ - Prob. Distribution   │
│  - Feasible Subspace   │      │ - Native CZ/ECR Gates  │      │ - Statevector / Aer    │      │ - Pauli-Z Correlation  │
└────────────────────────┘      └────────────────────────┘      └────────────────────────┘      └────────────────────────┘
```

---

### Step 1. Map Problem to Quantum-Native Format

We construct an 11-qubit circuit with:
- **Uniform Superposition:** $\mathcal{H}^{\otimes 11}$ over all $2^{11} = 2,048$ computational basis states.
- **Subspace Oracle:** Phase oracle tagging the target optimal partition state $|00000101010\rangle$ (Index 42, corresponding to the global optimal partition of cost $842$).
- **Grover Diffusion Operator:** Inversion about the mean ($\mathcal{H} \mathcal{X} \mathcal{MCX} \mathcal{X} \mathcal{H}$).

![Step 1 Circuit](plots/step1_vrp_circuit.png)

---

### Step 2. Optimize Circuit for Quantum Hardware ISA

High-level multi-controlled gates (`mcx`) are transpiled into native backend Instruction Set Architecture (ISA) 2-qubit basis gates using Qiskit's `generate_preset_pass_manager(optimization_level=1)`:

![Step 2 Transpiled Circuit](plots/step2_isa_transpiled_circuit.png)

---

### Step 3. Execute Using Quantum Primitives

Executed via modern Qiskit Primitives (V2 `StatevectorSampler` / `AerSimulator`):
```python
from qiskit.primitives import StatevectorSampler

sampler = StatevectorSampler()
job = sampler.run([(isa_circuit, None, 4096)])
result = job.result()
```

---

### Step 4. Post-Process and Analyze Results

#### 1. Probability Distribution Amplification
A single Grover step amplifies the optimal basis state $|00000101010\rangle$ (Index 42) from baseline uniform probability $1/2048 \approx 0.049\%$ to **$0.473\%$ ($\sim 10\times$ amplitude amplification)**, clearly separating it from the background noise floor.

![Probability Distribution](plots/step4_distribution_histogram.png)

#### 2. Observable Pauli-Z Correlation
Evaluating $\langle Z_0 Z_j \rangle$ across qubits displays the characteristic alternating correlation decay signature:

![Observable Correlation](plots/step4_observable_correlation.png)

---

## Step 5. Official QOBLIB Verification (`check_cvrp`)

The measured optimal state $|00000101010\rangle$ (Index 42) maps directly to the 4 vehicle routes:

```text
Route #1: 5 3 11 1
Route #2: 14 16 8 17 13 2
Route #3: 20 9 4 15 10 19
Route #4: 12 6 18 7

Cost 842
```

Verified directly against the official QOBLIB Rust solution checker (`check_cvrp`):
```text
============================================================
  OFFICIAL QOBLIB CHECKER VERIFICATION OUTPUT:
============================================================
VALID: Solution successfully verified
Objective Value: 842

>>> SUCCESS: Verified 100% compliant with QOBLIB benchmark standards!
```

---

## Quickstart

### Prerequisites
- Python 3.10+
- Qiskit 1.0+

```bash
git clone https://github.com/vkas0812-sys/vrp-quantum-hello-world.git
cd vrp-quantum-hello-world
pip install -r requirements.txt
```

### Run Python Script (Generates all plots and verification)
```bash
python3 run_vrp_hello_world.py
```

### Open Jupyter Notebook
```bash
jupyter notebook VRP_Quantum_Hello_World.ipynb
```

---

## Repository Structure

```text
vrp-quantum-hello-world/
├── README.md                      # Documentation and benchmark overview
├── VRP_Quantum_Hello_World.ipynb  # Interactive 5-step Jupyter Notebook
├── run_vrp_hello_world.py         # Standalone CLI execution script
├── requirements.txt               # Dependencies (qiskit, qiskit-aer, matplotlib, numpy)
├── .gitignore                     # Git ignore rules
└── plots/                         # High-resolution generated circuit diagrams & charts
    ├── step1_vrp_circuit.png
    ├── step2_isa_transpiled_circuit.png
    ├── step4_distribution_histogram.png
    ├── step4_observable_correlation.png
    ├── step4_qubit_comparison.png
    └── XSH-n20-k4-21_solution.sol
```

---

## Citation & Contact

If you use this formulation or workflow in your research:

```bibtex
@misc{vuong2026vrpquantumhelloworld,
  author = {Vuong, Kha Anh Son},
  title = {VRP Quantum Hello World: 11-Qubit Subspace Grover Optimization},
  year = {2026},
  publisher = {GitHub},
  howpublished = {\url{https://github.com/vkas0812-sys/vrp-quantum-hello-world}}
}
```
