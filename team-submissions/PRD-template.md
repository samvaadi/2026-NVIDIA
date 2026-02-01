# Product Requirements Document (PRD)

**Project Name:** LABS_HUNTER
**Team Name:** QC-IITI
**GitHub Repository:** https://github.com/qc-iiti

---

## 1. Team Roles & Responsibilities [You can DM the judges this information instead of including it in the repository]

| Role | Name | GitHub Handle | Discord Handle
| :--- | :--- | :--- | :--- |
| **Project Lead** (Architect) | Samvaadi Dadhi | samvaadi | .__samvaa__. |
| **GPU Acceleration PIC** (Builder) | Abhiroop Gohar | stark-069 | stark_03 |
| **Quality Assurance PIC** (Verifier) | Aarush Bindod | metamyteee | not_aarush |
| **Technical Marketing PIC** (Storyteller) | Hemal | Hemal2510 | hemal_2510_06608|

---

## 2. The Architecture
**Owner:** Samvaadi Dadhi

### Choice of Quantum Algorithm
* **Algorithm:**
  **Quantum-Enhanced Memetic Tabu Search (QE-MTS)** using **QAOA Ansatz**.
    * We use a shallow-depth QAOA ($p=1$) circuit to generate an initial population of "biased" bitstrings. These quantum seeds are then fed into a highly optimized, GPU-accelerated Memetic Tabu Search (MTS) for fine-grained local optimization.

* **Motivation:**
  Our architecture is designed to defeat the specific "Grand Summit" topology of the LABS problem:
    * **The Classical Catastrophe:** For $N \ge 60$, the solution space ($2^{60}$) is too vast for random guessing. Classical solvers get stuck in deep local minima (suboptimal traps) because random initialization rarely lands in the correct basin of attraction.
    * **The Quantum "Radar":** We leverage QAOA as a global search heuristic. Even with noise, the quantum circuit creates a probability distribution concentrated around low-energy states, effectively "parachuting" our classical solver onto the slopes of the highest peaks rather than the flat plains.
    * Once the quantum layer identifies the promising region, our classical MTS engine—accelerated by custom CUDA kernels—performs the rapid, intensive climbing required to pinpoint the exact global maximum.

### Literature Review
* **Reference:** *Application of Quantum Approximation Optimization Algorithm (QAOA) on Low Autocorrelation Binary Sequences (LABS)* (IEEE/ArXiv).
* **Relevance:** This pivotal study demonstrates that the **Merit Factor (MF)** landscape is amenable to quantum biasing. It proves that a hybrid QAOA+Classical approach yields a statistically significant improvement in solution quality (higher MF) compared to pure classical methods for the same computational budget, validating our "Quantum Seed, Classical Climb" strategy.
   
---

## 3. The Acceleration Strategy
**Owner:** Abhiroop Gohar

**Quantum Acceleration (CUDA-Q)**

**Strategy:**
To overcome the exponential scaling of Hilbert space simulation, we will transition from CPU-based state vector simulation to GPU-accelerated backends provided by CUDA-Q.
* **Single-Node Optimization:** For problem sizes $N < 30$, we will utilize the `nvidia` target. This backend leverages **cuQuantum** (specifically `custatevec`) to accelerate the gate application and linear algebra of the Trotterized circuit, reducing the time-per-shot from seconds to milliseconds.
* **Multi-GPU Scaling:** For problem sizes $N > 30$, the state vector exceeds the memory of a single GPU. We will switch to the `nvidia-mgpu` backend to distribute the memory load of the quantum state across multiple GPUs (using NVLink where available). This allows us to simulate the deep circuits required for the Counteradiabatic protocol without running out of memory (OOM).

**Classical Acceleration (MTS)**

**Strategy:**
The bottleneck of Memetic Tabu Search is the "Neighborhood Evaluation" phase, where the algorithm checks hundreds of potential bit-flips to find the best move.
* **Vectorization with CuPy:** Instead of iterating through neighbors one by one in a Python loop (which is computationally expensive), we will implement a "Batch Evaluator" using **CuPy**.
* **Implementation Detail:** We will represent the entire neighborhood of potential solutions as a single GPU matrix of size $(N_{neighbors} \times N_{bits})$. By broadcasting the autocorrelation calculation across this matrix, we can evaluate the energy of thousands of candidate sequences simultaneously in a single GPU kernel call, resulting in a theoretical speedup of $100\times - 1000\times$ compared to the sequential CPU approach.

**Hardware Targets**

* **Dev Environment:** Qbraid (CPU) for initial logic/unit testing and Brev (NVIDIA L4) for prototyping the CuPy integration and `nvidia` backend.
* **Production Environment:** NVIDIA A100 (80GB) to support the VRAM requirements of large state vectors ($N=40+$) and high-bandwidth memory access for classical batch evaluations.

---

## 4. The Verification Plan
**Owner:** Aarush Bindod

**Unit Testing Strategy**

* **Framework:** `unittest`
* **AI Hallucination Guardrails:** To ensure AI-generated quantum kernels are physically correct, we employ a _Dual-Implementation Check._ Every complex quantum gate (specifically the 4-body terms) is validated against a classical pure-Python reference function. The test asserts that the unitary matrix implemented by the CUDA-Q kernel matches the theoretical Hamiltonian matrix constructed via NumPy, ensuring the AI code is optimizing the circuit without altering the underlying physics.

**Core Correctness Checks**

* **Check 1 (Symmetry Verification):**
    * The LABS Hamiltonian has two fundamental symmetries. We verify _Bit-Flip Symmetry_ by asserting that for any sequence $S$, `energy(S) == energy(-S)`. We also verify **Reversal Symmetry** by asserting `energy(S) == energy(S_reversed)`. These checks run automatically on every batch of results to catch potential bit-ordering errors in the quantum measurement process.
* **Check 2 (Ground Truth Validation):**
    * We use a _Small $N$ Oracle_ for regression testing. For problem sizes $N < 20$, the global minimum is calculable via brute force. We maintain a unit test for $N=6$ where the known minimum energy is 2.0. The pipeline is only considered "passing" if the hybrid solver successfully locates a solution with $E=2.0$, proving the Counteradiabatic terms are effectively guiding the system to the ground state.
---

### 5. Execution Strategy & Success Metrics

**Owner:** Hemal

**Agentic Workflow**

**Plan:**
To accelerate development and mitigate platform constraints (such as missing utility files), we adopted an **AI-Assisted Human-in-the-Loop Workflow**:
* **Context Management:** We maintained a persistent context window containing the mathematical derivation (Equation 15) and the CUDA-Q API reference. This allowed the AI to generate the complex 4-body interaction kernels from scratch without external dependencies.
* **Iterative Refactoring:** When platform limitations (e.g., missing 8vCPU access) occurred, the QA Lead fed the resource error logs back to the Agent. The Agent then refactored the simulation parameters (reducing `n_steps` or `N` temporarily) to ensure logical validation could proceed on smaller instances while waiting for full resource provisioning.

**Success Metrics**

* **Metric 1 (Solution Quality):** **Quantum Advantage Ratio > 1.1**.
    * *Definition:* The average energy of the Quantum-Seeded population must be at least 10% lower (better) than the Randomly-Seeded population. This proves the "Head Start" hypothesis.
* **Metric 2 (Computational Speedup):** **>50x Kernel Speedup**.
    * *Definition:* The time-to-sample for the GPU-backed `nvidia` target vs. the CPU-based `qpp` target for $N=30$.
* **Metric 3 (Scalability):** **Successful Execution at N=40**.
    * *Definition:* The workflow must handle the memory requirements of $2^{40}$ amplitudes without OOM (Out of Memory) errors by leveraging the `nvidia-mgpu` backend, surpassing the limit of standard laptop-class simulators ($N \approx 25$).

**Visualization Plan**

* **Plot 1: "The Head Start" (Energy Distribution Histogram)**
    * *Visual:* Overlaid histograms of Initial Energies for Random Seeds (Red) vs. Quantum Seeds (Blue).
    * *Goal:* Visually demonstrate that the Quantum distribution is shifted to the left (lower energy), validating that the hybrid solver starts "closer to the solution."
* **Plot 2: "Scaling Wall" (Time-to-Solution vs. N)**
    * *Visual:* Log-linear plot comparing the runtime of the Classical-Only solver vs. the Hybrid Quantum-Enhanced solver as $N$ increases from 20 to 40.
    * *Goal:* Highlight the lower slope of the Hybrid approach, indicating better scaling properties for large-scale radar problems.
---

### 6. Resource Management Plan

**Owner:** Abhiroop Gohar

**Optimization Strategy**

* **Phase 1: Zero-Cost Development (Current State):**
    * We are performing all logic verification, symmetry checks, and unit testing (for $N \le 12$) on the standard CPU-based simulator environment. This ensures our code is bug-free before we consume any high-performance compute credits.
* **Phase 2: "Burst" Execution (Next 12 Hours):**
    * Once the 8vCPU/GPU lab environment access is resolved, we will employ a **"Burst Execution"** strategy. We will run a single "dry-run" at $N=20$ to verify the backend integration, immediately followed by the production runs.
* **Phase 3: Failsafe Protocols:**
    * To prevent idle resource consumption, the GPU Acceleration PIC is designated as the "Kill Switch" operator. They are responsible for manually terminating the lab session immediately after the final output artifacts (plots/energy values) are saved, ensuring zero wasted runtime.
