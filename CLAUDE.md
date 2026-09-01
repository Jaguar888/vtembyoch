# CLAUDE.md — Virtual Embryo Challenge, Task 1

This file is the standing brief for this repository. Read it at the start of every session. Where
it conflicts with the starter kit or the live challenge site, they win — tell me about the conflict.

---

## 1. Mission

You are building a submission for **Task 1 of the NeurIPS 2026 Virtual Embryo Challenge**
(virtualembryo.ai): given single-cell RNA data from mouse heart-centred dissections at **E8.5 and
E9.5**, predict the **gene-expression distribution** — a whole population of cells — at a
developmental stage the model has never seen. E10.5 is the public validation target; a hidden
**E12.5** determines the final ranking.

The score is not "how close is the average cell." It is a four-part question about a *population*:
which genes moved, in which direction, whether the set of cell states and their proportions is
right, and whether gene–gene co-regulation survived. A model that emits a mean profile, or a
tightly clustered blob, scores near zero on 50% of the panel.

**The success bar is explicit and unforgiving:** the normalisation maps the `copy_last` baseline
(reproduce the preceding stage verbatim) to a task score of **50**. Published reference dynamics
models — including a flow-matched neural ODE — *do not beat it*. Your job is to land meaningfully
above 50 on the hidden test stage. Treat "beat copy_last on a held-out extrapolation" as the only
milestone that counts.

**The core reframe, which should drive architecture:** E8.5 → E9.5 share 11 of 18/21 annotated cell
types; E9.5 → E10.5 share 13 of 23. So roughly **half the cells at the target stage have no
same-named predecessor**. They did not drift into place, they *differentiated*. Every cell-type-matched
baseline in the starter kit holds those cells fixed, and velocity extrapolation has nothing to move
them by. The modelling problem is **which cell states appear, and in what proportion** — not how far
existing ones travel. Any design that only transports existing cells forward is structurally capped.

---

## 2. Repository layout

```
.
├── CLAUDE.md                 this file
├── EXTERNAL_DATA.md          disclosure log — every external source, with stages
├── data/                     gitignored
│   ├── raw/                  E8.5_RNA.h5ad (571 MB), E9.5_RNA.h5ad (590 MB), E7.75
│   ├── panels/               T1__val.genes.txt, index.json, t1_composition.json
│   └── external/             anything ingested later, stage-filtered at load time
├── starter-kit/              official kit, gitignored, never edited
├── src/                      our code
├── configs/                  declarative run configs
├── results/                  experiments.csv, submission artefacts
└── reports/                  one markdown report per phase
```

**The official scorer is `veckit`**, a pip package: `pip install veckit`, source at
https://github.com/aristoteleo/veckit. It contains the reference implementations of `de_score`,
`de_genes`, `_signed_overlap`, `de_direction`, `mmd_unbiased`, `variogram_score`, `split_half`,
the baselines (`T1/baselines.py::copy_last`, `::pseudobulk_shift`), and `bake.py` which builds the
board bundles and the adversarial controls. The site also refers to a `score_h5ad.py` entry point;
if both exist, establish which is current and say so in `reports/phase1.md`.

CLI form:

```
veckit --task T1 --input pred.h5ad --target <truth>.h5ad --reference E9.5.h5ad
```

**Note what `--target` implies: local scoring requires a truth file you actually hold.** We do not
hold E10.5 or E12.5, so every local number comes from a proxy split we construct ourselves. This is
the constraint that makes Phase 2 load-bearing rather than optional.

E10.5 and E12.5 targets are **never distributed**. E12.5 inputs appear at the final phase without
labels; E10.5 answers are released when the final phase opens. You cannot score against either
locally.

Python env: `uv` or `conda`, everything pinned in a lockfile. Expect `scanpy`, `anndata`, `torch`,
`scikit-learn`, `POT`, and a `numpy` version check against the starter kit.

Timeline (from the live site, which supersedes the dates in the NeurIPS proposal PDF):
P2 development phase opened 2026-08-15 and is open now. **P3 test phase opens 2026-10-20**, when
validation answers are released and the test leaderboard opens. **Final submissions due 2026-12-02**,
official evaluation starts 2026-12-04, winners announced 2026-12-11. Budget backwards from 2 December.

If the starter kit or the data is not on disk, **stop and ask** — do not fabricate a schema.

---

## 3. Hard constraints (violating any of these invalidates the submission)

1. **Modality is whole-transcriptome dissociated single-cell RNA**: 32,285 genes, log1p-normalised,
   with cell-type annotation on the training stages. It is **not** the 500-gene MERFISH panel used
   by Tasks 2 and 3. Do not import a MERFISH assumption from the proposal PDF.
2. **No spatial coordinates** exist in this modality and none belong in a Task 1 submission.
3. **Output is a single AnnData `.h5ad`** whose `.X` is `[n_cells, 32285]`, float32, finite,
   non-negative. `var_names` must match the board panel **element by element, in order**. Take the
   panel from `T1__val.genes.txt`, **not from a data file** — no released stage is guaranteed to
   match a board. `--allow-reorder` makes the scorer reindex for you, but do not rely on it.
4. **`.X` must already be log-normalised.** This is *not* caught by validation: a raw count matrix
   is finite and non-negative, so it passes every check and is then scored as if it were on the log
   scale. Nothing will warn you; the score is simply wrong. Normalised-but-not-log-transformed is
   equally wrong.
5. **No cell-type labels in the submission.** `obs["celltype"]` is optional and *never read* — a
   frozen classifier fitted on the held-out truth types every cell from its own expression profile.
   You may use labels internally; never tune against your own labelling as if it were the scorer's.
6. **Cell count: minimum 1,000, board maximum 5,118 for T1:val.** Not a scored quantity, not
   compared to anything — it is a sample size. The heavy metrics take their own subsample anyway
   (MMD 2,000 cells; energy distance and variogram 1,500 each), so **a few thousand cells is the
   right answer** and going higher buys nothing. Check `index.json` for the live per-board limits.
   Note the scorer subsamples every stage to 10% before comparing: E9.5's 17,057 released cells
   become 1,706 on the scoring host. Your submission is not expected to mirror those counts.
7. **`obsm` is not read at all.** No coordinates in Task 1.
8. **External data is expressly encouraged for Task 1 — but the exclusion window is strict and
   is not the midpoint rule.** For the Task 1 extrapolation targets: **external data is excluded
   from after E9.5 up to and including E13.5**; E10.5 and E12.5 are excluded absolutely; anything
   **after E13.5** is permitted with disclosure. Data measured at exactly E9.5 is permitted (it is
   a released training stage). Data at exactly E13.5 is **not**. The stage label is not the test —
   a dataset labelled by somite count or Theiler stage that lands in the window is equally
   excluded, as is a model pre-trained on such data. Maintain `EXTERNAL_DATA.md` from day one with
   every dataset, accession, **the developmental stages it contains**, licence, and where it enters
   the pipeline. An undisclosed source is a violation whether or not it changed the result.
9. **Do not probe the leaderboard.** Validation returns a score, not answers, and submissions are
   limited. Model selection happens on local proxy splits; the leaderboard is confirmation only.
   At the final phase you get **at most two official test submissions**.

---

## 4. Splits

| Role | Stage | Notes |
|---|---|---|
| Train | E8.5, E9.5 | Real dissociated single cells, heart-centred, annotated. No E9.25 exists in this release. |
| Validation | E10.5 | Leaderboard-scored until the final phase; answers released when the final phase opens. |
| Test | E12.5 | Hidden, never distributed. Final ranking. |
| Outside the split | E7.75 | The one whole-embryo stage. Usable as background only — it is a different object (whole embryo, not heart dissection). Do not treat it as a fourth rung of the same ladder without an explicit domain-shift argument. |

Nothing is observed between E10.5 and E12.5. There is no bracketing stage. Interpolation has nothing
to lean on; this is pure extrapolation across a two-day gap during which the heart undergoes
looping and chamber morphogenesis.

---

## 5. Scoring — the shipped implementation is the contract

Let `X̂`, `X`, `X_ref` be predicted, observed, and reference log-normalised matrices over a common
ordered gene panel `G`. **For Task 1 the reference is the preceding developmental stage.**
`pred`, `true`, `ref` are the pseudobulk (cell-averaged) profiles.

**Weights (Task 1):** `de_score` 0.25 · `de_direction` 0.25 · `mmd_u` 0.30 · `variogram` 0.20.
No spatial/morphology term applies.

**`de_score` — DE gene recovery.** Ground-truth DE genes come from the *observed* data: two-sided
Mann–Whitney U of `X` against `X_ref`, Benjamini–Hochberg FDR at α = 0.05, minimum effect size 0.25
in mean log-expression → sets `U_true`, `D_true`. The prediction reports exactly
`n_u = |U_true|` up and `n_d = |D_true|` down genes, ranked by its own log fold change
`Δ_p = pred − ref`. With `ov = (|U_p ∩ U_true| + |D_p ∩ D_true|) / (n_u + n_d)`:

```
de_score = (ov − chance) / (1 − chance),
chance   = max( ov[Δ_p = +ref], ov[Δ_p = −ref] )
```

The chance correction is a null built from *exactly* the "submit a scaled copy of the reference"
attack, in both directions. 0 = no improvement over baseline expression level alone; 1 = exact
recovery; negative = wrong genes or wrong direction.

**`de_direction` — direction and ranking of change.** Rank-transform `Δ_p`, `Δ_t = true − ref`, and
`ref` into `r_p, r_t, r_r`; let `C` be their 3×3 Pearson correlation matrix. Then

```
de_direction = (C_pt − C_pr·C_tr) / sqrt( (1 − C_pr²)(1 − C_tr²) )
```

A partial correlation, because developmental responses are themselves correlated with baseline
expression level — an unadjusted correlation would credit you for reproducing that shared
dependence with no predictive signal.

**`mmd_u` — cell-state distribution (largest single weight, 30%).** Subsample both populations;
project onto a **30-component principal subspace estimated from the target alone**; RBF kernels
`k_s(x,y) = exp(−γ_s‖x−y‖²)` at `γ_s = s·γ_0`, `s ∈ {0.25, 0.5, 1, 2, 4}`, where `γ_0` is the
median-heuristic bandwidth **of the target**. Average the **unbiased** MMD² estimator (kernel
diagonal omitted) over the five bandwidths. The evaluation space is deliberately independent of
your submission — you cannot game the projection.

**`variogram` — gene–gene co-variation.** On 20,000 randomly sampled gene pairs `(i,j)`, with
`v_X(i,j) = E_cells |x_i − x_j|^0.5`:

```
variogram = E_(i,j) [ (v_pred(i,j) − v_true(i,j))² ]
```

This is the only term that constrains joint structure across genes. A prediction can match every
marginal and still fail here.

**Normalisation and aggregation.** Each metric is mapped to a skill score against the `copy_last`
baseline `m_base` and an attainable upper bound `m_ub` (estimated by scoring one half of the target
against the other half). With `d(m)` the distance from the upper bound oriented so larger = worse,
and `d_base = |m_ub − m_base|`:

```
skill(m) = min( d_base / (d_base + d(m)), 1 )  ∈ (0, 1]
S_task   = 100 × Σ_g w_g Σ_{m∈g} w_{m|g} · skill(m)
```

Upper bound → 1, baseline → 0.5, degrading → 0 asymptotically. **`S = 50` is `copy_last`.**
An undefined metric scores `skill = 0`, not omitted.

**The reference implementation ships with the starter kit** — `common/core_metrics.py::de_score`,
`de_direction`, `mmd_unbiased`, `variogram_score` — and `python score_h5ad.py --task T1 --input
pred.h5ad` runs the full panel locally and prints JSON. **Read that code before writing your own.**
The definitions above are for understanding what each term rewards; the shipped code is the
contract.

**Published calibration anchors for the T1:val board** (floor = `copy_last`), also in
`index.json` under `anchors`. With these you can reproduce a leaderboard score from a raw metric
value and check any local scorer against theirs:

| Metric | Floor | Ceiling |
|---|---|---|
| `de_score` | 0 | 0.8464 |
| `de_direction` | 0 | 0.7901 |
| `mmd_u` | 0.08359 | 0.00406 |
| `variogram` | 0.005219 | 0.000158 |

Nothing about the hidden test board is published.

**Deliverable:** a thin wrapper around the shipped metrics for fast batch iteration, plus unit
tests covering: exact-target-recovery, `copy_last`, a mean-collapsed prediction, and a scaled copy
of the reference. Assert agreement with `score_h5ad.py` to floating-point tolerance on a fixture,
and **treat any disagreement as a bug in your code until proven otherwise**.

One calibration fact worth internalising: on a naive overlap, a submission that had learned nothing
except which genes are highly expressed scored **0.61**, against **0.17** for the best real
baseline. That gap is exactly what the chance correction removes.

---

## 6. Baselines to reproduce before writing anything new

From the starter kit — these are transparent reference points, not upper bounds:

- `copy_last` — reproduce the preceding stage verbatim. `X̂ = X_ref`. **This is the floor and the
  bar.**
- `pseudobulk_shift` — per-cell-type constant velocity from the two most recent training stages,
  applied to real cells of the most recent stage: `Δ_k = X̄_k^(S−1) − X̄_k^(S−2)`,
  `x̂_i = max(0, x_i^(S−1) + Δ_{y_i})`, with `Δ_k = 0` for types absent at `S−2`. Preserves
  within-type heterogeneity rather than collapsing to a fitted mean.
- `shift_ode` — kept because its minimiser is that same constant shift; the equivalence should stay
  visible in your results table.
- `neural_ode` — autonomous field in gene space, flow-matched on a global entropic-OT coupling.
- `dynode_flow` — external flow-matching reference, unmodified.

**Known result you must not spend a week rediscovering:** the neural ODE does not beat the one-line
constant shift, and fitting the same extrapolation off the evaluation stages lands *every* setting
below `copy_last`. That is the published finding, not a bug in the starter kit. If your first
instinct is "train a neural ODE on E8.5→E9.5 and integrate to E12.5," you are reproducing a known
negative result.

### Where the score actually sits — read this before choosing an architecture

The organisers state the decomposition outright: `copy_last` submits **genuine single cells with
genuine gene–gene structure**, so it is very hard to beat on the distributional metrics, while
`pseudobulk_shift` wins on the expression-change metrics by actually moving in the right direction.

Map that onto the weights: `mmd_u` (30%) + `variogram` (20%) = **50% of the score is population
structure, where `copy_last` is already strong**. `de_score` (25%) + `de_direction` (25%) = **50%
is change, where doing nothing scores zero by construction**.

The implication is sharp and should constrain every design decision: **carry real cells forward so
the distributional half stays near `copy_last`, and win on the change half.** A generative model
that synthesises cells from scratch has to re-earn 50% of the score that shifting real cells gets
for free. That is a large bet and it needs to be justified against a measured alternative, not
assumed.

### Two cheap experiments to run before anything ambitious

1. **`damp`.** `pseudobulk_shift` takes a damping factor; the tutorial notes the first correction to
   try is `damp = Δt_target / Δt_observed`. The observed interval is E8.5→E9.5 = 1.0 day. The
   validation target is 1.0 day out (`damp ≈ 1`), the **test target is 3.0 days out (`damp ≈ 3`)**.
   The shipped baseline implicitly assumes `damp = 1` on both. Sweep it on a proxy split.
2. **Cell-type granularity.** `Δ_k` is computed per annotated type; finer or coarser types change
   what the shift can express. Cheap to sweep, and it interacts with the harmonisation problem in
   section 8.

### Adversarial controls are on the public board

`ctrl_one_cell` (one average cell tiled), `ctrl_scale_ref` (reference × 2.0) and `ctrl_shrink_ref`
(reference × 0.99) are scored on every run and left visible. A control scoring above the floor is
treated as a bug in the metrics. Two consequences: the traps in section 9 are not hypothetical, they
are instrumented; and if you find yourself reasoning toward something that resembles one of these,
you have found the exploit the organisers already closed.

---

## 7. The measurement problem — solve this before modelling

With only two public stages you have **no honest local extrapolation split**. E8.5 → E9.5 gives a
single-input, one-step problem that is degenerate and will not rank methods the way E9.5 → E12.5
does. Selecting on it will mislead you.

**Build a surrogate stage ladder from external public mouse data — inside the permitted windows
only.** This is the constraint that shapes the whole phase: you may use stages **at or before
E9.5**, and stages **strictly after E13.5**. Everything from just after E9.5 through E13.5
inclusive is off limits, by any route, under any staging vocabulary.

So there are exactly two legal places to build a surrogate extrapolation ladder:

- **The early window (≤ E9.5).** Dense in most atlases; lets you construct train `{t−k…t−1}` →
  predict `t` with a held-out gap. Caveat: gastrulation-window composition turns over faster than
  the E9.5→E12.5 interval, so it is not a like-for-like difficulty analogue.
- **The late window (> E13.5).** E14.5 → E17.5 is closer in character to the real jump — organs
  are forming rather than germ layers — and gives you a genuine multi-day extrapolation with
  differentiating populations. Verify stage labelling carefully; E13.5 itself is excluded.

Candidate sources, each of which must be **stage-filtered before ingestion, not after**:

- Qiu et al. 2024, *Nature* 626:1084 — mouse prenatal time-lapse, gastrula to birth, 12.4M cells,
  83 embryos. The densest ladder available, but it **contains the excluded window**. Ingest only
  the permitted stages, and write the filter as a hard assertion in the loader, not a notebook step.
- TOME (Qiu et al. 2022, *Nat Genet* 54:328) — stage-to-stage transition structure, early window.
- CZI CELLxGENE mouse developmental collections — check stage metadata per dataset.
- Avoid MOSTA (E9.5–E16.5) for Task 1: it is a different modality *and* sits largely inside the
  excluded window.

Before ingesting anything, write `src/data/stage_policy.py` with a single function that takes a
declared stage (including somite-count and Theiler-stage inputs) and returns permitted /
excluded / needs-human-review. Every external loader calls it. Log every decision. This is a
compliance artefact as much as an engineering one — an undisclosed or non-compliant source
invalidates the entry regardless of whether it helped.

**Required output of this phase:** a held-out protocol where you train on stages `{t−k … t−1}` and
predict stage `t` across a multi-stage gap, scored with the shipped metric panel and reported as
the same four metrics plus `S_task`. Every subsequent modelling claim must be backed by this
protocol, not by a leaderboard number and not by eyeballing a UMAP.

Also verify and document: gene-panel intersection and ID mapping between external data and the
released panel; normalisation differences; heart-dissection vs whole-embryo compositional shift.
Domain gap here is a real risk — quantify it, don't wave at it.

---

## 8. Modelling directions, in priority order

Test these against the surrogate ladder. Do not build all of them; kill fast.

**A. Composition + conditional generation (primary hypothesis).**
Decompose the target population into *which states are present in what proportion* and *what each
state looks like*.
- Fit a compositional trajectory over cell types across the external stage ladder — including
  emergence of types absent at earlier stages — and extrapolate it to the target stage. This is the
  half of the problem the starter-kit baselines structurally cannot address.
- Generate per-state expression with a generator **conditioned on continuous stage and cell state**
  — flow matching or diffusion in a reduced space, decoded to the full panel. Stage conditioning
  makes the field **non-autonomous**, which the reference neural ODE is not.
- Sample the final population by mixing generated states at the extrapolated proportions.

**B. Unbalanced / growth-aware transport.** If you stay in a transport framework, the coupling must
allow **mass creation and destruction** (unbalanced OT, or an explicit growth-rate field). Balanced
OT cannot express "this population appeared." This is the minimal fix to the known negative result.

**C. Structured strong baseline (build early, it may be competitive).** Take real E9.5 cells; apply
a per-type shift estimated across a permitted external ladder rather than from two stages; reweight
type proportions to an extrapolated composition; and **synthesise cells for newly-appearing types**.
Note the constraint: you cannot source those cells from an atlas *at* the target stage — that stage
is excluded. They must be extrapolated from permitted stages or generated. Cheap, interpretable,
preserves gene–gene structure by construction (helps `variogram`), and directly attacks the
appearance problem. Score it before investing in A.

**Two data facts that matter for any composition model:**

- `t1_composition.json` publishes the cell-type composition of the *released* stages, so the
  observation process is visible rather than something you reconstruct. Nothing in it describes
  E10.5 or E12.5.
- **The annotation vocabulary is not harmonised across stages.** `Endocardium` at E9.5 appears as
  `Endocardium-1` and `Endocardium-2` later; a label missing from a stage may be present under a
  different name rather than biologically absent. **Comparing label sets across stages overstates
  how much has changed.** Build your own harmonised mapping (marker-based or embedding-based, not
  string-based) before drawing any conclusion about type turnover, and treat the headline
  "two thirds of cells are new types" as an upper bound on real biological novelty.
- The Task 1 stages are a **heart-centred dissection**, not whole embryos and not isolated hearts,
  and how much surrounding tissue comes along **differs by stage**: Neural Tube is 4.6% of E8.5 and
  0% of E9.5; paraxial and extra-embryonic mesoderm behave similarly. The target carries both the
  biology and the dissection protocol. You are not asked to predict the protocol, but you are
  scored against a sample that reflects it — so a composition model fit purely on developmental
  logic will be systematically off.

**D. Co-variation preservation.** Whatever generates expression, avoid per-gene independent
sampling — it can match marginals and still fail `variogram`. Prefer methods that carry real cells
forward, or low-rank + structured residual, and monitor `variogram` as a first-class objective, not
an afterthought.

---

## 9. Pitfalls (each of these is a scored failure mode, not a style note)

- **Mode collapse.** A tightly clustered prediction passes a mean-based sanity check and then fails
  the distributional terms. Add a diagnostic that would catch it *before* scoring: effective
  support, per-type entropy, nearest-neighbour distance distribution vs target.
- **Scaled copy of the reference.** Both DE metrics build their null from exactly this, in both
  directions. It scores ~0 by design.
- **Assuming the 500-gene MERFISH panel.** Task 1 is whole-transcriptome.
- **Shipping or tuning to cell-type labels.** Labels are never part of a submission; the frozen
  probe re-assigns them.
- **Selecting on the leaderboard.** Limited submissions, and validation ≠ test stage.
- **Treating E7.75 as a training rung.** Different object.

---

## 10. Phased plan, with gates

Do not advance past a gate without the artefact.

| Phase | Work | Gate |
|---|---|---|
| 0 | Repo scaffold, pinned env, data audit (shapes, gene order, normalisation, per-stage cell-type tables and overlaps) | `reports/data_audit.md` with the type-overlap counts reproduced from the actual data |
| 1 | Read `starter-kit/common/core_metrics.py`; wrap it for fast batch scoring; add tests; reproduce all five starter-kit baselines | Results table; your wrapper agrees with `score_h5ad.py` to fp tolerance |
| 2 | External data ingestion + surrogate stage ladder + held-out extrapolation protocol | A protocol that ranks the known baselines in the published order (neural ODE ≤ constant shift ≤ … ) — if it doesn't, the protocol is wrong |
| 3 | Structured strong baseline (direction C) | Beats `copy_last` on the surrogate protocol, or a written explanation of why not |
| 4 | Composition model (A1) and conditional generator (A2), separately ablated | Each component's marginal contribution measured |
| 5 | Selection, ensembling, submission packaging, format check, `EXTERNAL_DATA.md`, method write-up | Format check passes; two candidate submissions ranked by local protocol with a documented tie-break |

---

## 11. Working practices

- Every experiment appends one row to `results/experiments.csv`: run id, git sha, config hash,
  protocol, four raw metrics, four skill scores, `S_task`, wall time. No claim without a row.
- Seed everything; report variance across ≥3 seeds before declaring an improvement. Differences
  smaller than seed noise are not improvements.
- Keep configs declarative (`configs/*.yaml`); no hyperparameters buried in code.
- After each phase, write a short `reports/phaseN.md`: what you tried, what the numbers were, what
  you're killing, what's next.
- **Stop and ask me** rather than guessing about: the exact submission file contract, the
  normalisation convention, whether a specific external dataset is permitted, and any point where
  the starter kit contradicts this brief. The starter kit and
  `virtualembryo.ai/challenge/evaluation?task=1` are authoritative over this document.
- Two unresolved facts to settle in Phase 1 and record in `reports/phase1.md`: (a) each task
  directory ships both `metrics.py` and `metrics_v2.py` - establish which one the leaderboard runs;
  (b) `T1/baselines.py` is cited by the baselines page but is **not** in the public repo, so
  `copy_last` and `pseudobulk_shift` must be reimplemented from the published descriptions and the
  tutorial notebook. Neither is more than a few lines, but they are ours to get right.

---

## 12. Definition of done

A reproducible pipeline that (a) scores locally in agreement with `score_h5ad.py`, (b) reproduces the starter-kit baselines, (c) beats `copy_last` by a margin exceeding
seed noise on a held-out extrapolation protocol with a realistic stage gap, (d) emits a
format-valid AnnData submission for E10.5 and E12.5, and (e) ships a method write-up with full
external-data disclosure.

---

## Open questions — ask me these in the first session

1. Is the starter kit downloaded, and are the training stages on disk?
2. What compute is available (GPU model/count, wall-clock budget)?
3. Human Team track or Agent Team track? (The Agent track requires sharing the agent system and the
   complete evolutionary trace before prizes are decided — that changes how the repo should be
   instrumented from day one.)
4. Is external data actually desired, or do you want a clean two-stage-only solution? External data
   is permitted and encouraged, but it carries a strict exclusion window and a disclosure
   obligation. This is the single biggest fork in the plan.
5. Is Task 1 the whole scope, or should the repo be structured so Tasks 2 and 3 slot in later?
6. Registered on virtualembryo.ai and downloaded the training stages yet? Nothing in phase 0 can
   start without `E8.5_RNA.h5ad`, `E9.5_RNA.h5ad`, and `T1__val.genes.txt`.
