# Data Dictionary

`box_office_dataset_1990-2025_revised.csv` — 6,706 films released 1990–2025, 56 columns.

Assembled from **BoxOfficeMojo** (revenue, theater counts, daily ranks, distributor) and **TMDB** (identifiers, cast and crew, genres, runtime, budget). The extraction code is not currently in the repository.

"Coverage" is the share of the 6,706 rows that are non-null. Several revenue columns sit near 50% because BoxOfficeMojo daily breakdowns only exist for wide releases — this is the source of the sample-selection bias noted in the README.

## Identifiers and release

| Column | Type | Coverage | Description |
|---|---|---|---|
| `Unnamed: 0` | int | 100% | Export artifact from a pandas `to_csv` with the index included. Dropped by the notebook. |
| `title` | str | 100% | Film title as listed on TMDB. |
| `tmdb_id` | int | 100% | TMDB identifier. Unique after cleaning, and the join key used throughout the notebook. |
| `imdb_id` | str | 100% | IMDb identifier (`tt` + digits). Used for de-duplication. |
| `release_date` | date | 100% | US theatrical release date, `YYYY-MM-DD`. |
| `release_month` | int | 100% | Month of release, 1–12. |
| `release_day_of_week` | int | 100% | Day of week, **Monday = 0 … Sunday = 6**. 43% of the dataset is 4 (Friday). |
| `original_language` | str | 100% | ISO 639-1 code of the original language. |
| `origin_country` | str | 100% | ISO 3166-1 country of origin. 48 distinct values, 83% `US`. Bucketed by the notebook into `origin_country_refined`. |

## Film attributes

| Column | Type | Coverage | Description |
|---|---|---|---|
| `production_budget_raw` | float | 75.2% | Reported production budget in **nominal** USD. |
| `production_budget_adj` | float | 75.2% | Production budget in **2025 USD**, CPI-adjusted by release year. |
| `runtime` | int | 100% | Runtime in minutes. |
| `primary_genre` | str | 100% | First genre in `genres_list`. Redundant, and dropped by the notebook. |
| `genres_list` | str | 100% | Pipe-delimited TMDB genres, e.g. `Action\|Adventure\|Science Fiction`. One-hot encoded into `genre_*` columns. |
| `mpaa_rating` | str | 100% | MPAA rating. Seven values, including 5 rows of literal `Error` which the notebook drops. Bucketed into `mpaa_rating_refined`. |
| `format_premium` | float | 100% | Only two values: `1.00` (6,703 rows) and `1.25` (3 rows). Effectively constant, and consequently of near-zero feature importance. Dropped from the model. The notebook does not define how it was derived. |
| `spoiler_risk` | float | 100% | Values in {0, 0.5, 1, 1.5, 2, 2.5, 3.5}; 79% are 0. Dropped from the model. The notebook does not define how it was derived. |
| `wiki_hype` | float | 26.8% | Total English Wikipedia pageviews over the 30 days before release. The Wikimedia pageviews API only begins in 2015, which is why coverage is low. Divided by 30 in the notebook to give a daily average. **Never used as a model feature** — see the README correction notice. |

## Cast and crew

| Column | Type | Coverage | Description |
|---|---|---|---|
| `director_id` | float | 100% | TMDB person ID of the director. |
| `director_name` | str | 100% | Director name. |
| `producer_id` | float | 98.5% | TMDB person ID of the lead producer. |
| `producer_name` | str | 100% | Producer name. |
| `cast_ids` | str | 99.9% | Pipe-delimited TMDB person IDs, top-billed first. The notebook uses the first three. |
| `cast_names` | str | 99.9% | Pipe-delimited cast names, aligned with `cast_ids`. |

## Franchise

| Column | Type | Coverage | Description |
|---|---|---|---|
| `is_sequel` | int | 100% | 1 if the film is a sequel or franchise entry, else 0. 1,006 sequels. |
| `sequel_parent_rating` | float | 14.7% | TMDB rating of the previous franchise entry, 0–10. |
| `sequel_parent_revenue_adj` | float | 12.4% | Domestic lifetime revenue of the previous franchise entry, in 2025 USD. Correlates **+0.705** with the target among sequels — it is strongly monotonic in the predecessor's success. |
| `franchise_hunger` | float | 14.7% | **Years elapsed since the previous franchise entry.** Verified against known films: *Top Gun: Maverick* 36.0, *Blade Runner 2049* 35.3, *Incredibles 2* 13.6, *Avatar: The Way of Water* 13.0. Median 3.4 years. Correlates **−0.067** with the target — longer gaps are associated with marginally *lower* opening revenue, not higher. |

## Distribution

| Column | Type | Coverage | Description |
|---|---|---|---|
| `distributor` | str | 100% | Distributing studio as listed by BoxOfficeMojo. Clustered into `distributor_strength` tiers 0–6. |
| `opening_theaters` | float | 50.6% | Number of theaters on opening day. **The single dominant feature in the model** (54% of total importance). Set by the distributor days before release in response to tracking and pre-sales — closer to the studio's own forecast than to an independent predictor. |
| `calendar_ratio` | float | 49.9% | A ratio, median 0.62, range 0.01–107.6. Not used anywhere in the notebook and its derivation is not documented. Treat as unverified. |

## Revenue — aggregates

All revenue columns are **nominal USD**. The notebook creates CPI-adjusted `_adj` counterparts in 2025 dollars.

| Column | Type | Coverage | Description |
|---|---|---|---|
| `bom_dom_lifetime` | float | 84.4% | Domestic (US + Canada) lifetime gross. Drives the star-power History Engine. |
| `bom_intl_lifetime` | float | 63.0% | International lifetime gross. |
| `bom_ww_lifetime` | float | 84.4% | Worldwide lifetime gross. |
| `target_dom_7day` | float | 50.7% | **The prediction target.** Domestic gross over the first seven days. The model is trained on `log1p` of the CPI-adjusted version. |

## Revenue — daily, days 1–7

Three columns per day for the first seven days of release, `N` = 1…7. Coverage declines gently from 50.7% on day 1 to 49.2% on day 7.

| Column | Type | Description |
|---|---|---|
| `rev_day_N` | float | Domestic gross on day N of release, nominal USD. Day 1 is release day. |
| `theaters_day_N` | float | Theater count on day N. |
| `rank_day_N` | float | Daily box-office rank on day N, 1 = highest-grossing film that day. |

The daily columns feed the decay-profile clustering (the ratio of each day to day 1) and the seasonality retention scores. They are **not** available at prediction time for an unreleased film, and are not model features.

## Columns created by the notebook

These do not exist in the CSV. They are engineered during the run and written to the intermediate `dataset_refined.csv`:

`*_adj` (CPI-adjusted revenue), `*_log` (log1p transforms), `year`, `genre_*` (one-hot), `mpaa_rating_refined`, `origin_country_refined`, `{director,producer,cast_1,cast_2,cast_3}_{peak,momentum,stability,is_debut}` (star-power History Engine), `decay_cluster`, `comp_budget_pressure`, `cannibalization_score`, `distributor_strength`, `release_week`, `release_tier`, `market_momentum_lagged`, `perf_ratio`, `hype_cluster`, `wiki_hype_avg`, `wiki_hype_avg_imputed`, `covid_impact`, `target_dom_7day_log`.

Note that `decay_cluster` and `perf_ratio` are computed but **not** used as model features — `decay_cluster` is exploratory, and `perf_ratio` was removed as target leakage.
