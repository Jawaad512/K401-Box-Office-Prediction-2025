# Post-mortem: forecasting *Avatar: Fire and Ash*

A forecast was submitted on 15 December 2025, four days before the film opened. The film has since released, so the forecast is scoreable. This document records what was predicted, what happened, what was wrong with the code that produced it, and which of those errors actually mattered.

---

## 1. The result

| | |
|---|---|
| Submitted prediction (7-day domestic) | **USD 211.0M** |
| Actual (7-day domestic) | **USD 153.8M** |
| Error | **+37.2%** |

Daily domestic gross, 19–25 December 2025:

| Dec 19 | Dec 20 | Dec 21 | Dec 22 | Dec 23 | Dec 24 | Dec 25 |
|---|---|---|---|---|---|---|
| $36.5M | $28.4M | $24.3M | $13.3M | $16.5M | $10.7M | $24.1M |

Opening three-day weekend: $89.2M.

The comparison that matters is against the predecessor. *Avatar: The Way of Water* took $197.7M over its first seven days in December 2022. In nominal terms *Fire and Ash* fell **22.2%** short of it; adjusted to 2025 dollars, which is the space the model works in, the predecessor's figure is $219.1M and the real decline is **29.8%**.

The model predicted a franchise step-up. The franchise stepped down.

---

## 2. Four defects, and which ones mattered

The notebook as submitted could not be re-run to reproduce its own published numbers. Re-executing it from a clean kernel surfaced four defects. They are listed below with what each was worth, measured by fixing them one at a time.

| # | Defect | Effect on the anchor |
|---|---|---|
| 1 | `covid_impact` hard-coded to `1` for a December 2025 release, outside the notebook's own COVID window | **+$51M** |
| 2 | `cannibalization_score` assigned by index instead of joined on `tmdb_id`, landing on the wrong films | corrupted most rows |
| 3 | `merge_asof(direction='nearest')`, letting a film's market-momentum value come from a week *after* it opened | **−$96M** |
| 4 | `perf_ratio`, a weekly revenue median that included the film's own revenue | **−$9M** |

Walking the corrections forward:

| State | Anchor | vs actual $153.8M |
|---|---|---|
| As submitted | $211.0M (midpoint) | +37.2% |
| Defects 1 and 2 fixed | $248.5M | +61.6% |
| Defect 3 also fixed | $152.4M | −0.9% |
| Defect 4 also fixed — current | **$143.1M** | **−7.0%** |

Three things are worth pulling out of that table.

**The COVID flag made the forecast look better, not worse.** Setting `covid_impact` to 1 suppressed the prediction rather than inflating it: on the corrected pipeline, flipping it back to 1 moves the anchor from $248.5M down to $197.2M. Had that one flag been right and the two leaks left in place, the submitted forecast would have been substantially *higher* than $211M and the published error correspondingly worse. Two mistakes partially cancelled, and the visible outcome was less wrong than the model behind it.

**The largest single error was leakage, not a typo.** One word — `'nearest'` instead of `'backward'` in a `merge_asof` — was worth $96M, more than the COVID flag and the cannibalization bug combined. It let the market-momentum feature read partly from the present rather than the past. This is the least visible defect in the project and by far the most consequential.

**Fixing the leak also reversed a modelling conclusion.** With the leak present, CatBoost ranked third of four on top-decile precision and fold stability. With it removed, CatBoost ranks first on every metric. The original write-up chose CatBoost for a reason that was factually wrong (see §4); the choice turns out to be defensible, but not for any reason that was given at the time.

---

## 3. Why the corrected number is not a success story

The corrected model predicts $143.1M against an actual $153.8M — about 7% low. That is a better result than the submitted forecast by a wide margin, and it should not be read as the model working.

The model systematically under-predicts large films. Binning the test set by decile of actual revenue and comparing medians:

| Decile of actual | 0 | 3 | 5 | 7 | 8 | 9 (top) |
|---|---|---|---|---|---|---|
| Predicted ÷ actual | 1.55 | 1.69 | 1.08 | 0.85 | 0.75 | **0.63** |

The model shrinks the top decile by 37% and inflates the bottom by 55%. This is ordinary regression to the mean, and it is the exact failure mode the 90th-percentile model was bolted on to compensate for.

So the model arrived near the right answer by applying a large downward correction that it applies to *every* blockbuster, to a film that genuinely declined. The shrinkage imitated franchise fatigue. It did not detect it.

On a franchise entry that grew instead of shrank, the same shrinkage would have produced a large miss in the opposite direction. This is a sample of one, and the mechanism that produced the good result is not one that generalises.

---

## 4. The blind spot the model actually has

No feature in the model can express a franchise declining from its predecessor.

Two features carry franchise information:

- **`sequel_parent_revenue_adj_log`** — the previous entry's domestic lifetime gross. It correlates **+0.705** with the target among sequels. It is strictly monotonic in the predecessor's success: the bigger the last film, the higher this pushes the prediction. *The Way of Water* was enormous, so this feature pushed hard upward.
- **`franchise_hunger`** — years since the previous franchise entry. Median 3.4 years. It correlates **−0.067** with the target, so longer gaps are associated with marginally lower openings. *Fire and Ash* arrived 3.01 years after its predecessor, which sits in the 2–4 year bucket — the bucket with the *highest* median 7-day gross in the dataset ($66.0M). So this feature also nudged upward, though weakly and for a different reason.

Between them these features encode "the last one was big" and "the gap was short". Neither can encode "the audience has had enough". There is no *franchise* feature for declining interest in an ongoing series — a franchise's third entry and its first are described by the same columns.

This is a limitation of the feature set, not of the algorithm. No amount of tuning would have found it.

### The signal was there anyway, and the forecast discarded it

That claim should not be overstated, because the decline *was* visible in the data — just not in the franchise features.

| | Opening theaters |
|---|---|
| *Avatar: The Way of Water* (2022) | 4,202 |
| *Avatar: Fire and Ash* (2025) | **3,800** |
| Assumed at forecast time | 4,000 |

20th Century Studios booked *Fire and Ash* into **9.6% fewer theaters** than its predecessor. Theater counts are set in the days before release in response to tracking and pre-sales, so this is the distributor publishing its own view that demand would be softer than last time. It is precisely the signal the franchise features lacked — and `opening_theaters`, at 54% of the model's total importance, was carrying it.

The forecast assumed 4,000 rather than confirming 3,800. At the true value the corrected model predicts **$132.2M** against an actual $153.8M.

So there are two distinct failures here, with different remedies:

1. **The franchise features cannot represent decline.** Fixing this needs a new feature — a trajectory ratio across a series, for instance.
2. **The one feature that did carry the decline was overridden by an unverified assumption.** Fixing this needs no modelling at all. It needs either confirming the input, or publishing the forecast as a function of the input you cannot confirm.

The second is the more embarrassing of the two, and by far the cheaper to avoid.

---

## 5. The uncertainty that was never reported

The notebook acknowledged that the opening theater count was unconfirmed — "widely believed to be 3800–4000" — and then chose 4,000 and proceeded to a single point estimate. Since `opening_theaters` carries 54% of the model's feature importance, that assumption deserved to be quantified.

| `opening_theaters` | Anchor | Ceiling (Q0.90) |
|---|---|---|
| 3,500 | $110.2M | $146.3M |
| **3,800 (actual)** | **$132.2M** | $172.0M |
| 4,000 (assumed) | $143.1M | $176.9M |
| 4,200 | $171.6M | $208.0M |
| 4,500 | $178.3M | $216.6M |

The theater count actually came in at 3,800. Across the plausible 3,500–4,500 range the forecast moves **$68.1M**. The gap between the two *models* at the chosen input is **$33.8M**.

The published range — $202M to $220M, an $18M spread — was built entirely from model disagreement, and reported the smaller source of uncertainty while treating the larger one as known. A forecast this sensitive to one unconfirmed input should have been published as a function of that input.

---

## 6. What the model was really using

An ablation on the test set, retraining the tuned anchor on progressively smaller feature sets:

| Feature set | n | Test R² | Top-decile precision |
|---|---|---|---|
| Full feature set | 55 | 0.9082 | 67.2% |
| `opening_theaters` alone | 1 | 0.8533 | 56.9% |
| theaters + budget + distributor | 3 | 0.8951 | 67.2% |
| No full-data aggregates | 50 | 0.8994 | 67.2% |

One variable delivers 94% of the full model's R². Three deliver 99%.

The star-power History Engine, the cannibalization OLS, the decay clustering, the Wikipedia hindcasting and the market-momentum EMA are collectively worth about 0.013 R². The History Engine in particular is the most careful piece of engineering in the project — chronological processing with a strict read-then-write protocol, `NaN` for cold starts rather than imputation, three distinct career signals — and on this task it is close to decorative.

That is worth saying plainly, because the alternative is a suspiciously high R² that a reader will correctly distrust.

There is a sharper point underneath it. `opening_theaters` is set by the distributor in the days before release, in response to tracking data and pre-sales. It is the studio's own demand forecast. A model whose dominant feature is someone else's forecast is not independently predicting the box office so much as reading that forecast back with extra steps.

---

## 7. A correction to an earlier claim

An earlier version of this analysis asserted that the overshoot was "structural, not a data-entry artifact" and that correcting the inputs would not rescue it. That was based on measuring the COVID flag and the theater assumption, but not the `merge_asof` leakage. Once that leak is removed the overshoot largely disappears, so the claim was wrong. It is left here rather than deleted because the reasoning failure — concluding that a problem was structural before all the measurable causes had been measured — is the more useful thing to remember.

---

## 8. Known limitations that remain

- **Preprocessing leakage.** `distributor_strength`, `release_tier`, `hype_cluster`, `cannibalization_score` and `market_momentum_lagged` are all fitted across the full dataset before the train/test split. The ablation puts the cost at 0.009 R², so the inflation is small — but the architecture is wrong. The fix is to fit them inside a pipeline on training folds only.
- **Sample selection.** `dropna(subset=['rev_day_7_adj'])` leaves 2,940 of 6,706 films. BoxOfficeMojo only publishes daily breakdowns for wide releases, so the surviving sample is skewed toward exactly the films the model then claims to be good at.
- **Top-decile shrinkage.** Documented in §3 and uncorrected. The 90th-percentile model is a blunt patch, not a calibration.
- **`decay_cluster`** is computed across five cells and never enters the feature set. It is exploratory analysis, not modelling.

---

## 9. What I would do differently

1. **Fit every aggregate inside a pipeline** on training folds, rather than across the full dataset before splitting. This is the single change that would most improve the project's defensibility.
2. **Build a T-minus-six-months variant** with no distribution signals — no `opening_theaters`, no `distributor_strength`. The R² would fall a long way, and the resulting model would be a genuine forecast rather than a reading of the studio's own. The contrast between the two is more interesting than either alone.
3. **Report intervals, not points**, and derive them from input uncertainty rather than model disagreement.
4. **Add a franchise-trajectory feature** — the ratio of each entry to the previous one across a series — so that decline is at least representable.
5. **Re-run before publishing.** Three of the four defects above would have been caught by a single `Restart & Run All`, because the notebook could not reproduce its own numbers and nobody checked.
