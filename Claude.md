# Weak Lensing Manuscript Revision — Context for Claude

## Paper
**Title:** Impact of Screening on Weak Gravitational Lensing Observables in Relativistic N-body Simulations
**Authors:** Mubtasim Fuad, Sonia Akter Ema, Md Rasel Hossen
**Journal:** Scientific Reports
**Manuscript ID:** 0b29de9e-5337-4a91-a008-3fc630afa5ca
**Revision deadline:** 15 August 2026 (extension granted by editor)

## Key Files
- Manuscript v3: `/home/rafi/Documents/weak-lensing-draft/main-v.3.0.tex`
- Revision response: `/home/rafi/Documents/weak-lensing-draft/revision-responses/response-for-main-v.3.0.tex`
- References: `/home/rafi/Documents/weak-lensing-draft/references.bib`

## What the Paper Does
Controlled comparison of weak gravitational lensing observables (convergence κ, shear |γ|) between two relativistic N-body codes:
- **gevolution (gev):** Full weak-field GR, solves nonlinear Einstein equations
- **screening (scr):** Linearized perturbation approach, Yukawa-like Helmholtz equation, ~40% faster

Uses the 3D Ray Bundle Tracer (3D-RBT) algorithm on 21 paired realizations per code, at source redshifts z_s = 0.05, 0.2, 0.4, 0.6. Reports PDFs, tail probabilities, and angular power spectra (APS).

**Original claimed result:** scr enhances lensing observables by ~5% relative to gev across all scales and redshifts.

---

## Current Situation (as of 2026-06-29)

### Root Cause of ~5% Offset — IDENTIFIED and CONFIRMED

The ~5% APS enhancement was an **artifact, not physics**. Root cause: asymmetric radiation treatment.

- `gev` used default `T_cmb = 2.7255 K` and `N_ur = 3.046` → activates photon/neutrino radiation in background evolution
- `scr` ignores radiation in its background evolution by design

This gave the two codes different expansion histories from z=100 onward, artificially inflating all weak-lensing statistics.

**Fix:** Set `T_cmb = 0` and `N_ur = 0` in **both** codes.

### Diagnostic Cases Run (on dedicated paired diagnostic realization)

| Case | T_cmb, N_ur | background.hpp | ΔP_Φ (z=0) | ΔP_δ (z=0) | ΔH/H (z=100) |
|------|-------------|----------------|------------|------------|---------------|
| Original | gev: default; scr: ignored | different | +3.47% | +3.48% | −1.45% |
| Case I | gev: 0,0; scr: unchanged | different | +0.004% | +0.013% | +0.005% |
| Case II | both: 0,0 | different | −0.007% | +0.002% | 0.000% |
| Case III | both: 0,0 | same (scr version) | −0.007% | +0.002% | 0.000% |
| Eingorn et al. (2022) | radiation neglected in both | — | ≤0.04% | — | — |

Cases II and III are identical → `background.hpp` difference has no effect, only `T_cmb`/`N_ur` settings matter.

With matched settings (Case II/III):
- `ΔP_Φ ≤ 0.02%` at all k, all z ✓
- `ΔP_ρN = 0.000%` at all k, all z (particle positions identical at z=100) ✓
- `ΔH/H = 0.000%` at all z ✓
- `ΔD/D = 0.000%` at all z (linear growth factor identical) ✓
- Φ field cross-correlation = 1.00000000 at z=100 ✓

Reproduces Eingorn et al. (2022) benchmark. ✓

### Remaining Understood Issue — NOT a Bug

`P_delta(k<0.05, z=100) = 3.234%` residual persists. This is explained (confirmed June 23–25):
- `gev` outputs **energy density contrast** `δT⁰₀/T̄⁰₀` (Eingorn et al. notation: `δε/ε̄`)
- `scr` outputs **mass density contrast** `δρ/ρ̄`
- Relation: `δε = δρ/a³ + 3ρ̄Φ/a³` — the `3ρ̄Φ` term causes divergence at large scales (small k) where `Φ/δ` is non-negligible
- This is a **code-architecture labeling difference**, not a physical discrepancy
- Eingorn et al. (arXiv:2309.02989) explicitly document this distinction
- The particle Newtonian density `P_ρN` and `P_δN` are 0.000% different at z=100 ✓
- The residual decays from 3.234% at z=100 to 0.041% at z=0 (because δ grows and the Yukawa term becomes negligible)

The reviewer's request to check "particle power spectrum P(k) at z=100" is answered by `P_δN = 0.000%`.

### Power Spectrum Output Taxonomy (confirmed June 26)

| Output | Definition | Codes use same projection? | ΔP at z=100 |
|--------|-----------|--------------------------|-------------|
| P_Φ | Gravitational potential | identical | 0.000% |
| P_ρN | Newtonian matter density (CIC) | identical | 0.000% |
| P_δN | Newtonian density contrast δρ/ρ̄ (CIC) | identical | 0.000% |
| P_T00 | Relativistic energy density T⁰₀ | differs | 3.234% (k<0.05) |
| P_δ | Energy density contrast δT⁰₀/T̄⁰₀ | differs | 3.234% (k<0.05) |

Within scr: `P_δ = P_δN` (confirmed 0.000%), because scr's T⁰₀ projection is simple mass counting.

---

## What Needs to Be Done

### Before Rerunning (immediate)
1. **Fix convergence formula** in the ray-tracing post-processing code:
   - Current (wrong): `κ = 1 - √(|g|² + 1/μ)`
   - Correct (exact): `κ = 1 - 1/√(μ(1-|g|²))`
   - This is a sub-percent correction in the weak-lensing regime but must be fixed for rigor before rerunning to avoid a second full rerun.
   
2. **Decide on |g| vs |γ| notation:** Reviewer 3 (point 4b) asks to use `g`. If adopted, notation must be updated consistently throughout manuscript. (Currently undecided.)

### Computation (the long part, ~2 weeks)
3. **Rerun all 21 paired simulations** with matched settings (`T_cmb=0`, `N_ur=0`) in both gev and scr.
4. **Rerun the full 3D-RBT ray-tracing pipeline** with corrected κ formula to regenerate all maps, PDFs, APS figures.

### After Rerun — Manuscript and Response Updates
5. Update all numerical results (the ~5% numbers will change — expected to become a small genuine physical signal ~0.02% or a new, correctly-measured comparison).
6. Write actual responses for items currently marked `\details` or "will touch on it later":
   - Editor comment response (currently `\details`)
   - Reviewer 1, Major Comment 1 main text + sub-items 1a, 1b, 1c, 1d
   - Reviewer 1, Major Comment 2 (resolution limits at ℓ>1000)
   - Reviewer 3, final comment (currently `\details`)
7. Address figure layout reorganization (Reviewer 1, point 5): combine panels into 2×2 grids.
8. Add missing citations (several `\red{need to add citation}` markers throughout manuscript).
9. Remove `\red{}` and `\blue{}` draft markers before final submission.

### Optional / Pending Guidance
- Contact Eingorn et al. to confirm matched settings and get confirmation of the `P_delta` code-architecture interpretation.
- Decide whether to mention CPU hours (currently conflicting numbers in manuscript).

---

## Key Physical Context

### Why the Real Comparison Still Matters
With matched settings, the genuine perturbation-level difference between gev and scr is:
- `ΔP_δN` grows from 0.000% at z=100 to mean ~0.08% at z=0 (max ~8% at individual k-modes)
- This reflects higher-order GR corrections (frame-dragging, nonlinear metric terms) that gev includes but scr omits
- The actual scientific comparison — measuring the impact of the linearized screening approximation on weak-lensing observables — is still valid; results just need to be recomputed with fair settings

### Known Issues in Manuscript (blue/red highlighted in v3.0)
- `\blue{...}` = author notes/questions, not finalized text
- `\red{...}` = text to be revised/added
- `\details` = response not yet written

### Simulation Setup
- Box: (320 h⁻¹ Mpc)³, 256³ particles, resolution 1.25 h⁻¹ Mpc
- 21 paired realizations (same seeds for gev and scr)
- Source redshifts: z_s = 0.05, 0.2, 0.4, 0.6
- HEALPix maps at N_side=128 (196,608 pixels, ~0.46° resolution)
- ℓ_max = 128 for APS
- Cosmology: Planck 2018 (h=0.67556, Ω_CDM=0.2638, etc.)
