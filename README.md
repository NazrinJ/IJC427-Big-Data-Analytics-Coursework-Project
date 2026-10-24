# Big Data Analysis of ICLR Peer Review and OpenAlex Citations

Coursework project for **IJC427 Big Data Analytics**, MSc Data Science, University of Sheffield (2025–2026).

This project uses **PySpark on Databricks** to analyse peer-review data from the ICLR conference (decisions, reviewer scores, keywords) and links it to **OpenAlex** citation data. The goal is to answer six analytical questions about how review outcomes relate to reviewer scores, research topics and later citation impact.

---

## Tech Stack

| Area | Tools |
|---|---|
| Processing | PySpark (DataFrame API, `pyspark.sql.functions`) |
| Environment | Databricks Free Edition (serverless compute) |
| Format | Jupyter notebook (`.ipynb`) |

No external libraries beyond PySpark are required.

---

## Repository Structure

```
├── assignment.ipynb   # Full analysis: data cleaning, Q1–Q6 and extended analyses
└── README.md
```

---

## Data

The datasets were provided as part of the module and are **not included** in this repository.

| Dataset | Role |
|---|---|
| `iclr26v1` | Main dataset; all reported results come from this version |
| `iclr24v2`, `iclr25v2` | Used only to validate that the code runs consistently on other versions |
| OpenAlex subset | Citation counts and normalised citation percentiles, linked to ICLR papers by title |

---

## How to Run

1. Import `assignment.ipynb` into Databricks (Free Edition works).
2. Upload the datasets to your workspace.
3. Update the file paths defined at the **top of the notebook**.
4. Select **Run all**. No other setup is needed.

---

## Data Cleaning and Preparation

- Inspected the schema with `printSchema()` and previewed key columns (scores, keywords) before any processing.
- Excluded **19,673 blank decision strings**, as no outcome could be reliably assigned to them.
- Normalised inconsistent decision labels (e.g. "Accept (Poster)", "Accept: poster", "Invite to Workshop Track") into three categories, **accepted, rejected and withdrawn**, using a single reusable `label_decision` function applied across all questions and dataset versions.
- Excluded **20,289 papers with empty score arrays** from score-based questions to avoid division by zero.
- Exploded keyword arrays, removed empty strings, and lowercased and trimmed values so that variants such as "LLM" and "llm" are counted together.

Every question in the notebook lists its assumptions explicitly (e.g. strict vs. inclusive thresholds, how missing values are treated).

---

## Questions and Key Results

### Q1. Decision outcomes
**Approach:** keyword-based label normalisation with `when().otherwise()`, then `groupBy` counts.

- **36,233 papers:** 11,349 accepted, 17,290 rejected, 7,594 withdrawn (about one in three accepted).
- Mean reviewer score: **6.49** for accepted, **4.68** for rejected, **4.02** for withdrawn papers.

### Q2. Accepted papers with at least one low score
**Approach:** `array_min()` on each paper's score array, avoiding an unnecessary `explode()`.

- **1,596 accepted papers (about 14%)** received at least one score below 5.
- An extended analysis of score spread found accepted papers with strong reviewer disagreement (e.g. scores of [8, 6, 10, 1]).

### Q3. Top 10 papers by mean score
**Approach:** mean computed with the `aggregate` higher-order function (Spark has no built-in array mean), with deterministic tie-breaking.

- The top paper had a perfect mean score of 10.0; seven of the top ten are from 2025.
- **Extended analysis:** acceptance rate jumps from **19% to 80%** between mean scores of 5 and 6, and exceeds **98%** at 7 or above, so a mean of 6 acts as a practical acceptance boundary.

### Q4. Fast-rising keywords
**Approach:** yearly keyword counts, then a **self-join** on keyword and consecutive year to find terms rising from fewer than 5 to more than 50 occurrences.

- Only four keywords met the threshold, e.g. **"llm"** (4 in 2023 → 76 in 2024) and **"test-time scaling"** (3 → 87 in 2025), alongside "large reasoning models" and "spatial reasoning".
- **Extended analysis:** keywords with the highest acceptance rates were mostly theoretical topics (e.g. "reinforcement learning theory" at 55%) rather than fast-growing applied ones.

### Q5. ICLR papers matched in OpenAlex by year
**Approach:** reusable function with an inner join on title.

- Most matches came from **2020 (433)** and **2021 (423)**, dropping to 52 in 2022 and only 1–2 per year for 2024–2026, likely because OpenAlex had not yet indexed recent papers.
- **Extended analysis:** average citations per paper fall sharply for more recent years, reflecting the time papers need to accumulate citations.

### Q6. Low-scored but highly cited papers
**Approach:** join with OpenAlex, then filter for mean score below 5 and a normalised citation percentile of at least 0.95.

- **13 papers** had a mean score below 5 but ranked in the top 5% for citations in their field and year, all from 2020 or earlier.
- **Extended analysis:** median citations are similar across outcomes (23 rejected, 22 accepted, 18 withdrawn), so accepted papers do not consistently attract more citations.

---

## Limitations

- ICLR and OpenAlex are matched by **title only**, so papers with slightly different titles across sources may be missed.
- Keyword-based label normalisation could miss acceptance formats that use none of the expected terms.
- Citation counts depend on paper age and visibility, so comparisons across years should be interpreted with care.
