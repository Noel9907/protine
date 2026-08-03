# Measuring the Epistasis Gap: a calibrated simulator and an honest ceiling for combinatorial perturbation prediction

Working plan, drafted 2026-07-30. Four-person final-year BTech major project.

---

## 1. The thesis

The field of combinatorial perturbation prediction cannot currently tell whether its
methods work, because the evaluation data is too small and too noisy to separate them.

Two published results sit in direct contradiction:

- GEARS (Nature Biotechnology, 2023) reports that it predicts five genetic-interaction
  subtypes and is roughly twice as accurate as the next best method on the strongest
  interactions.
- Ahlmann-Eltze, Huber & Anders (Nature Methods, 2025) report that deep-learning
  perturbation models, GEARS among them, do not outperform a deliberately simple
  additive linear baseline. A separate Nature Methods benchmark of 27 methods across
  29 datasets reaches a compatible conclusion.

Both cannot be straightforwardly true. This project builds the instrument that settles
the question, and then uses it.

**The fact that makes this tractable and urgent.** Essentially every claim in this field
about predicting *combinations* rests on Norman et al. 2019: 131 double perturbations in
a single cell line (K562, CRISPRa, ~111k cells, 19,264 genes). The other commonly used
combinatorial set, Wessels, adds 158 double perturbations. That is on the order of 290
measured gene pairs in existence, ever, with transcriptome readout. Meanwhile
single-perturbation atlases run to genome scale.

You cannot learn a general theory of genetic interaction from 131 examples, and you very
likely cannot *evaluate* one either. Nobody has quantified how badly.

---

## 2. Prior art — what is genuinely taken, and what is not

Checked 2026-07-30. Do not restate any of these as novel.

| Claim | Status |
|---|---|
| Scoring genetic-interaction subtypes separately | **Taken.** GEARS does synergy, suppression, neomorphism, epistasis, redundancy. |
| A performance ceiling from experimental reproducibility | **Partly taken.** Split-half "technical duplicate baseline" exists (arXiv 2506.22641). |
| GRN simulators for single-cell data | **Taken.** SERGIO, GRouNdGAN, dyngen, BoolODE. |
| GRN simulator with *genetic perturbations* | **Taken.** GeneSPIDER2 (2024) claims to be first, validated against real Perturb-seq. |

What survives, and is the actual contribution:

1. **No simulator generates combinatorial perturbations with a controllable, mechanistic,
   ground-truth genetic-interaction structure.** GeneSPIDER2 does single knockdowns.
   Nobody lets you dial in "make this gene pair synergistic with effect size X" and get
   realistic single-cell counts back.
2. **The existing split-half ceiling is a known underestimate** — the literature says so
   explicitly — and nobody has computed a bias-corrected ceiling.
3. **Nobody has computed the ceiling for *epistasis specifically*.** Epistasis is a
   difference of differences, `δ_AB − δ_A − δ_B`. It compounds noise from three separately
   estimated quantities. Its signal-to-noise ratio must be far worse than that of main
   effects, and the magnitude of that gap is unmeasured. This is the headline number.
4. **Nobody has asked how much data would be enough.** The simulator answers it.

---

## 3. The four pillars

One per team member. Each is individually defensible in a viva, and each produces a
result even if the others stall.

### Pillar A — The simulator (systems / scientific computing)

Build from scratch a stochastic gene-regulatory simulator producing realistic
single-cell count matrices under single and double perturbation.

- **Promoter logic is where epistasis comes from.** Use a thermodynamic (MWC-style)
  promoter-occupancy model: two transcription factors binding a shared promoter with a
  cooperativity parameter ω. Independent binding (ω = 1) yields near-additive behaviour;
  cooperative binding yields synergy; steric exclusion yields suppression. **Epistasis
  emerges from a physical parameter you control rather than being injected as a fudge
  term.** This is the single most important design decision in the project — it is what
  makes the ground truth meaningful.
- **Transcriptional bursting via the telegraph model** (promoter on/off, burst size and
  frequency). This is what actually generates the overdispersed count distributions seen
  in scRNA-seq; Gaussian noise will not reproduce them.
- **Hybrid tau-leaping / SSA** for the chemical kinetics. Numerical stability under stiff
  systems is a real problem here and solving it is real work.
- **GPU parallelism over cells.** Each cell is an independent stochastic trajectory, so
  this is embarrassingly parallel — a rare case where a 4GB card is entirely adequate.
  Target 100k cells × a few thousand genes.
- **Technical noise layer:** UMI resampling, library-size variation, dropout as a function
  of mean expression, and per-guide CRISPRa efficiency variability (a real confound that
  most simulators ignore).

### Pillar B — Calibration (simulation-based inference / statistics)

Make the simulator's output statistically indistinguishable from real Norman K562 data.

- Define summary statistics: mean-variance relationship, dropout-vs-mean curve,
  library-size distribution, distribution of single-perturbation effect sizes, and the
  eigenvalue spectrum of the gene-gene correlation matrix.
- Implement **SMC-ABC** (sequential Monte Carlo approximate Bayesian computation) from
  scratch to fit simulator parameters. Neural posterior estimation is the stretch goal.
- **Success criterion — the discriminator test.** Train a classifier to distinguish real
  cells from simulated ones. If it cannot beat chance, calibration has succeeded. This is
  crisp, gradeable, and makes an excellent slide.

### Pillar C — The ceiling and the audit (statistics / the headline result)

- Build a hierarchical measurement-error model of per-perturbation effect estimates,
  using per-cell counts to model cell-level sampling noise and guide-level variability.
- Derive a **bias-corrected reliability estimate** (Spearman-Brown-style correction to
  split-half, extended to the multivariate delta case), fixing the known underestimate.
- **Propagate to epistasis.** Compute the achievable ceiling on `δ_AB − δ_A − δ_B`.
- **Validate the estimator on simulated data first**, where true epistasis is known, then
  apply it to Norman. This closes the loop: Pillar A exists partly to prove Pillar C is
  correct.
- Re-run GEARS, the additive baseline, CPA and scGPT under a protocol stratified by true
  non-additivity, with the ceiling drawn on every plot.
- **Address regression to the mean.** Selecting the "top-K strongest interactions" selects
  partly on noise — and that is precisely how the 2x claim is reported. Quantify it.

> **Use the original authors' implementations for every method you audit.** Reimplementing
> GEARS from scratch would make the critique worthless. "From scratch" applies to your
> contributions, not to the things you are auditing.

### Pillar D — Model, power analysis, tool (ML / product)

- **Model:** predict the epistatic residual directly, so the additive part is handled
  analytically and all capacity goes where it matters. Give it interaction structure by
  construction — bilinear or tensor factorisation over perturbation embeddings, or an
  inductive bias mirroring the simulator's promoter logic. Pretrain on simulation,
  fine-tune on the 131 real pairs. This is sim-to-real transfer for Perturb-seq.
- **Power analysis — the most quotable output of the project.** Use the simulator to
  answer: how many double perturbations must a wet lab measure, at what sequencing depth,
  to detect epistasis of a given effect size? This produces a design curve that is
  directly actionable for experimentalists.
- **Tool:** pick two genes; see additive vs GEARS vs ours vs ground truth, with error bars
  from the noise model, and an epistasis heatmap over gene-pair space. Plus a simulator
  playground — build a small network by hand, set ω, watch the epistasis it produces.

---

## 4. Timeline (Aug 2026 – Apr 2027)

| Phase | Window | Deliverable |
|---|---|---|
| 0 | Aug–Sep | Norman preprocessing reproduced; additive baseline reproduced; GEARS running. Simulator v0 (deterministic ODE, small network). **Reproduction report.** |
| 1 | Oct–Nov | Simulator v1 (stochastic, GPU). Ceiling estimator derived and validated on simulation. **The ceiling number for Norman — first publishable-shaped result.** |
| 2 | Dec–Jan | SMC-ABC calibration; discriminator test; stratified re-evaluation of published methods. **The audit.** |
| 3 | Feb–Mar | Residual model, sim-to-real transfer, power analysis, interactive tool. |
| 4 | Apr | Thesis, paper draft, viva. |

The first real result lands in November. That de-risks the whole year.

---

## 5. Risks, and why the project survives them

**The simulator never calibrates convincingly.** Then sim-to-real transfer fails and
reviewers discount the synthetic results. *Mitigation:* Pillar C stands alone. There the
simulator is used only to validate the ceiling estimator, not to make biological claims,
so a merely qualitatively-realistic simulator is still sufficient.

**Nothing beats the additive baseline.** This is the *most likely outcome* and it must be
a win from day one. The result "here is the ceiling, here is how far every published
method is from it, and here is how much data would be required to close the gap" is a
stronger contribution than another model on a leaderboard.

> **Do not let the team's success criterion be "our model wins."** Write the evaluation
> protocol and the success criteria **before** running the audit, and timestamp them
> (a dated commit is enough). Pre-registering the analysis is what makes a negative
> result publishable rather than a disappointment, and examiners respond well to it.

**Scope.** This is ambitious. The phase sequencing is deliberate: partial completion still
yields a coherent thesis.

---

## 6. Compute

Near-zero rented GPU spend. The simulator parallelises over cells and fits comfortably in
4GB. ABC is the compute sink but each simulator run is cheap, so it runs overnight on
local hardware. All datasets are public and gigabyte-scale.

---

## 7. Key references

- Ahlmann-Eltze, Huber & Anders — *Deep-learning-based gene perturbation effect prediction
  does not yet outperform simple linear baselines*, Nature Methods 2025.
  https://www.nature.com/articles/s41592-025-02772-6
- *Benchmarking algorithms for generalizable single-cell perturbation response prediction*,
  Nature Methods 2025. https://www.nature.com/articles/s41592-025-02980-0
- Roohani et al. — *Predicting transcriptional outcomes of novel multigene perturbations
  with GEARS*, Nature Biotechnology 2023.
  https://www.nature.com/articles/s41587-023-01905-6
- *Diversity by Design: Addressing Mode Collapse Improves scRNA-seq Perturbation Modeling
  on Well-Calibrated Metrics* — technical duplicate ceiling. https://arxiv.org/abs/2506.22641
- Dibaeinia & Sinha — *SERGIO*, Cell Systems 2020. https://pubmed.ncbi.nlm.nih.gov/32871105/
- *GeneSPIDER2: large scale GRN simulation and benchmarking with perturbed single-cell
  data*, NAR Genomics and Bioinformatics 2024.
  https://academic.oup.com/nargab/article/6/3/lqae121/7759978
- Norman et al. 2019 Perturb-seq (K562 CRISPRa, 131 double perturbations).
- Arc Institute Virtual Cell Challenge 2025 wrap-up.
  https://arcinstitute.org/news/virtual-cell-challenge-2025-wrap-up
