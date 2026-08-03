# Build Plan — Wasserstein Cryo-EM

Linear order. Do these in sequence. Each phase ends with a thing that works.

**Start:** August 2026 · **End:** April 2027 · **~35 weeks**

---

## Phase 0 — Setup
**Weeks 1–2**

- [ ] Python + PyTorch + CUDA working on the 4GB card
- [ ] Install cryoSPARC or RELION (to run as a baseline later, not to copy)
- [ ] Download one small real dataset from EMPIAR
- [ ] Download CryoBench

**Read:** cryoSPARC guide, "Expectation Maximization in Cryo-EM" page. That's it for now.

**Done when:** you can load a particle stack and display 100 particle images in a grid.

---

## Phase 1 — Forward model
**Weeks 3–5**

The thing that makes fake data. Everything later is validated against this.

- [ ] Load a PDB structure → 3D density volume on a grid
- [ ] Rotate volume by `R ∈ SO(3)`, project along z → 2D image
- [ ] Apply CTF (contrast transfer function)
- [ ] Add noise at realistic SNR (very low — like 0.05)

**Done when:** your fake particle images are visually indistinguishable from real EMPIAR ones.

> This phase is short but do not rush it. A bug here silently poisons everything downstream.

---

## Phase 2 — Reconstruction, poses known
**Weeks 6–9**

Easiest version of the problem. It will work. Build confidence and infrastructure.

- [ ] Implement the Fourier slice theorem: 2D FFT of a projection = a central slice of the 3D FFT
- [ ] Insert slices into a 3D Fourier grid at known orientations
- [ ] Handle interpolation onto the grid properly (this is where accuracy leaks)
- [ ] Weight by CTF, solve the least-squares problem
- [ ] Compute FSC (Fourier Shell Correlation) vs ground truth

**Done when:** you feed in 10,000 simulated images with known angles and recover the original volume. FSC above 0.5 out to decent resolution.

**This is your first real milestone.** Screenshot it.

---

## Phase 3 — Ab initio, poses unknown
**Weeks 10–17**

The hard classical part. Now you don't know the angles.

- [ ] Expectation-Maximization: for each image, a posterior distribution over SO(3)
- [ ] Discretize SO(3) sensibly (start coarse — a few thousand orientations)
- [ ] Coarse-to-fine angular search (branch-and-bound style pruning)
- [ ] Random initialization + SGD, cryoSPARC-style
- [ ] Test on synthetic first, then a real EMPIAR dataset

**Done when:** you reconstruct a recognizable volume from scratch with no pose information.

**Risk:** this is the phase most likely to eat time. Budget the full 8 weeks. If you're behind by week 15, downsample images harder (64³ volumes are fine) and move on.

---

## Phase 4 — Baselines
**Weeks 18–21**

Before you can be better, you need something to be better *than*.

- [ ] Implement linear heterogeneity yourself: covariance estimation + PCA over volumes
- [ ] Install and run **cryoDRGN** on a CryoBench dataset
- [ ] Install and run **RECOVAR** on the same dataset
- [ ] Build the evaluation harness: per-conformation FSC, latent-space plots, ground-truth comparison

**Done when:** you have a results table with three baselines in it and an empty row labelled "ours".

---

## Phase 5 — The optimal transport engine
**Weeks 22–27**

Standalone component. Build and test it in isolation before wiring it in.

- [ ] Entropic OT (Sinkhorn) between two 2D densities — toy case, verify against a known solution
- [ ] Extend to 3D grids
- [ ] **Separable/convolutional Sinkhorn** — the key trick, turns the cost into 1D convolutions along each axis. Without this it is far too slow.
- [ ] GPU implementation, benchmark it
- [ ] Wasserstein geodesics and barycenters between two volumes
- [ ] Make it differentiable (you need gradients through it)

**Done when:** you can compute a Wasserstein geodesic between two conformations of the same protein and the morph *flows* instead of dissolving. Render it as a GIF — this single animation is the visual proof your whole thesis rests on.

**Risk:** speed. Sinkhorn in a training loop is expensive. Measure early. If it's too slow, options are: coarser grids, fewer Sinkhorn iterations, sliced-Wasserstein approximation as a fallback.

---

## Phase 6 — The method
**Weeks 28–33**

Wire Phase 5 into Phase 3.

- [ ] Encoder: image → latent `z`
- [ ] Decoder: `z` → 3D volume
- [ ] **The new part:** the latent space carries the Wasserstein metric — distance in `z` equals mass transport between the decoded volumes
- [ ] Train jointly with pose estimation from Phase 3
- [ ] Evaluate on CryoBench against your three baselines
- [ ] Ablation: same model with Euclidean latent vs Wasserstein latent. This isolates your contribution.

**Done when:** the results table has a number in the "ours" row.

> If you don't beat the baselines, the ablation still shows what the geometry does. That is a legitimate result — report it honestly.

---

## Phase 7 — Write and demo
**Weeks 34–35**

- [ ] Side-by-side morph: cryoDRGN vs yours, same molecule
- [ ] Interactive viewer — drag through the latent space, watch the molecule move
- [ ] Thesis
- [ ] Paper draft if the numbers are good

---

## Rules

**Validate on synthetic before real.** Every single phase. You always have ground truth from Phase 1 — use it.

**Downsample aggressively.** 64³ or 128³ volumes for all development. Full resolution only at the end. Your 4GB card is fine for everything up to Phase 6.

**Rent GPUs only in Phase 5 and 6.** Everything before that runs locally.

**Keep the ablation sacred.** Euclidean vs Wasserstein latent, everything else identical. That comparison is your entire novelty claim — protect it from confounds.

---

## Datasets

| What | Where | Use |
|---|---|---|
| CryoBench | github (ez-lab) | Main benchmark, has ground truth |
| EMPIAR-10076 (ribosome) | EMPIAR | Classic heterogeneity test case |
| Your own simulated data | Phase 1 | Debugging everything |

## Key papers

Read these three properly. Skim the rest as needed.

1. **cryoSPARC** (Nature Methods 2017) — how ab initio reconstruction works
2. **cryoDRGN** (Nature Methods) — the main baseline you're attacking
3. **Computational Optimal Transport** (Peyré & Cuturi) — the OT book, chapters on Sinkhorn

Later: RECOVAR (PNAS 2025), 3DFlex, CryoBench (NeurIPS 2024).
