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

### Before Rerunning (immediate) — DONE (2026-06-30)
0. ~~**Update to correct Planck 2018 parameters**~~ **DONE (2026-07-03)**
   - Old parameters were Planck 2013 (A_s=2.215e-9); now updated to Planck 2018 matching Eingorn et al. (2022) exactly
   - Updated files: gevolution/screening custom_settings.ini + diag_settings.ini (h, omega_b, omega_cdm, A_s, n_s)
   - Updated Python: template_job_py.txt, doppler-convergence.py, halo-doppler-convergence.py (wm, wd, H0)
   - Raytracing repo: only template_job_py.txt had cosmological params (all others clean)
   - Manuscript line 146: values updated + wrapped in \red{}; N_eff removed (T_cmb=0, N_ur=0 in both codes)
   - `\cite{aghanim2020}` citation is now correct

1. ~~**Fix convergence formula** in the ray-tracing post-processing code~~ **FIXED**
   - Was wrong: `κ = 1 - √(|g|² + 1/μ)` (valid only for true shear γ, not reduced shear g)
   - Now correct: `κ = 1 - 1/√(μ(1-|g|²))` (exact relation from the Jacobian: det A = μ⁻¹, A = (1-κ)I - Γ, |g|=|Γ|/(1-κ))
   - Fixed in: `/mnt/d/coding/raytracing/template_fitting_py.txt` line 325

2. ~~**Fix snapshot loading bug in ray-tracing job template**~~ **FIXED**
   - Old code: 6 snapshot files (snap000=z0.55 → snap005=z0.0); only 6 of 11 potential slots loaded; snap6–10 = zeros
   - Old branch `elif 0.55 <= redshift < 0.66` used snap6 (zeros) with wrong `z_1=0.66` — catastrophically wrong for z_s=0.6
   - New code: 7 snapshot files (snap006=z0.0 → snap000=z0.60), `potential=(7,...)`, snap0–snap6 all correctly loaded
   - Fixed branch: `elif 0.55 <= redshift <= 0.60`, `z_1=0.60`, correctly interpolates snap5(z=0.55)↔snap6(z=0.60)
   - Dead branches for z > 0.60 replaced by minimal `else` using snap6 constant (unreachable for z_s ≤ 0.6)
   - Fixed in: `/mnt/d/coding/raytracing/template_job_py.txt`

3. **Decide on |g| vs |γ| notation:** Reviewer 3 (point 4b) asks to use `g`. If adopted, notation must be updated consistently throughout manuscript. (Currently undecided.)

### Computation (the long part, ~2 weeks)
3. **Rerun all 21 paired simulations** with matched settings (`T_cmb=0`, `N_ur=0`) in both gev and scr.
4. **Rerun the full 3D-RBT ray-tracing pipeline** with corrected κ formula to regenerate all maps, PDFs, APS figures.

**Current status (as of 2026-07-07):** Running 5 paired realizations (reduced from 21 due to time constraint). Gev ray-tracing started July 7. Sequential execution required — server RAM (125 GB) prevents parallel sets (each set uses ~75 GB). Storage also prevents gev and scr running concurrently (5 sets × 124 GB = 620 GB each; server has 981 GB total).

**Revised timeline:**
- ~July 23: gevolution ray-tracing + fitting complete
- ~July 24: transfer gev final outputs to local, delete raw files (595 GB) to free server storage
- ~Aug 9: screening ray-tracing + fitting complete
- Aug 10–15: revision writing (~6 days available)
- **Aug 15: submission deadline**

Manuscript analysis (2026-07-07): ~60-65% structurally written; all specific numerical results must be replaced. Response letter: ~17-18 reviewer items, ~9 complete, ~5 placeholder (`\details`), ~4 contingent on new numbers. Estimated writing after results: 3–4 days focused work — August 15 is feasible but tight.

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
- Cosmology: **Planck 2018** (h=0.6766, omega_b=0.02242, omega_cdm=0.11933, A_s=2.105e-9, n_s=0.9665) — matching Eingorn et al. (2022) exactly; T_cmb=0, N_ur=0 in both codes (radiation excluded)

### Linux Simulation Server
- SSH: `ju@ju-astro` or `ju@103.159.2.161`, password: `@astrophysics`
- Server user: `ju` → `$HOME = /home/ju/`

**Directory layout on server:**
```
/home/ju/
├── gevolution/             # code repo + compiled binary ./gevolution
│   └── settings/           # custom_settings.ini + settings_01.ini (generated per run)
├── screening/              # code repo; binary also named ./gevolution
│   └── settings/
├── fuad-scripts/astro/     # fish helpers: astro.fish, read-seed.py, read-settings.py, ...
└── ssd-ext/
    ├── gevolution-phi/
    │   ├── outputs/output_01/ … output_21/   # lcdm_snap000_phi.h5 … snap006_phi.h5 + cdm Gadget2
    │   ├── rockstar-outputs/output_01/ …
    │   └── revolver-outputs/output_01/ …
    └── screening-phi/      # same structure
```

**Run command:** `mpirun -np 16 ./gevolution -n 4 -m 4 -s settings/settings_$i.ini` (16 MPI = 4 nodes × 4 procs)

**Fish helper functions** (local: `my-scripts/astro/astro-server.fish` = server: `~/fuad-scripts/astro/astro.fish`):
- `create_settings_seed g|s START END` — copies custom_settings.ini → settings_NN.ini, injects seed + output path
- `run_simulation g|s START END` — generates settings + runs mpirun for each index
- `run_gevolution START END` / `run_screening START END` — run only (settings must exist)
- Usage from code dir: `cd ~/gevolution && create_settings_seed g 1 5 && run_gevolution 1 5`

**Seeds (21 total):** `42 413 360 444 124 955 266 62 846 935 845 908 537 827 317 109 861 785 415 956 470`
- First 5 for initial comparison: 42, 413, 360, 444, 124 → output_01 through output_05

**Settings files** (updated 2026-06-30 with `T_cmb=0`, `N_ur=0`, 7 snapshots):
- gevolution: `/home/rafi/github/gevolution/settings/custom_settings.ini` (sync to `~/gevolution/settings/`)
- screening: `/home/rafi/github/screening/settings/custom_settings.ini` (sync to `~/screening/settings/`)
- `snapshot_redshifts = 0.6, 0.55, 0.44, 0.33, 0.22, 0.11, 0` → snap000=z0.60 … snap006=z0.00

### Ray-Tracing Pipeline (local templates at `/mnt/d/coding/raytracing/`)
- `template_job_py.txt` — 3D-RBT geodesic tracer; reads snapshot .h5 files, integrates null geodesics
- `template_fitting_py.txt` — ellipse fit → μ, |g| (reduced shear), κ; outputs `.npy` maps
- `generate_scripts_for_sets.sh` — run `./generate_scripts_for_sets.sh 1 2 3 4 5` from raytracing dir
  - Creates `set1/`, `set2/`, … with 8 job scripts + 1 `run_fitting.py` per set
  - Substitutes: `#OUTPUT_DIR#` → `output_01`, `#JOB_NUM#` → 1–8, `#SET_NUM#` → 01–05
  - `_SIM_TYPE_` in job filenames is replaced by a separate deployment script (not in repo)
- 8 MPI jobs × geodesics per job; HEALPix N_side=128 (196,608 pixels); integration `u = np.linspace(0, -1230, 1000)`
- Snapshot reads: `filenames[0]` = snap006 (z=0.0) → `interpolator_phi_snap0` … `filenames[6]` = snap000 (z=0.60) → `interpolator_phi_snap6`
- Output `.npy` files: `final_outputs/mu_#SIM_TYPE#_#SET_NUM#.npy`, `gamma_…`, `k_…`

---

---

## Physics Letters B — Doppler Lensing Paper (Address After Weak Lensing Revision)

The published Doppler lensing paper (Physics Letters B) has the same cosmological parameter citation issue identified in the weak lensing revision:
- Paper cites "Planck 2018" (`\cite{aghanim2020}` or equivalent) but lists Planck 2013 values (A_s=2.215e-9, n_s=0.9619, h=0.67556)
- Analysis code (doppler-convergence.py) used wm=0.312046079, wd=0.687953921 — also Planck 2013

**This is likely NOT a scientific erratum** — simulations and analysis were internally self-consistent with Planck 2013. Results are physically valid.

**Findings (verified 2026-07-03):**

1. **chi_s=1536 Mpc/h**: Correct Planck 2013 value is 1538.07 Mpc/h; Planck 2018 gives 1539.55 Mpc/h. The 1536 value is off by ~2 Mpc/h (~0.13%) — negligible, likely a rounded estimate. Not a scientific concern.

2. **T_cmb/N_ur mismatch**: CONFIRMED present in Doppler paper simulations (same old gevolution settings). From weak lensing diagnostics: mismatch caused +3.47% in P_Φ and +3.48% in P_δ at z=0. Halo velocities are driven by gravitational potential, so a similar artifact likely affected the published gev vs scr Doppler convergence comparison. This may require an **Erratum** (not just corrigendum) if the main gev/scr result is materially affected.

3. **Citation mislabeling** (Planck 2018 cited but Planck 2013 values used): minor, publisher's corrigendum sufficient on its own.

**Options for correcting published work (Physics Letters B / Elsevier):**
- **Erratum**: corrects factual errors (wrong results from code/settings error) — published as a linked notice, original paper remains. Appropriate if T_cmb/N_ur mismatch materially changed gev/scr comparison.
- **Corrigendum**: author-initiated correction, same process. Appropriate for citation mislabeling alone.
- **Retraction**: only for fundamental invalidity or fraud — not appropriate here.

**Action when returning to this:**
- Rerun Doppler lensing simulations with T_cmb=0, N_ur=0 in both codes and Planck 2018 parameters
- Compare new vs old results; if gev/scr difference changes significantly → submit Erratum to Physics Letters B
- If difference is negligible → corrigendum for citation label only

---

## Manuscript Cross-Check Findings (2026-07-03)

Cross-checked: `main-v.3.0.tex`, `response-for-main-v.3.0.tex`, gevolution and screening `custom_settings.ini`.

### Settings — Clean
Both settings files are consistent and correct: `T_cmb=0`, `N_ur=0`, 7 snapshots, boxsize=320 Mpc/h, Ngrid=256. No issues.

### Fix before rerun (no rerun needed)

**Planck citation wrong (line 146):** `\cite{aghanim2020}` must be changed to Planck 2013 (Ade et al. 2014, arXiv:1303.5076). A_s=2.215e-9 is definitively Planck 2013 — Planck 2018 gives A_s≈2.101e-9 (~5% lower). n_s=0.9619 and N_eff=3.046 also match 2013, not 2018.

### Fix after rerun (manuscript + response will be fully rewritten)

| Issue | Location | Note |
|-------|----------|------|
| ~5% result presented as real | Abstract, Results, Conclusions | Pre-fix artifact; entire narrative changes after rerun |
| T_cmb/N_ur fix not in manuscript body | Nowhere | Must be added to methodology section |
| 23 `\red{}` + 9 `\blue{}` draft markers | Throughout | All must be resolved |
| κ formula blue editorial block | Lines 178–182 | Exposes wrong code; remove after rerun confirms formula |
| \|g\| vs \|γ\| notation unresolved | Throughout | Decision pending |
| Source redshifts z_s=0.05,0.2,0.4 not exact snapshots | Line 189 | Say "reached by interpolation", not "corresponding snapshots" |
| 4 missing citations | Lines 148, 166, 530, 532 | + raw GitHub URL at line 152 |
| Editor comment response | Response line 189 | `\details` — empty |
| Reviewer 3 final comment | Response line 1702 | `\details` — empty |
| Reviewer 1 Major 1a–1d, Major 2 | Response lines 228–267 | "will touch on later" — all deferred |

### Complete items in response
- Root-cause investigation (T_cmb/N_ur, diagnostic cases) — fully documented
- Reviewer 2 — fully complete
- Reviewer 3 sub-comments 2, 3, 4a–d, 5a–b — substantive responses written

---

## Codebase Compilation for Claude Web Upload

Run from the repo root (`~/gevolution` or `~/screening`) in fish to produce a
single text file suitable for uploading to Claude web.

Use `custom_settings.ini` for production runs (the active simulation settings);
use `diag_settings.ini` only for diagnostic/debug uploads.

```fish
# gevolution
echo "=== GEVOLUTION REPO COMPILATION ===" > combined_codebase_gev.txt
for file in settings/custom_settings.ini main.cpp gevolution.hpp background.hpp
    echo -e "\n====================================================================\nFILE: $file\n====================================================================\n" >> combined_codebase_gev.txt
    cat $file >> combined_codebase_gev.txt
end
```

```fish
# screening (run from ~/screening)
echo "=== SCREENING REPO COMPILATION ===" > combined_codebase_scr.txt
for file in settings/custom_settings.ini main.cpp gevolution.hpp background.hpp
    echo -e "\n====================================================================\nFILE: $file\n====================================================================\n" >> combined_codebase_scr.txt
    cat $file >> combined_codebase_scr.txt
end
```

Output files: `~/gevolution/combined_codebase_gev.txt` and `~/screening/combined_codebase_scr.txt`.
