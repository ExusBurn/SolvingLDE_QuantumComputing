# Solving Linear Differential Equations with Quantum Computing (LCHS)

**Course:** Numerical Solutions of Differential Equations (NSDE) — Final Project  
**Institution:** Indian Institute of Science (IISc)  
**Notebook:** `lchs.ipynb`

---

## Overview

This project implements the **Linear Combination of Hamiltonian Simulation (LCHS)** algorithm to solve the 1D diffusion (heat) equation on a quantum computer simulator. The solution is benchmarked against the classical **Forward-Time Central-Space (FTCS)** finite difference method.

The diffusion equation being solved is:

$$\frac{\partial u}{\partial t} = \alpha(t) \frac{\partial^2 u}{\partial x^2}, \quad x \in [0, 2\pi], \quad \text{(periodic BCs)}$$

Two cases are studied: constant diffusivity $\alpha = 0.25$, and time-varying diffusivity $\alpha(t) = t$.

---

## Background: What is LCHS?

The LCHS method (Dong et al.) converts the simulation of a **non-unitary** evolution $e^{-Lt}$ (which governs diffusion) into a **linear combination of unitary** quantum evolutions — something quantum computers can natively run.

The key identity is:

$$e^{-Lt} = \int_{-\infty}^{\infty} g(k) \cdot e^{i(I + ikL)t} \, dk$$

where $g(k)$ is a carefully chosen kernel function:

$$g(k) = \frac{1}{C_\beta (1 - ik) \exp\left((1 + ik)^\beta\right)}, \quad \beta = 0.7$$

Each $e^{i(I+ikL)t}$ term is a unitary evolution that can be run as a quantum circuit. The integral is approximated by a finite sum using **composite Gauss-Legendre quadrature**, giving $J = 64$ unitary terms.

---

## Algorithm: Step-by-Step

### 1. Problem Discretisation

- Spatial domain $[0, 2\pi]$ is discretised into $N = 16$ equally spaced points with spacing $dx$.
- The diffusion operator $\frac{\partial^2}{\partial x^2}$ is approximated by the **second-order central finite difference** Laplacian matrix $L \in \mathbb{R}^{16 \times 16}$ with periodic boundary conditions:

$$L_{ii} = \frac{2\alpha}{dx^2}, \quad L_{i,i\pm1} = \frac{-\alpha}{dx^2}$$

### 2. LCHS Quadrature Setup

- The integral over $k \in [-K, K]$ with $K = 4$ is split into panels of width $h = 0.5$.
- Within each panel, $Q = 4$ Gauss-Legendre nodes are used (computed from scratch via Newton's method on Legendre polynomials).
- This produces $J = K\_\text{panels} \times Q = 16 \times 4 = 64$ quadrature nodes $k_j$ and weights $c_j$.
- The 1-norm $\|c\|_1 = \sum |c_j|$ acts as a normalisation factor $c_1$ for extracting the solution from the quantum state.

### 3. Quantum Circuit Construction (per quadrature node)

For each node $k_j$, the following circuit is built using Qiskit:

| Register | Qubits | Role |
|---|---|---|
| `sys` | 4 | Encodes the 16-point solution vector $u(x, t)$ |
| `k` | 6 | Ancilla register for the LCHS superposition over $k$ |

**Circuit steps:**
1. **State Preparation** — $\hat{V}$: Prepares the initial condition $u_0$ as a normalised quantum amplitude vector on the `sys` register.
2. **Ancilla Preparation** — Prepares a uniform superposition over $J = 64$ LCHS terms on the `k` register using a `StatePreparation` gate.
3. **SELECT Oracle** — Applies the controlled unitary $e^{i(I + ik_j L)t}$ on the `sys` register, conditioned on the `k` register. Each term is implemented as a `UnitaryGate` via `scipy.linalg.expm`.
4. **Measurement / Statevector extraction** — The first $N = 16$ amplitudes of the statevector are extracted and multiplied by $c_1$ to recover $u(x, T)$.

### 4. Classical Reference: FTCS

The classical solver uses the explicit FTCS scheme:

$$u_i^{n+1} = u_i^n + \frac{\alpha \Delta t}{dx^2}\left(u_{i+1}^n - 2u_i^n + u_{i-1}^n\right)$$

with CFL $= 0.25$ for stability. This serves as the ground truth for comparison.

---

## Experiments

### Case 1: Constant Diffusivity ($\alpha = 0.25$)

Three initial conditions of increasing spectral complexity are tested:

| IC | Expression |
|---|---|
| IC1 | $\sin(x)$ |
| IC2 | $\sin(x) + \tfrac{1}{2}\sin(2x)$ |
| IC3 | $\sin(x) + \tfrac{1}{2}\sin(2x) + \tfrac{1}{3}\sin(3x)$ |

Snapshots are taken at 11 times from $T = 0$ to $T = 2$ (every 0.2 s).

### Case 2: Time-Varying Diffusivity ($\alpha(t) = t$)

The same three ICs are run with $\alpha(t) = t$ — meaning diffusion is slow initially and accelerates over time. Snapshots are taken at $T \in \{0.1, 0.2, \ldots, 1.0\}$.

---

## Output Plots

All plots are saved to `plots/` and `results_*/` directories.

### `plots/`
| File | Description |
|---|---|
| `plot_ic1_sinx.png` | FTCS vs LCHS spatial profiles at $T = 0, 1, 2$ — IC1 |
| `plot_ic2_sin_sin.png` | FTCS vs LCHS spatial profiles — IC2 |
| `plot_ic3_three_sin.png` | FTCS vs LCHS spatial profiles — IC3 |

### `results_constant_alpha/`
| File | Description |
|---|---|
| `fig1_combined_spatial.png` | Spatial profiles for all 3 ICs side by side |
| `fig2_combined_amplitude.png` | Max amplitude decay over time for FTCS and LCHS |
| `ic3_spectra/fig3_logscale_spectra.png` | Log-scale Fourier mode magnitudes over time |
| `ic3_spectra/fig4_fitted_decay_rates.png` | Fitted per-mode exponential decay rates |
| `ic3_spectra/fig5_spectral_error.png` | Per-mode error between LCHS and FTCS in Fourier space |
| `ic3_spectra/fig6_spectral_heatmap.png` | Heatmap of Fourier amplitudes (mode vs time) |
| `ic3_spectra/fig7_per_mode_decay.png` | Individual mode amplitude decay curves |

### `results_alpha_t/`
| File | Description |
|---|---|
| `fig1_ftcs_profiles_alphat.png` | FTCS spatial profiles at multiple times — $\alpha(t)=t$ |
| `fig2_lchs_profiles_alphat.png` | LCHS spatial profiles — $\alpha(t)=t$ |
| `fig3_amplitude_decay_alphat.png` | Amplitude decay comparison for all 3 ICs — $\alpha(t)=t$ |

---

## Parameters

| Parameter | Value | Meaning |
|---|---|---|
| `N` | 16 | Spatial grid points |
| `Q` | 4 or 8 | Gauss-Legendre nodes per panel |
| `K_trunc` | 4.0 | Quadrature truncation range $[-K, K]$ |
| `h1` | 0.5 | Panel width for composite GL quadrature |
| `beta` | 0.7 | LCHS kernel decay exponent |
| `CFL` | 0.25 | Stability parameter for FTCS timestep |
| `ALPHA` | 0.25 | Diffusivity (constant-$\alpha$ case) |
| `J` | 64 | Total number of LCHS unitary terms |

---

## Dependencies

```
numpy
scipy
matplotlib
qiskit
qiskit-aer (optional, for circuit simulation)
```

Install with:
```bash
pip install numpy scipy matplotlib qiskit
```

---

## Running the Notebook

```bash
jupyter notebook lchs.ipynb
```

Run cells in order. The final section ("All Plots & Algorithm, Everything") is the self-contained production run that generates all saved figures.

---

## References

- Quantum algorithm for linear non-unitary dynamics with near-optimal dependence on all parameters (Dong et al.)
- Lecture Notes on Quantum Algorithms for Scientific Computation(Lin Lin)
- https://quantum.cloud.ibm.com/docs/en/guides
