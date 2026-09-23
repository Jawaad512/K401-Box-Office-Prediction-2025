# K401-Box-Office-Prediction-2025

This is a box-office movie revenue prediction project completed as a coursework for a Machine Learning course (K401) in the Institute of Business Administration, University of Dhaka. 

## Objective

The key objective was to predict the opening week aggregated revenue (domestic) for the movie 'Avatar: Fire and Ash' which, during the completion of the project, remained to be released on 19 December, 2025. The project deadline was 15 December, 2025 (EOD), significantly shaping the choice of features taken to predict.

## Steps

1. Extraction: Around 6700 movies were extracted from 1990-2025 to populate the master dataset for the project. 'BoxOfficeMojo' and 'TMDB' were accessed through API and scraping for movie data.

2. Transformation: The master dataset was cleaned and some new features were engineered to reflect various time-based trends (e.g. seasonality) which are not cleanly reflected in raw data.

3. Modelling: Four Gradient Boosting Machines were pit alongside a baseline linear regression model to choose the best option for the prediction. Finally, CatBoost was chosen. Given the Avatar franchise's track record as a blockbuster, two versions of CatBoost were chosen - the first version was trained with a simple RMSE loss function, and the second was trained with a 90th percentile loss function to get closer to outliers.

4. Final Prediction: **USD 211M** was submitted on 15 December 2025, as the midpoint of the two models (USD 202M from the RMSE anchor, USD 220M from the 90th percentile model).

## Correction notice

The notebook as originally submitted could not be re-run to produce the numbers it reported. Four defects were responsible, two of them leakage:

| # | Defect | Effect |
|---|---|---|
| 1 | `covid_impact` set to `1` in the Avatar input, for a film released in December 2025 — outside the notebook's own COVID window of 2020-03-01 to 2021-12-31 | +$51M on the anchor |
| 2 | `cannibalization_score` transferred between frames by assignment rather than a key join. Pandas aligns on the index, and the receiving frame had been re-indexed and re-sorted in between, so the score landed on the wrong films | corrupted most rows |
| 3 | `merge_asof(direction='nearest')` on the weekly market table, which can pull a film's market-momentum value from a week *after* it opened | **−$96M on the anchor** |
| 4 | `perf_ratio` — the median 7-day revenue of a film's own release week, a median computed over the films released that week, including the film itself | −$9M on the anchor |

Defect 3 was the largest single effect in the project, and it was leakage rather than a typo. Fixing it also reversed the model tournament: CatBoost ranked third of four on top-decile precision and fold stability beforehand, and first on every metric afterward.

With all four corrected, the notebook produces:

| Model | Loss function | Prediction |
|---|---|---|
| Anchor | RMSE | **USD 143.1M** |
| Ceiling | Quantile, alpha = 0.90 | **USD 176.9M** |

These are reported as a central estimate and an upper bound rather than averaged. The anchor is close to unbiased on the test set (log bias +0.028); the quantile model is deliberately biased upward (+0.628, a multiplier of about 1.87x). Averaging an unbiased estimator with one inflated by design does not estimate anything.

*Avatar: Fire and Ash* opened to **USD 153.8M** over its first seven days. The corrected central estimate is about 7% below that; the submitted USD 211M was about 37% above it.

That closeness should not be read as vindication, and the notebook does not read it that way. The model under-predicts the top decile of the test set by 37% — ordinary regression to the mean — and Avatar genuinely declined from its predecessor. The shrinkage happened to imitate franchise decay on this one film. Every franchise feature in the model (`franchise_hunger`, `sequel_parent_revenue`) is monotonically positive in past franchise success, so the model still has no way to represent a franchise declining. It would miss in the other direction on a franchise that grows. This is a sample of one.

## Known limitations

- **Preprocessing leakage.** `distributor_strength`, `release_tier`, `hype_cluster`, `cannibalization_score` and `market_momentum_lagged` are all fitted across the full dataset before the train/test split. The ablation in the notebook measures the cost at 0.009 R², so the inflation is small — but the architecture is wrong, and the fix is to fit them inside a pipeline on training folds only.
- **`opening_theaters` is close to the answer, not a predictor.** It carries 54% of total feature importance, and on its own delivers 94% of the full model's R². Theater counts are set by the distributor days before release in response to tracking and pre-sales, so the model's dominant feature is effectively the studio's own demand forecast.
- **Sample selection.** `dropna(subset=['rev_day_7_adj'])` leaves 2,940 of 6,706 films, skewed toward wide releases.
- **Top-decile shrinkage.** The calibration table in the notebook shows the model predicting 37% below actual in the top decile and 55% above in the bottom.

## Reproducing

`Kernel → Restart & Run All` reproduces every number in the notebook, including the artifacts. The committed run was produced in:

| | |
|---|---|
| Python | 3.12.10 |
| numpy / pandas / scipy | 2.5.3 / 2.3.3 / 1.18.1 |
| scikit-learn / statsmodels | 1.9.1 / 0.15.0 |
| CatBoost / XGBoost / LightGBM | 1.2.10 / 3.4.1 / 4.7.0 |
| matplotlib / seaborn | 3.11.2 / 0.13.2 |

The run is deterministic: independent executions produce identical predictions to the cent. Cell 123 writes an intermediate `dataset_refined.csv` into the working directory; it is regenerated on every run and is not committed.

Note that pandas 3.x is *not* supported — the cleaning cells rely on pre-copy-on-write assignment semantics.

## Files

1. `Artifacts/` contains both trained models — `catboost_conservative.cbm` (the RMSE anchor) and `catboost_quantile_90.cbm` (the 90th percentile ceiling) — along with the feature schema and the prediction summary. All four are regenerated by the last cell of the notebook, so they cannot drift from the code that produced them.
2. `Box_Office_Prediction.ipynb` is the key ipynb file containing the entire prediction.
3. `box_office_dataset_1990-2025_revised.csv` is the master dataset for the project.
4. `required_libraries.txt` are the libraries needed to be installed prior to running the code in an ipynb supported environment.
5. `images/` holds the figures referenced by the notebook's markdown, extracted from the base64 that was previously inlined.
