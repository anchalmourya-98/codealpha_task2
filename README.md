# Netflix Titles — Exploratory Data Analysis

Exploratory analysis of Netflix's catalog (8,807 titles) using Python, pandas, and matplotlib. The goal was to move past surface-level `.value_counts()` output and actually test whether the patterns in the data hold up statistically — including where they don't.

## Dataset

8,807 rows × 12 columns — title, type (Movie/TV Show), director, cast, country, date added, release year, rating, duration, genre(s), description.

## Approach

The full report follows five steps:

1. **Questions asked before touching the data** — e.g. is the catalog movie- or TV-heavy, has the content mix gotten more mature over time, do movies from different countries actually differ in runtime?
2. **Data structure** — column types, null counts per column, and two structural issues that shape everything downstream: `country` and `listed_in` are comma-separated multi-label fields, and `duration` mixes units (minutes for movies, seasons for TV shows).
3. **Trends and anomalies** — catalog composition, growth by year added, top genres/countries, rating distribution, and a check for rows where `year_added` is earlier than `release_year`.
4. **Hypothesis testing** — three tests run with `scipy.stats`:
   - Chi-square test on whether mature-rated content increased 2015–2020 (result: p = 0.06, not significant at α = 0.05 — reported as inconclusive, not rounded up)
   - Mann-Whitney U test on release-year distribution, Movies vs TV Shows (p ≈ 2×10⁻¹⁰⁵, significant)
   - Welch's t-test on movie runtime, US vs India (p ≈ 5×10⁻²⁰², significant — Indian movies average ~34 minutes longer)
5. **Data issues to flag** — 30% missing `director` values, the multi-label country/genre fields, the mixed-unit duration field, a partial 2021 (data cuts off Sept 25, 2021 — don't read a 2021 dip as a real decline), and 14 logically inconsistent `year_added`/`release_year` rows.

See [`netflix_analysis_report.md`](./netflix_analysis_report.md) for the full analysis.

## Tools

Python, pandas, matplotlib, scipy.stats, Jupyter Notebook.

## Notes for reviewers

Genre and country counts in this dataset are tag-appearance counts, not title counts — a title with `country = "United States, India"` is counted in both buckets. That distinction is called out explicitly wherever those fields are used.
