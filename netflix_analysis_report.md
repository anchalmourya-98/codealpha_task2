# Netflix Titles — Exploratory Data Analysis Report

**Dataset:** `netflix_titles.csv` (8,807 rows, 12 columns)
**Source:** Kaggle — Netflix Movies and TV Shows
**Author:** Anchal

---

## 1. Questions Asked Before Analysis

Before touching the data, I framed the questions I wanted the analysis to answer. This keeps the exploration goal-directed instead of just "running functions on a dataframe."

1. Is Netflix's catalog more movie-heavy or TV-heavy, and has that mix shifted over time?
2. When did Netflix add most of its current catalog — is growth steady, accelerating, or slowing?
3. Which countries and genres dominate the catalog, and do the country/genre fields even support a clean single-label count?
4. Has content skewed toward more mature ratings over time, or has the audience mix stayed stable?
5. Do movies from different countries differ systematically in length (e.g., Bollywood vs Hollywood runtimes)?
6. Where are the structural weak points in this dataset (missing values, duplicate encodings, logically inconsistent rows) that would bias any of the above answers if ignored?

Each section below answers one or more of these.

---

## 2. Data Structure

| Column | Type | Non-null | Notes |
|---|---|---|---|
| show_id | object | 8,807 | unique identifier, no duplicates |
| type | object | 8,807 | `Movie` or `TV Show` |
| title | object | 8,807 | no duplicate titles |
| director | object | 6,173 | **2,634 missing (29.9%)** |
| cast | object | 7,982 | 825 missing |
| country | object | 7,976 | 831 missing; **multi-valued**, comma-separated |
| date_added | object → datetime | 8,797 | 10 missing; parsed with mixed format |
| release_year | int64 | 8,807 | complete |
| rating | object | 8,803 | 4 missing |
| duration | object | 8,804 | 3 missing; **mixed units** — "min" for movies, "Season(s)" for TV shows |
| listed_in | object | 8,807 | **multi-valued**, comma-separated genre tags |
| description | object | 8,807 | free text |

**Derived columns created for analysis:** `year_added` (from `date_added`), `mature` (boolean flag for TV-MA/R/NC-17), `duration_min` (numeric minutes, movies only), `country_first` (first listed country, for single-label comparisons).

Two structural issues shape every later step: `country` and `listed_in` are comma-separated multi-label fields, and `duration` encodes two different units depending on `type`. Both are treated explicitly in the sections below rather than silently averaged over.

---

## 3. Trends, Patterns, and Anomalies

### 3.1 Movies dominate the catalog 2:1
6,131 movies vs. 2,676 TV shows.

![Type distribution](images/01_type_distribution.png)

### 3.2 Catalog growth peaked in 2019
Titles added per year rose steadily from 2015 through 2019 (82 → 429 → 1,188 → 1,649 → 2,016), then declined in 2020 (1,879). **Anomaly check:** the raw data also showed 1,498 titles for 2021 — but the latest `date_added` in the file is **2021-09-25**, so 2021 is a partial year, not a real decline. It is excluded from the chart below to avoid a false "Netflix is shrinking" conclusion.

![Titles added by year](images/02_titles_by_year.png)

### 3.3 Genre and country fields are multi-valued — raw counts overstate precision
Exploding `listed_in` on commas, the top genres are International Movies (2,752), Dramas (2,427), Comedies (1,674), International TV Shows (1,351), and Documentaries (869). Exploding `country` the same way: United States (3,689), India (1,046), United Kingdom (804), Canada (445), France (393).

**These are tag-appearance counts, not title counts** — a single title with `country = "United States, India"` is counted once in each bucket. Reporting these as "1,046 Indian titles" without that caveat would misrepresent the data.

![Top genres](images/03_top_genres.png)
![Top countries](images/04_top_countries.png)

### 3.4 Content skews toward mature ratings
TV-MA is the single largest rating (3,207 titles, 36.4%), followed by TV-14 (2,160, 24.5%). Combined, TV-MA + TV-14 account for over 60% of the catalog.

![Rating distribution](images/05_rating_distribution.png)

### 3.5 Anomaly: 14 titles were "added" before their release year
`year_added` is earlier than `release_year` for 14 titles (e.g., *Fuller House*, released 2020, added 2019). This is very likely a **regional release-date mismatch** in the source data (Netflix's own metadata sometimes tags `release_year` as the year of a later re-release or restoration) rather than a genuine data-entry error — flagged here, not silently dropped. [likely]

---

## 4. Hypothesis Testing

### H1 — "Netflix's catalog has become more mature-rated over time" (2015–2020, restricted to years with reasonable sample sizes to avoid the 2008–2013 near-zero-count years distorting the test)

- Mature share by year: 2015: 39.0% → 2016: 41.3% → 2017: 43.2% → 2018: 47.2% → 2019: 46.9% → 2020: 45.7%
- Chi-square test of independence (year × mature/non-mature): χ² = 10.61, **p = 0.060**

**Result: not statistically significant at the conventional α = 0.05 threshold**, though the direction is consistent (a ~7-point rise from 2015 to 2018, then a plateau). [certain on the numbers, likely on interpretation] The honest conclusion is "a mild upward trend that the data doesn't confirm at standard significance," not "Netflix definitely skews more mature over time." A larger or more recent dataset would be needed to confirm the trend.

![Mature content trend](images/06_mature_trend.png)

### H2 — "Movies and TV shows come from different release-year distributions"

- Median release year: Movies = 2016, TV Shows = 2018
- Mann-Whitney U test (non-parametric, since release year is skewed): **p ≈ 2.1 × 10⁻¹⁰⁵**

**Result: statistically significant.** [certain] TV shows in the catalog are systematically newer than movies — consistent with Netflix leaning on older, licensed film libraries while investing more heavily in recent/original TV content.

### H3 — "US and Indian movies differ in average runtime"

- Mean duration: United States = 92.0 min (n=2,360), India = 126.5 min (n=927)
- Welch's t-test: **p ≈ 5.1 × 10⁻²⁰²**

**Result: statistically significant, and the effect size is large** [certain] — Indian movies on Netflix run about 34 minutes longer on average, consistent with Bollywood's traditional longer-format runtime conventions.

![Duration US vs India](images/07_duration_us_india.png)

---

## 5. Data Issues to Address in Further Analysis

1. **Director missing in 30% of rows.** Any director-level analysis (e.g., "top directors") only describes the 70% of titles with data — it is not representative of the full catalog and should be labeled as such.
2. **Country and genre are multi-valued strings, not categorical fields.** They require `.str.split(', ').explode()` before counting, and even then produce tag-level, not title-level, counts. Mixing these up is the single easiest way to misreport this dataset.
3. **Duration mixes two units by `type`** (minutes for movies, seasons for TV shows). It must never be averaged across the whole dataset without first filtering by `type`.
4. **2021 is a partial year** (data ends 2021-09-25). Any year-over-year trend chart that includes 2021 as a full year will show a fake decline.
5. **14 rows have `year_added` < `release_year`**, a logical inconsistency worth flagging (not necessarily removing) in any temporal analysis.
6. **10 missing `date_added` values and 3 missing `duration` values** are small enough (<0.2%) to drop without materially affecting results, but should be dropped explicitly and documented, not silently ignored via `.dropna()` calls scattered through the notebook.
7. **H1's significance level is borderline (p = 0.060).** This should be reported as inconclusive, not rounded up to "confirmed" — a common overreach in portfolio-style EDA writeups.

---

*Charts generated from `netflix_titles.csv` using pandas and matplotlib. Statistical tests (chi-square, Mann-Whitney U, Welch's t-test) run with `scipy.stats`.*
