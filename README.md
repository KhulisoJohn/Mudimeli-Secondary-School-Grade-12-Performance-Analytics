# Mudimeli Grade 12 Performance Analytics

Turning Grade 12 average marks from Mudimeli Secondary School into clear answers: which subjects perform well, how boys and girls compare, and what this means for career choices.

**Baseline year:** 2026 (Terms 1 to 3, Term 4 to follow)
**Data:** Averages only. No learner names, no individual marks.
**Approach:** Design Thinking (see `docs/design-thinking.md`), driven by questions and answers.

---

## Questions This Project Answers

| # | Question |
|---|---|
| Q1 | How many learners take each subject, and how many are boys and girls? |
| Q2 | What is the average mark per subject, per term? |
| Q3 | How do classes compare within each subject? |
| Q4 | How do boys and girls compare, and where is the gap largest? |
| Q5 | Which classes outperform others in the same subject? |
| Q6 | Is performance improving or dropping from term to term? |
| Q7 | Which subjects dropped the most, and for which class or gender? |
| Q8 | Which subjects move together (e.g. Maths and Physical Sciences)? |
| Q9 | Which strong subjects point toward which career fields? |
| Q10 | Which subjects are at risk of a weak final result? |
| Q11 | Where should support be focused first? |

---

## Tech Stack

| Tool | Used for |
|---|---|
| Excel | Data capture, pivot tables, quick charts |
| PostgreSQL | Storage, quality checks, analysis queries |
| Python (pandas, SQLAlchemy, matplotlib, seaborn) | Cleaning, loading, analysis, charts |
| Jupyter Notebook | Exploring and documenting the analysis |
| Power BI or React + ASP.NET Core | Dashboard (optional, later phase) |
| Git and GitHub | Version control (code only, never raw data) |

---

## Project Structure

```
mudimeli-grade12-analytics/
├── README.md
├── .gitignore
├── .env.example                  # DATABASE_URL=postgresql://user:pass@localhost:5432/school_analytics
│
├── docs/
│   ├── design-thinking.md        # Empathise, Define, Ideate, Prototype, Test, Implement
│   ├── data-dictionary.md        # Every column explained
│   ├── permission-and-popia.md   # Permission letter, privacy rules
│   └── findings.md               # Question and answer log
│
├── data/
│   ├── raw/                      # Files exactly as received (NEVER committed)
│   ├── interim/                  # Partly cleaned files
│   └── processed/
│       └── school_averages_clean.csv
│
├── excel/
│   ├── 01_capture_template.xlsx  # Data entry with dropdowns
│   └── 02_analysis_workbook.xlsx # Pivot tables and charts
│
├── sql/
│   ├── 01_create_tables.sql
│   ├── 02_quality_checks.sql
│   ├── 03_analysis_queries.sql   # One query per question (Q1 to Q11)
│   └── 04_views.sql
│
├── python/
│   ├── requirements.txt
│   ├── 01_clean_data.py
│   ├── 02_load_to_postgres.py
│   ├── 03_analysis.py
│   └── 04_charts.py
│
├── notebooks/
│   └── grade12_analysis.ipynb
│
├── outputs/
│   ├── charts/
│   ├── tables/
│   └── reports/                  # Principal summary, teacher sheets, career guide
│
└── dashboard/                    # Power BI file or web app (later)
```

---

## Data Model

One row per class x subject x gender x term.

| Column | Type | Example |
|---|---|---|
| year | INT | 2026 |
| term | INT (1 to 4) | 3 |
| class | TEXT | 12A |
| subject | TEXT | Mathematics |
| gender | CHAR (F or M) | F |
| learners | INT | 14 |
| avg_pct | NUMERIC | 52.4 |
| pass30_pct | NUMERIC (optional) | 85.7 |
| pass50_pct | NUMERIC (optional) | 42.9 |

**Rule:** Never average averages. Combine groups with a weighted average: `SUM(avg_pct x learners) / SUM(learners)`.

---

## Step-by-Step Guide

### Step 1. Permission and setup

1. Get written permission from the principal (and circuit office if required). Save a copy in `docs/permission-and-popia.md`.
2. Request averages only: per year, term, class, subject and gender, with learner counts.
3. Create the folder structure above and run `git init`.
4. Add this to `.gitignore` so no school data reaches GitHub:
   ```
   data/raw/
   data/interim/
   data/processed/
   .env
   ```

### Step 2. Capture the data in Excel

1. Open `excel/01_capture_template.xlsx` and create a sheet named **Data** with the columns from the Data Model.
2. Turn it into a table (select data, press `Ctrl+T`) and name it `tblAverages`.
3. Create a sheet named **Lists** holding valid values for term, class, subject and gender.
4. Add dropdowns: Data tab, Data Validation, List, pointing to the Lists sheet. Add number rules too: `avg_pct` between 0 and 100, `learners` whole number of 0 or more.
5. Capture or paste the 2026 averages. Save a copy of what you received in `data/raw/`.

### Step 3. Quick answers in Excel (first look)

Use `excel/02_analysis_workbook.xlsx`.

1. Add a helper column `weighted = avg_pct * learners` to the table.
2. Insert a PivotTable from `tblAverages`.
3. **Q1:** Rows = subject, Columns = gender, Values = Sum of learners.
4. **Q2:** Rows = subject, Columns = term, Values = Sum of weighted and Sum of learners. Add a Calculated Field `avg = weighted / learners`.
5. **Q3 and Q5:** Rows = class, Columns = subject, Values = avg. Apply Conditional Formatting, Color Scales, for a heatmap.
6. **Q4:** Rows = subject, Columns = gender, Values = avg. Add a column for the gap (girls minus boys).
7. Insert charts: clustered bar for boys vs girls, line chart for term trend.

### Step 4. Clean the data with Python

`python/requirements.txt`:
```
pandas
openpyxl
sqlalchemy
psycopg2-binary
python-dotenv
matplotlib
seaborn
jupyter
```

`python/01_clean_data.py`:
```python
import pandas as pd
from pathlib import Path

RAW = Path("data/raw/school_averages_2026.xlsx")
OUT = Path("data/processed/school_averages_clean.csv")

df = pd.read_excel(RAW, sheet_name="Data")
df.columns = df.columns.str.strip().str.lower()

df["subject"] = df["subject"].str.strip().str.title()
df["class"] = df["class"].str.strip().str.upper()
df["gender"] = df["gender"].str.strip().str.upper().str[0]

required = ["year", "term", "class", "subject", "gender", "learners", "avg_pct"]
df = df.dropna(subset=required)

assert df["avg_pct"].between(0, 100).all(), "avg_pct outside 0-100"
assert df["gender"].isin(["F", "M"]).all(), "unexpected gender value"
assert df["term"].isin([1, 2, 3, 4]).all(), "unexpected term"
assert df.duplicated(["year", "term", "class", "subject", "gender"]).sum() == 0, "duplicate rows"

OUT.parent.mkdir(parents=True, exist_ok=True)
df.to_csv(OUT, index=False)
print(f"Saved {len(df)} clean rows to {OUT}")
```

### Step 5. Create the database and load the data

`sql/01_create_tables.sql`:
```sql
CREATE TABLE school_averages (
  year        INT NOT NULL,
  term        INT NOT NULL CHECK (term BETWEEN 1 AND 4),
  class       TEXT NOT NULL,
  subject     TEXT NOT NULL,
  gender      CHAR(1) NOT NULL CHECK (gender IN ('F','M')),
  learners    INT NOT NULL CHECK (learners >= 0),
  avg_pct     NUMERIC(5,2) NOT NULL CHECK (avg_pct BETWEEN 0 AND 100),
  pass30_pct  NUMERIC(5,2),
  pass50_pct  NUMERIC(5,2),
  PRIMARY KEY (year, term, class, subject, gender)
);
```

`python/02_load_to_postgres.py`:
```python
import os
import pandas as pd
from sqlalchemy import create_engine
from dotenv import load_dotenv

load_dotenv()
engine = create_engine(os.environ["DATABASE_URL"])

df = pd.read_csv("data/processed/school_averages_clean.csv")
df.to_sql("school_averages", engine, if_exists="append", index=False)
print(f"Loaded {len(df)} rows")
```

### Step 6. Quality checks in SQL

`sql/02_quality_checks.sql`:
```sql
-- Small groups (weak evidence, possible privacy risk)
SELECT * FROM school_averages WHERE learners < 5;

-- Learner totals per subject per term (compare with school enrolment)
SELECT term, subject, SUM(learners) AS total
FROM school_averages
GROUP BY term, subject
ORDER BY term, subject;

-- Missing gender in a class-subject-term combination
SELECT year, term, class, subject, COUNT(DISTINCT gender) AS genders
FROM school_averages
GROUP BY year, term, class, subject
HAVING COUNT(DISTINCT gender) < 2;
```
Compare the weighted school averages against the school's own term analysis report. If they do not match, find out why before going further.

### Step 7. Answer the questions with SQL

Save one query per question in `sql/03_analysis_queries.sql`. The full queries for Q1 to Q6 are in `docs/design-thinking.md`. Example for Q4 (boys vs girls):

```sql
SELECT subject,
  ROUND(SUM(avg_pct*learners) FILTER (WHERE gender='F')
        / NULLIF(SUM(learners) FILTER (WHERE gender='F'),0), 1) AS girls_avg,
  ROUND(SUM(avg_pct*learners) FILTER (WHERE gender='M')
        / NULLIF(SUM(learners) FILTER (WHERE gender='M'),0), 1) AS boys_avg
FROM school_averages
WHERE year = 2026 AND term = 3
GROUP BY subject;
```

Create reusable views in `sql/04_views.sql`, for example `v_subject_term_avg`, so Python, Excel and Power BI all read the same numbers.

### Step 8. Deeper analysis and charts in Python

`python/03_analysis.py`:
```python
import os
import pandas as pd
from sqlalchemy import create_engine
from dotenv import load_dotenv

load_dotenv()
engine = create_engine(os.environ["DATABASE_URL"])
df = pd.read_sql("SELECT * FROM school_averages WHERE year = 2026", engine)
df["w"] = df["avg_pct"] * df["learners"]

# Q2: weighted average per subject per term
g = df.groupby(["subject", "term"])[["w", "learners"]].sum()
g["avg"] = g["w"] / g["learners"]
subject_term = g["avg"].unstack("term").round(1)

# Q4: boys vs girls (Term 3)
t3 = df[df["term"] == 3]
gg = t3.groupby(["subject", "gender"])[["w", "learners"]].sum()
gg["avg"] = gg["w"] / gg["learners"]
gender_gap = gg["avg"].unstack("gender")
gender_gap["gap_F_minus_M"] = (gender_gap["F"] - gender_gap["M"]).round(1)

# Q6: change from Term 1 to Term 3
subject_term["change_t1_t3"] = (subject_term[3] - subject_term[1]).round(1)

# Q8: subject relationships across class and gender groups
pivot = t3.pivot_table(index=["class", "gender"], columns="subject", values="avg_pct")
correlations = pivot.corr().round(2)

subject_term.to_csv("outputs/tables/subject_term_avg.csv")
gender_gap.to_csv("outputs/tables/gender_gap.csv")
correlations.to_csv("outputs/tables/subject_correlations.csv")
```

`python/04_charts.py` (starter):
```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

gap = pd.read_csv("outputs/tables/gender_gap.csv", index_col=0)
gap[["F", "M"]].plot(kind="bar", figsize=(10, 5))
plt.title("Average mark by subject: girls vs boys (2026 Term 3)")
plt.ylabel("Average %")
plt.tight_layout()
plt.savefig("outputs/charts/gender_by_subject.png", dpi=150)

st = pd.read_csv("outputs/tables/subject_term_avg.csv", index_col=0)
sns.heatmap(st[["1", "2", "3"]], annot=True, fmt=".1f", cmap="RdYlGn")
plt.title("Subject averages by term (2026)")
plt.tight_layout()
plt.savefig("outputs/charts/subject_term_heatmap.png", dpi=150)
```

Use `notebooks/grade12_analysis.ipynb` to explore, and to write each answer in plain words next to its chart.

### Step 9. Record the answers

For every question, fill in `docs/findings.md`:

| Question | Answer (plain language) | Evidence (table or chart) | Confidence | Recommended action |
|---|---|---|---|---|

Be honest about limits: three terms is a short trend, small groups are weak evidence, and correlation is not causation.

### Step 10. Share the results

1. **Principal report (1 page):** top and bottom subjects, boys vs girls, term trends, support priorities.
2. **Teacher sheets:** one per subject, split by class and gender.
3. **Career guide poster:** strong subjects linked to career fields and study options.
4. **Dashboard (optional):** connect Power BI to PostgreSQL, or build a React + ASP.NET Core API reading the SQL views.

Present findings to the principal and teachers first, and ask whether they match what they see in class. Adjust before sharing wider.

### Step 11. Repeat and grow

- Add Term 4 and final NSC results when released.
- Backfill earlier years (2025, 2024 and back toward 2010) to show long-term trends.
- Compare Term 1 to 3 averages with final results to test how well the term marks predicted outcomes.
- Later, follow up on where learners went after matric to sharpen the career guidance.

---

## Ethics and Privacy (POPIA)

- Written permission from the school before any data is used.
- Averages only. No names, no individual marks.
- Groups with fewer than 5 learners are flagged and not used for conclusions.
- Raw data stays off GitHub.
- Results are for support and guidance, never to rank or shame teachers or learners.

---

## Roadmap

- [x] Design Thinking document
- [ ] Capture 2026 Terms 1 to 3
- [ ] Quality checks complete
- [ ] Q1 to Q11 answered and logged
- [ ] Principal report and career guide
- [ ] Term 4 and NSC results added
- [ ] Earlier years backfilled
- [ ] Dashboard

---

## Author

Khuliso, Mudimeli Village, Limpopo, South Africa
