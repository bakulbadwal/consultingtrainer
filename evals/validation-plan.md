# Calibration Plan: LLM Judge for Profit Diagnosis Game (Accuracy + Efficiency)

**Context note:** No real candidate transcripts exist yet, so this plan front-loads a bootstrap-labeling step (synthetic candidate transcripts spanning known failure archetypes) before the standard train/dev/test calibration flow can run. It follows the `validate-evaluator` methodology (data splits, TPR/TNR, Rogan-Gladen correction) but adapts it for: (a) a 0–10 ordinal accuracy rubric rather than binary Pass/Fail, and (b) a 0–10 continuous efficiency score.

---

## 1. How many hand-labeled cases, and why

**Target before trusting the judge: ~120 hand-graded candidate transcripts** (not scenarios — actual attempt transcripts, multiple per scenario), not the ~100 the base skill recommends for a simple binary judge. Reasons this needs to be larger/denser than the binary default:

- The accuracy axis is an **11-point ordinal rubric with 6 defined bands** (0–1, 2–3, 4–5, 6–7, 8–9, 10), not Pass/Fail. A single binary TPR/TNR check would hide miscalibration at the boundaries the rubric is most likely to blur — especially **2–3 vs 4–5** (defensible-misread wrong answer vs. correct-by-elimination) and **6–7 vs 8–9** (imprecise-but-right vs. fully-supported-and-ruled-out), which require the judge to read reasoning quality, not just the final label.
- The prompt explicitly warns against **cross-axis contamination** (correct answer via wasteful path; wrong answer via efficient-looking path). You cannot verify a judge resists this halo effect from naturally-occurring transcripts alone — you need to *deliberately construct* decorrelated cases (right+wasteful, wrong+efficient) and confirm the judge scores each axis independently.
- Four root-cause categories (price erosion, share/volume loss, mix shift, cost inflation) each come with a built-in "tempting distractor," per the golden set's design. Judge failure modes are likely to concentrate on specific distractor patterns, so each category needs its own adequate sample, not just a pooled total.

**Composition of the 120:**
- Stratify across **all 4 root-cause categories** (~30 transcripts each).
- Within each category, stratify across the **6 accuracy bands**, oversampling the boundary-adjacent bands (aim ~15–20 per band rather than a flat ~10, with extra density at 2–3/4–5 and 6–7/8–9).
- Independently stratify a subset (~20–25 transcripts) as **deliberately axis-decorrelated stress cases**: correct diagnosis + needlessly long/redundant pull sequence, and wrong diagnosis + a clean, minimal, well-reasoned (but ultimately misinterpreted) sequence.
- Since no real transcripts exist, generate these as **synthetic candidate transcripts** against the golden-set scenarios (multiple "personas" per scenario: expert, elimination-guesser, off-by-one-pull, fabricator, efficient-but-wrong, thorough-but-wasteful), then have a domain expert (someone who could actually referee the case-interview logic — not an outsourced rater) hand-grade each on both axes using the same rubric the judge sees.

Below ~60–70 labeled cases, confidence intervals on TPR/TNR and on the efficiency correlation will be too wide to certify automation; 120 is the point where per-category, per-band splits stay statistically usable through train/dev/test.

## 2. Train / dev / test split

| Split | Size (of 120) | Purpose | Rules |
|---|---|---|---|
| **Train** | ~18 (15%) | Few-shot examples embedded in the judge prompt | Pick the *clearest* cases: at least one clean example per accuracy band, and — critically — at least one explicit correct+wasteful and one wrong+efficient example, since the prompt's anti-contamination instruction needs a concrete anchor. |
| **Dev** | ~54 (45%) | Iterative judge-prompt refinement | Never appears in the prompt. Re-run after every prompt edit. |
| **Test** | ~48 (40%) | Final unbiased measurement | Untouched until judge stabilizes on dev. Run exactly once. |

Stratify **all three splits** by category × accuracy band × the decorrelated-stress-case flag, so no split is accidentally easier or skewed toward one root-cause category. If multiple candidate transcripts share the same underlying scenario, keep all transcripts for a given scenario_id inside a single split — otherwise the judge (and you, while eyeballing disagreements) can pattern-match the scenario's answer key across splits, leaking signal from train/dev into test.

## 3. TPR/TNR thresholds for "automate" vs. "still needs human review" (accuracy axis)

The accuracy rubric has one clearly load-bearing binary cut and one stricter one — calibrate both:

- **Primary cut ("did the candidate get the right root cause"): score ≥4 = Pass, ≤3 = Fail.** This is the decision most likely to gate a real pass/fail judgment on candidates, so it's the one to hold to the strict bar:
  - **Trustworthy enough to fully automate:** TPR > 90% AND TNR > 90% on the held-out test set.
  - **Minimum acceptable with light-touch spot-checking:** TPR > 80% AND TNR > 80%.
  - **Below 80% on either metric:** still needs human review on every case — do not automate.
- **Secondary cut ("fully rigorous": score ≥8 vs <8)** — this is a finer distinction (rules out alternatives, quantifies magnitude) and LLM judges tend to be less reliable here. Apply the same 90/90 target, but if it plateaus at 80–85%, that's an acceptable reason to route only *this* boundary (scores landing in 6–9) to human secondary review while still automating the 0–5 vs 6+ decision.
- **Asymmetric error cost:** in this rubric, a **false Pass at the primary cut** (judge says correct root cause when a human says wrong) is more costly than a false Fail, because band 0–1 explicitly includes "misreads data the candidate itself pulled" — a case a human would clearly flag as wrong. Weight TNR slightly higher than TPR when deciding whether to automate; if forced to choose which metric to improve first, prioritize TNR.
- **Practical automation mode:** rather than a hard automate/don't-automate switch, use score-band **confidence triage** — auto-accept judge scores that land solidly inside a band (0–1, 4–5, 8–9 core) and route boundary scores (2–3/4–5 edge, 6–7/8–9 edge) to human review, since that's empirically where disagreements will cluster.

## 4. Efficiency axis: adapting the TPR/TNR framing

Efficiency is a **continuous 0–10 score**, so raw binary TPR/TNR doesn't apply directly. Replace it with:

- **Tolerance-band agreement:** define "judge agrees with human" as within **±1 point** (or ±1.5 for a looser bar) of the human-assigned efficiency score. Report % of dev/test cases within tolerance — target **>85–90%** before treating the judge as automatable on this axis, same spirit as the 90% TPR/TNR bar.
- **Rank correlation (Spearman's ρ or Kendall's τ)** between judge and human efficiency scores across the full dev/test set. Efficiency is most likely used comparatively (ranking candidates, or trend-tracking a repeat candidate over time) rather than as an absolute pass/fail cutoff, so relative ordering matters more than exact-value matching. Target **ρ > 0.8**.
- **MAE / RMSE** in points on the 0–10 scale as a plain-language summary stat (target MAE < 1.0–1.5).
- **If a categorical automate/don't-automate decision is still needed** (e.g., a downstream product bucket like "efficient / adequate / wasteful"), collapse both judge and human scores into 3 tiers (8–10 / 4–7 / 0–3) and compute a 3-class confusion matrix, then derive per-tier recall/specificity the same way TPR/TNR is derived for the binary case — this gives you a familiar go/no-go readout without pretending the underlying score is binary.
- **Explicit halo-effect check:** using the axis-decorrelated stress cases from the labeled set (§1), verify the judge's efficiency score on *correct-but-wasteful* transcripts isn't inflated relative to human labels, and its accuracy score on *wrong-but-efficient-looking* transcripts isn't deflated or inflated by path quality. Report these as a separate slice, not blended into the aggregate MAE/ρ — a judge can pass aggregate efficiency calibration while still failing this specific contamination check, and the prompt explicitly calls this out as the axis's main risk.

## 5. Bias-correction steps if the judge skews systematically harsh or lenient

**Accuracy axis (binary cut), for correcting an aggregate pass-rate estimate on unlabeled production data:**
- Apply Rogan-Gladen: `theta_hat = (p_obs + TNR − 1) / (TPR + TNR − 1)`, using TPR/TNR measured once on the held-out test set, clipped to [0,1].
- Report a bootstrap 95% CI (per the skill's `bootstrap_ci` / `judgy` approach) alongside the point estimate — with a 48-case test set, CIs will be meaningfully wide, so don't present the corrected rate without a range.
- Disaggregate TPR/TNR **by root-cause category** before applying a single global correction factor. Given the golden set's tempting-distractor design, it's plausible the judge is harsher (or more lenient) specifically on categories where the distractor is strong (e.g., mix shift being confused with cost inflation) — a category-specific correction factor may be needed rather than one global number.

**Efficiency axis (continuous), where Rogan-Gladen doesn't apply directly:**
- Compute the **mean signed error** (judge score − human score) on dev/test. A consistent non-zero offset (e.g., judge runs +1.2 harsh on average) indicates a simple additive bias — correct with `corrected_score = raw_score − offset`, fit on dev, validated (not re-fit) on test.
- If the bias isn't uniform across the range (common LLM-judge failure: compression toward the middle, i.e. harsh on genuinely efficient paths and lenient on wasteful ones), an additive offset won't fix it — fit an **isotonic regression** mapping raw judge scores to human scores on dev, and apply that monotonic mapping to production scores instead.
- Re-check the corrected mapping doesn't just relocate the halo-effect problem from §4 — validate the correction separately on the decorrelated stress-case slice.

**For both axes:**
- **Fix the prompt before fixing the output statistically.** If disagreement inspection (dev-set step) shows a legible pattern — e.g., judge consistently over-credits "ruled out alternatives" when the candidate merely asserts rather than cites evidence — that's a rubric-wording or few-shot-example fix, which should be tried first. Statistical correction (Rogan-Gladen / isotonic mapping) is a stopgap for whatever residual bias survives prompt iteration, not a substitute for it.
- **Pin the exact judge model snapshot** (dated ID, not a floating alias) once calibrated, and treat any change to the model, the prompt, or the case-setup/menu format as a trigger to re-run this full calibration — the TPR/TNR and bias-correction numbers are only valid for the exact judge configuration they were measured against.
