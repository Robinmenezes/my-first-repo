# QUANT.md — Quantitative Modeling Playbook

A single-file protocol for building, evaluating, and re-evaluating quantitative
models and algorithms in this repo. Loaded by the `/quant` slash command.

---

## 0. Entrypoint

Given a problem statement:

1. Classify it into a case bucket via **§2 Problem Taxonomy**.
2. Pick a starting protocol from **§5 Protocols**.
3. Run **§6 The Loop**.
4. Pass **§8 Validation** gates before claiming a model is done.
5. Ship the artifact contract in **§9 Output Spec**.
6. Never violate **§10 Hard Stops**.

If the case doesn't fit cleanly in §2, frame it explicitly as a hybrid and pick
the dominant bucket — don't invent a new one without updating this file.

---

## 1. Mission & Success Criteria

A model is "done" when **all** of the following hold:

| Criterion | Default threshold | How to override |
|---|---|---|
| Beats §4 baseline | by ≥ 5% on the chosen metric | State per-case in problem brief |
| Calibrated | Brier / reliability check passes | N/A for regression with proper scoring rule |
| Out-of-sample stable | walk-forward / held-out fold within 1σ of in-sample | Document if data is i.i.d. and explain |
| Reproducible | seed + data hash + env captured | Mandatory |
| Decision-ready | output maps cleanly to a downstream action | State the action in the brief |

**Stop criteria — ship when:**
- Baseline already passes all gates above.
- Diminishing returns: 3 consecutive iterations yield < 1% improvement.
- Validation gates pass and the marginal complexity isn't paying for itself.

**Stop criteria — abandon when:**
- After 5 iterations no method beats the §4 baseline on the validation set.
- Data quality issues (§3) can't be resolved at source.

---

## 2. Problem Taxonomy

Every incoming case lands in exactly one of these buckets:

| Code | Type | Examples |
|---|---|---|
| **A** | Point regression | House price, demand level, LTV |
| **B** | Classification | Default / no-default, churn, fraud |
| **C** | Time-series forecast | Daily revenue, weekly demand, vol forecasting |
| **D** | Stochastic optimization | Portfolio allocation, inventory under uncertainty, staffing |
| **E** | Causal estimation | Treatment effect, uplift modeling, policy evaluation |
| **F** | Structural / SDE modeling | Option pricing, hazard models, physics-style dynamics |
| **G** | Sequential decision (RL / bandit) | Adaptive allocation, online pricing, treatment sequencing |

---

## 3. Data Registry

Source of truth: `data/registry.yaml` (create when first dataset is added).

Each dataset entry must include:
- `name`, `path`, `schema` (column → dtype), `pii: true|false`
- `license` and `refresh_cadence`
- `quality_notes` (known issues, leakage risks, missingness)
- `splits` (train/val/test definition, including time boundaries for case C/G)

**Data quality protocol — run before any model:**
1. Schema check: columns, dtypes, nullability.
2. Leakage scan: any feature that includes target-time information.
3. Drift baseline: PSI / KS between train and val.
4. Missingness map and decision (impute / drop / model explicitly).
5. Class balance / target distribution snapshot.

Synthetic data generators live in `data/synthetic/` with their generation seeds
checked in.

---

## 4. Method Catalog

Per case type. Each case has a **mandatory baseline** and a **stretch ladder**.
Reference implementations under `src/methods/<case>/`.

| Case | Mandatory baseline | Stretch ladder |
|---|---|---|
| A | OLS with regularization (ridge / lasso) | GBM (LightGBM/XGBoost) → MLP → stacked ensemble |
| B | Logistic regression | GBM → calibrated GBM → MLP → stacked |
| C | Seasonal naive + ARIMA | Prophet → state-space → LightGBM with lag features → N-BEATS / TFT |
| D | LP / convex (CVXPY) | Stochastic LP → SDDP → MPC → HJB / dynamic programming |
| E | OLS with controls + IPW | Doubly robust → causal forests → IV / DiD if design supports |
| F | Closed-form where it exists (e.g. Black-Scholes for vanilla options) | Monte Carlo → SDE fit (MLE / Kalman) → PDE solver |
| G | ε-greedy bandit | Thompson sampling → contextual bandit → tabular Q-learning → deep RL (last resort) |

Baseline must run end-to-end before any stretch method is implemented.

---

## 5. Protocols (decision tree)

One-liner per branch. If a branch isn't listed, default to the case baseline.

```
A + tabular + n < 10k          → ridge OLS, then GBM
A + tabular + n ≥ 10k          → GBM, then stacked
A + text / image features      → GBM on embeddings, then fine-tune

B + tabular + balanced         → logistic, then GBM with calibration
B + imbalanced (< 5% positive) → logistic + class weights → GBM + calibration + threshold tuning

C + < 3 seasons of data        → seasonal naive only — do not fit complex models
C + ≥ 3 seasons + univariate   → ARIMA / ETS, then state-space
C + multivariate / exogenous   → LightGBM with lag features, then TFT

D + finite horizon + linear    → LP (CVXPY)
D + finite horizon + nonlinear → SLSQP / interior point
D + uncertainty + tractable    → stochastic LP / scenarios
D + uncertainty + intractable  → MPC with rolling re-optimization, then HJB

E + RCT data                   → diff-in-means + CIs, done
E + observational + good controls → DR estimator + sensitivity analysis
E + observational + IV         → 2SLS with weak-IV diagnostics

F + vanilla derivative         → closed form
F + path-dependent / exotic    → Monte Carlo with control variates
F + need calibration to market → SDE fit (MLE / Kalman) then MC

G + offline data only          → IPS / DR off-policy evaluation
G + can experiment online      → contextual Thompson sampling
G + long horizon + simulator   → policy iteration / actor-critic
```

---

## 6. The Loop

Each iteration logs to `notebooks/<case>-<YYYYMMDD>.ipynb` (see §9).

1. **Frame** — restate problem, metric, baseline target, decision the model serves.
2. **Baseline** — fit §4 baseline end-to-end. If it passes §8, **stop and ship**.
3. **Diagnose** — residual plots, calibration, error-mode catalogue, subgroup
   breakdowns. Identify the single biggest failure mode.
4. **Iterate** — propose one method targeting that failure mode (cite §4 / §7
   for justification), implement, evaluate. **One change at a time.**
5. **Validate** — run §8 gates. If any fail, do not advance.
6. **Ship** — emit the §9 artifact contract.

If step 4 has produced no improvement for 3 consecutive iterations, escalate to
§7 (auto-research).

---

## 7. Auto-Research Triggers

Research fires on **concrete conditions**, not vibes:

| Trigger | Action |
|---|---|
| Baseline misses §1 threshold by > 10% | WebSearch `"<case-type> SOTA <current year>"`; read top 3 abstracts; pick the most relevant for §4 stretch ladder |
| 3 consecutive iterations with < 1% gain | WebSearch `"<failure-mode> <case-type>"`; cite at least one paper before next iteration |
| Residuals show non-stationarity (case C/G) | Search `regime detection`, `changepoint`, `online learning`; consider sliding-window training |
| Causal estimate (case E) sensitive to one control | Search `omitted variable bias <domain>`; run sensitivity analysis (Rosenbaum / E-value) |
| New method proposed | Require **one citation + one reference implementation** before writing code |

Document each research action in the notebook's "research log" section with: query, top finding, decision (apply / discard / defer).

---

## 8. Validation & Re-evaluation

**Required gates per case** (in addition to §1 criteria):

| Case | Required |
|---|---|
| A | k-fold CV (k=5), residual plot, subgroup error breakdown |
| B | Stratified k-fold, calibration plot, PR curve at relevant operating point |
| C | Walk-forward CV (no peeking), seasonality check, error vs horizon |
| D | Sensitivity to top 3 input parameters, stress test under perturbed scenarios |
| E | Robustness check (alternative estimator), placebo test, sensitivity to unmeasured confounders |
| F | Calibration to held-out market data, MC convergence diagnostics, Greeks sanity |
| G | Off-policy evaluation, regret curve, exploration-exploitation diagnostics |

**Re-evaluation triggers (after ship):**
- Performance drop > 2σ on rolling window → retrain.
- Input drift (PSI > 0.2 on any top-10 feature) → investigate, possibly retrain.
- Concept drift (target distribution shift) → retrain + revisit §5 choice.
- New data source added → re-run §3 quality protocol, then re-evaluate.

**Champion-challenger:** new model must beat champion on **two** consecutive
validation windows before replacing.

---

## 9. Output Spec

Every successful loop iteration produces:

| Artifact | Location | Contents |
|---|---|---|
| Code | `src/models/<case>/` | Pure functions for `fit`, `predict`, `evaluate` |
| Notebook | `notebooks/<case>-<YYYYMMDD>.ipynb` | Sections: problem, data, baseline, model, validation, decision, research log |
| Report | `docs/<case>-report.md` | Executive summary + reproducibility appendix |
| Reproducibility | embedded in report | Seed, data hash, package versions, git SHA |
| API (optional) | `src/api/<case>.py` | FastAPI app with `/predict` and `/healthz` |

The report's executive summary must answer:
- What problem? What metric? What did we ship?
- What's the model's error mode the reader should know about?
- When should it be retrained?

---

## 10. Hard Stops (anti-patterns)

- **Never tune on the test set.** Test is for the final number; tune on val.
- **Never ensemble before fixing leakage.** Ensembling hides root causes.
- **Never ship before the case's §8 gates pass.** No exceptions for time pressure.
- **Never trust a model that beats baseline by > 2×** without explicit
  root-cause analysis. Almost always leakage or a wrong metric.
- **Never skip the baseline.** The baseline is the success criterion; without
  it, "improvement" has no meaning.
- **Never auto-deploy a model that hasn't been live in shadow mode** for the
  re-evaluation window in §8.

---

## 11. Reference Cases

Add a row here every time a case is completed end-to-end:

| Case | Bucket | Notebook | Report | Notes |
|---|---|---|---|---|
| _(none yet)_ | | | | |
