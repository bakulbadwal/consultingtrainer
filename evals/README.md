# Evals — Profit Diagnosis Game

An evaluation harness for the trainer's **Profit Diagnosis Game**, the drill with the most
interesting scoring problem: a candidate gets a randomized business situation and a *fixed budget
of data pulls*, and is scored on two axes that can move independently — did they reach the correct
root cause (**accuracy**), and how close was their sequence of pulls to the minimal sufficient path
(**path efficiency**).

Path efficiency is why this needs an LLM judge rather than a string match. Whether a given data pull
was a sound next step depends on what the previous pulls had already revealed — that's a judgment
call about reasoning, not a lookup.

## What's here

| File | |
|---|---|
| `golden-set.json` | 12 scenarios across SaaS, freight, apparel, dental, telecom, CPG, industrial distribution, beverage, HVAC, steel, retail, and consulting. Each has the correct root cause, the optimal pull sequence, and — deliberately — a *plausible wrong answer* plus why it's tempting. |
| `judge-prompt.md` | The judge: 0–10 rubrics on both axes, explicit anti-halo-effect instructions (a right answer via a wasteful path is not evidence of efficiency, and vice versa), and two fully worked grading examples. |
| `validation-plan.md` | How the judge would be calibrated: ~120 labeled transcripts, train/dev/test splits, TPR/TNR thresholds for automation, and Rogan-Gladen bias correction if it skews harsh or lenient. |

## Status — designed, not yet validated

**No transcripts have been graded and the judge has not been calibrated.** There are zero human
labels, so its agreement rate with human judgment is unmeasured — which by the validation plan's
own standard means its scores are not yet trustworthy.

This is deliberate scoping, not an oversight. The design is the part that generalizes; the
calibration is mechanical but slow (the labeling is irreducibly human).

**Accurate description of this work:** a designed eval harness — golden set, two-axis LLM judge,
and a calibration plan.
**Not accurate yet:** "validated," "calibrated," or any claimed agreement rate.

## To finish it

1. Generate ~25 synthetic candidate transcripts across the 12 scenarios, spanning personas
   (expert · elimination-guesser · off-by-one-pull · efficient-but-wrong · thorough-but-wasteful)
2. Hand-grade them on both axes — the irreducibly human step
3. Run the judge on the same transcripts
4. Measure: TPR/TNR on the accuracy cut, ±1 tolerance-band agreement and Spearman ρ on efficiency
5. Write up the disagreements — where the judge and a human diverged is the most informative output

Even 25 labels (vs. the plan's ideal 120) yields a real, defensible number.

## Method note

Built with a three-agent parallel workflow: golden-set generation and judge-prompt design ran
concurrently (independent of each other), then the validation plan ran once both completed, since
it depends on both. Tooling: the `evals-skills` plugin (`write-judge-prompt`, `validate-evaluator`).
