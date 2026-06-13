# Data: provenance, access, and schemas

This project is **open in code, partially gated in data**. SEC N-PORT filings are public;
S&P Global / Sustainable1 ESG scores are proprietary and reached through a WRDS license and
cannot be redistributed here. This document gives the source and the exact schema of every
input file so that anyone with the appropriate access can rebuild them and reproduce the
results.

> **Do not commit data to the repo.** `.gitignore` already excludes everything in `data/`
> except this README. ESG scores in particular are licensed and must not be redistributed.

Place all input files directly in `./data/` (or set `ESG_DATA_DIR` to another folder).

---

## 1. `nport_holdings_2024.csv` — public, free to rebuild

**Source.** SEC EDGAR Form **N-PORT** filings for 2024 (registered investment companies'
monthly portfolio holdings). These are public; the upstream scraper is a separate utility
(not required to run the analysis once this CSV exists).

**Grain.** One row per fund-holding (a position held by a fund series).

**Columns used by the notebook (Sections 1–6):**

| Column | Type | Notes |
|---|---|---|
| `series_name` | str | Fund series name (the unit of analysis; also drives ESG / category labels). |
| `security_name` | str | Issuer/security name (normalized to `security_name_clean` at load). |
| `pct_nav` | float | Position weight as a percentage of the fund's net assets. |
| `value_usd` | float | Position market value in USD (coerced to numeric at load). |
| `lei` | str | Legal Entity Identifier of the issuer; used for issuer-level overlap. May be blank. |
| `cusip` | str | CUSIP identifier; used in the LEI/CUSIP coverage diagnostic. May be blank. |
| `asset_category` | str | N-PORT asset category code (e.g., `EC` equity, `DBT` debt, `CASH`). |
| `issuer_category` | str | N-PORT issuer category code (e.g., `CORP`, `UST`). |
| `country` | str | Issuer country code (e.g., `US`); used for geographic features. |

---

## 2. `fund_tier_shares.csv` — requires S&P Global ESG scores (WRDS)

**Source.** Built externally by joining the N-PORT holdings to **S&P Global / Sustainable1
ESG scores** (via WRDS), assigning each rated holding to a within-industry ESG-score tier
(high / medium / low), then aggregating to fund level.

**Grain.** One row per fund series.

**Columns used (Section 8):**

| Column | Type | Notes |
|---|---|---|
| `series_id` | str | Fund series identifier (merge key with `holdings_tiered.csv`). |
| `series_name` | str | Fund series name (join key to the ESG labels). |
| `pct_high_w`, `pct_med_w`, `pct_low_w` | float (0–1) | Value-weighted shares of the rated sleeve in each ESG-score tier (sum to 1). |
| `pct_high_n`, `pct_med_n`, `pct_low_n` | float (0–1) | Count-based ("number") shares of rated positions in each tier. |
| `rated_weight` | float (0–100) | Share of the fund's NAV that is ESG-rated (used to convert sleeve shares to whole-NAV shares). |

> **Tiering convention.** `high`/`med`/`low` are *within-industry* ESG-score tiers, so that a
> fund is credited for holding strong performers relative to industry peers rather than for
> tilting toward inherently high-scoring sectors. The exact cut points are fixed in the
> upstream build step.

---

## 3. `holdings_tiered.csv` — requires S&P Global ESG scores (WRDS)

**Source.** The holding-level file behind `fund_tier_shares.csv`: N-PORT positions enriched
with each holding's ESG score and CSA industry classification.

**Grain.** One row per fund-holding (rated and unrated positions; the notebook filters to the
rated sleeve, `esg_score` present and `pct_nav > 0`).

**Columns used (Section 8):**

| Column | Type | Notes |
|---|---|---|
| `series_id` | str | Fund series identifier (merge key with `fund_tier_shares.csv`). |
| `series_name` | str | Fund series name. |
| `registrant_name` | str | Fund registrant (used in the "who passes the green test" audit). |
| `security_name` | str | Security/issuer name (matched against the curated green list). |
| `pct_nav` | float | Position weight as a percentage of net assets. |
| `esg_score` | float | S&P Global ESG score for the holding (non-missing = "rated sleeve"). |
| `csaindustryname` | str | S&P Global CSA industry name (drives the sin and neutral regexes). |

---

## 4. Generated files (written to `outputs/`, not inputs)

The notebook produces these; they are listed so the data flow is transparent. The first is
written by Section 2 and read back by Section 8 — it is the only "input" to Section 8 that
the notebook itself creates.

| File | Written by | Read by |
|---|---|---|
| `esg_net_scores.csv` | Section 2 | Section 8.1 (the `series_name` → `is_esg_final` labels) |
| `esg_funds_final.csv`, `esg_holdings.csv` | Section 2 | — |
| `horse_race_*.csv/.png`, `feature_importance.png`, `baseline_auc_comparison.csv` | Sections 6–7 | — |
| `fund_tier_shares_labeled.csv` | Section 8.1 | Sections 8.2, 8.4, 8.6 |
| `fund_involvement_labeled.csv` | Section 8.2 | Sections 8.3, 8.5 |
| `green_companies_audit.csv` | Section 8.2 | Section 8.5 (and 8.2 when `USE_AUDITED_LIST=True`) |
| `cohens_d_report_v3.csv`, `esg_tier_group_test.csv`, `*_bin_distributions.csv`, `tier_distribution_stats.csv`, `green_bin_extreme_funds.csv`, `*.png` | Section 8 | — |

---

## Identifier bridge (for rebuilding the ESG-scored files)

Matching holdings to S&P Global ESG scores on WRDS goes through an identifier bridge —
Capital IQ (CIQ) → TruCost → S&P Global ESG identifiers — to align security identifiers
across the holdings and the ratings universe. Document the exact linking-table versions you
use when you rebuild, since coverage changes across WRDS vintages.

## Licensing summary

- **SEC N-PORT** — public domain (U.S. government work).
- **S&P Global / Sustainable1 ESG scores** — proprietary; used under WRDS subscription terms;
  **not redistributed** in this repository.
