# Predicting the opening week of *Avatar: Fire and Ash*

A box-office forecast, submitted before the film released, and scored afterward.

| | |
|---|---|
| **Predicted** (7-day domestic, submitted 15 Dec 2025) | **USD 211.0M** |
| **Actual** (7-day domestic, 19–25 Dec 2025) | **USD 153.8M** |
| **Error** | **+37.2%** |

The model predicted a franchise step-up. *Fire and Ash* opened 22% below *The Way of Water* in nominal terms, and 30% below it in inflation-adjusted terms. It was a franchise step-down, and nothing in the model could represent one.

Built as coursework for a Machine Learning course (K401) at the Institute of Business Administration, University of Dhaka. The forecast deadline was 15 December 2025, four days before release, which shaped every feature choice in it.

**[POSTMORTEM.md](POSTMORTEM.md) is the interesting document in this repository.** It records what was predicted, what happened, which parts of the code were wrong, and which of those errors actually mattered.

---

## Why the model missed

**None of the franchise features can express a franchise declining from its predecessor.**

Two features carry franchise information. `sequel_parent_revenue_adj_log` is the previous entry's domestic lifetime gross — it correlates +0.705 with the target among sequels and is strictly monotonic in the predecessor's success, so an enormous *Way of Water* pushed the forecast hard upward. `franchise_hunger` is the gap in years since the previous entry; *Fire and Ash* arrived 3.01 years later, which sits in the highest-grossing gap bucket in the dataset, so it nudged upward too.

Between them these encode "the last one was big" and "the gap was short". Neither can encode "the audience has had enough". A franchise's third entry and its first are described by identical columns, so nothing in the franchise features measures fatigue.

**But the decline signal was in the data — it just wasn't in the franchise features, and the forecast discarded it.**

*The Way of Water* opened in **4,202** theaters. *Fire and Ash* opened in **3,800** — a 9.6% cut. 20th Century Studios, with access to tracking and pre-sales, booked this film into materially fewer screens than its predecessor. That is a studio forecasting softer demand, and it is exactly the kind of judgement `opening_theaters` encodes.

The forecast assumed 4,000. At the true 3,800 the corrected model predicts **$132.2M** against an actual $153.8M.

So the honest version of "why the model missed" is not that the information was unavailable. The franchise features were blind to decline, but the model's single most important feature — 54% of total importance — was carrying the studio's own downgrade, and an unverified assumption threw part of it away. The fix for the first problem is a new feature. The fix for the second is to confirm your inputs, or publish the forecast as a function of the ones you cannot.

## What the model was actually using

| Feature set | n | Test R² | Top-decile precision |
|---|---|---|---|
| Full feature set | 55 | 0.9082 | 67.2% |
| `opening_theaters` alone | 1 | 0.8533 | 56.9% |
| theaters + budget + distributor | 3 | 0.8951 | 67.2% |
| No full-data aggregates | 50 | 0.8994 | 67.2% |

One variable delivers 94% of the full model's R². Three deliver 99%.

The star-power History Engine, the cannibalization OLS, the decay clustering, the Wikipedia hindcasting and the market-momentum EMA are collectively worth about 0.013 R². The History Engine is the most careful piece of engineering here — chronological processing with a strict read-then-write protocol, `NaN` for cold starts rather than imputation, three distinct career signals — and on this task it is close to decorative. It is kept because it is correct and because it would matter in a model built further from release.

## What `opening_theaters` actually is

It is not a predictor of the box office so much as a reading of someone else's prediction.

Theater counts are set by the distributor in the days before release, in response to tracking data and pre-sales. The model's dominant feature — 54% of total importance — is effectively the studio's own demand forecast, available only about a week out. An R² of 0.90 on a task this noisy should be treated as a signal that something like this is in the feature set, and here it is.

It is also the largest source of uncertainty in the forecast, and it was never reported as one. The theater count was unconfirmed at submission ("widely believed to be 3800–4000"); 4,000 was assumed and 3,800 was actual. Across a plausible 3,500–4,500 range the forecast moves **$68.1M**, against a $33.8M gap between the two models. The published range of $202M–$220M was built entirely from model disagreement — the smaller source of uncertainty — while the larger one was treated as known.

## Known limitations

- **Preprocessing leakage.** `distributor_strength`, `release_tier`, `hype_cluster`, `cannibalization_score` and `market_momentum_lagged` are fitted across the full dataset before the train/test split. The ablation puts the cost at 0.009 R², so the inflation is small — but the architecture is wrong, and the fix is to fit them inside a pipeline on training folds only.
- **Top-decile shrinkage.** The model predicts 37% *below* actual in the top decile and 55% *above* in the bottom — ordinary regression to the mean, and worst exactly where this project needed it to be best.
- **Sample selection.** `dropna(subset=['rev_day_7_adj'])` leaves 2,940 of 6,706 films. BoxOfficeMojo publishes daily breakdowns only for wide releases, so the surviving sample skews toward the films the model then claims to be good at.
- **`decay_cluster`** is computed across five cells and never enters the feature set. It is exploratory analysis, not modelling.
- **Extraction code is not in the repository.** The scraping and API work across BoxOfficeMojo and TMDB for 6,700 films is arguably the most substantial engineering in the project, and it is currently invisible.

## What I would do differently

1. Fit every aggregate inside a pipeline on training folds rather than across the full dataset before splitting.
2. Build a T-minus-six-months variant with no distribution signals — no `opening_theaters`, no `distributor_strength`. The R² would fall a long way, and what remained would be a genuine forecast rather than a reading of the studio's.
3. Report intervals derived from input uncertainty, not from model disagreement.
4. Add a franchise-trajectory feature so decline is at least representable.
5. Re-run before publishing. Three of the four defects below would have been caught by a single `Restart & Run All`.

---

## Correction notice

> The version submitted for assessment on 15 December 2025 is preserved, unmodified, at the tag [**`v1.0-as-submitted`**](https://github.com/Jawaad512/K401-Box-Office-Prediction-2025/tree/v1.0-as-submitted). Everything below documents what changed since, and why.

The notebook as originally submitted could not be re-run to produce the numbers it reported. Four defects were responsible, two of them leakage:

| # | Defect | Effect on the anchor |
|---|---|---|
| 1 | `covid_impact` set to `1` for a December 2025 release, outside the notebook's own COVID window of 2020-03-01 to 2021-12-31 | +$51M |
| 2 | `cannibalization_score` transferred between frames by assignment rather than a key join, landing on the wrong films | corrupted most rows |
| 3 | `merge_asof(direction='nearest')`, which can pull a film's market-momentum value from a week *after* it opened | **−$96M** |
| 4 | `perf_ratio` — a weekly revenue median computed over the films released that week, including the film itself | −$9M |

Defect 3 was the largest single effect in the project, and it was leakage rather than a typo. Fixing it also reversed the model tournament: CatBoost ranked third of four on top-decile precision and fold stability beforehand, and first on every metric afterward.

With all four corrected, the notebook produces:

| Model | Loss function | Prediction |
|---|---|---|
| Anchor | RMSE | **USD 143.1M** |
| Ceiling | Quantile, alpha = 0.90 | **USD 176.9M** |

Reported as a central estimate and an upper bound rather than averaged. The anchor is close to unbiased on the test set (log bias +0.028); the quantile model is deliberately biased upward (+0.628, a multiplier of about 1.87×). Averaging an unbiased estimator with one inflated by design does not estimate anything — which the original notebook half-acknowledged when it conceded there was "no statistically significant weight" to justify the midpoint.

The corrected central estimate lands about 7% below the actual result, against +37% for the submitted figure. **That is not vindication, and [POSTMORTEM.md](POSTMORTEM.md) §3 explains why.** The model shrinks every top-decile film by roughly a third; Avatar genuinely declined; the shrinkage imitated franchise fatigue rather than detecting it. On a franchise that grew, the same behaviour would have missed badly in the other direction. It is a sample of one.

## Method

1. **Extraction.** ~6,700 films from 1990–2025, via BoxOfficeMojo scraping and the TMDB API.
2. **Transformation.** Cleaning, CPI adjustment to 2025 dollars, and engineered features for star power, competition, seasonality, market momentum and hype.
3. **Modelling.** Four gradient boosting machines against a linear-regression baseline. CatBoost was selected — it leads on test R², log RMSE, top-decile precision, mean cross-validated R² and fold stability, though by margins small enough (0.0016 R² over XGBoost) that the choice is defensible rather than decisive.
4. **Prediction.** An RMSE anchor for the central estimate and a 90th-percentile quantile model for the upper bound.

## Reproducing

`Kernel → Restart & Run All` reproduces every number in the notebook, including the artifacts. Execution counts run 1..N in order with no errors.

```
pip install -r requirements.txt
```

The committed run used Python 3.12.10 with numpy 2.5.3, pandas 2.3.3, scikit-learn 1.9.1, CatBoost 1.2.10, XGBoost 3.4.1 and LightGBM 4.7.0. Exact pins are in [requirements.txt](requirements.txt).

Two environment constraints are worth knowing:

- **pandas 3.x is not supported.** The cleaning cells rely on pre-copy-on-write assignment semantics.
- **LightGBM 4.7 segfaults against numpy < 2**, so numpy and LightGBM must be upgraded together.

The run is deterministic — independent executions produce identical predictions to the cent. The notebook writes an intermediate `dataset_refined.csv` into the working directory at the end of feature engineering; it is regenerated on every run and is gitignored.

## Files

| Path | Description |
|---|---|
| [`POSTMORTEM.md`](POSTMORTEM.md) | Predicted vs actual, the four defects, and what each was worth. Start here. |
| `Box_Office_Prediction.ipynb` | The full pipeline: extraction notes, cleaning, feature engineering, modelling, diagnostics and prediction. |
| `box_office_dataset_1990-2025_revised.csv` | The master dataset, 6,706 films. |
| [`DATA_DICTIONARY.md`](DATA_DICTIONARY.md) | All 56 columns: type, coverage and definition. |
| `Artifacts/` | Both trained models (`catboost_conservative.cbm`, `catboost_quantile_90.cbm`), the feature schema and the prediction summary. All regenerated by the notebook's last cell, so they cannot drift from the code that produced them. |
| `images/` | Figures referenced by the notebook's markdown, extracted from previously inlined base64. |
| `requirements.txt` | Pinned dependencies. |

## License

MIT — see [LICENSE](LICENSE).
