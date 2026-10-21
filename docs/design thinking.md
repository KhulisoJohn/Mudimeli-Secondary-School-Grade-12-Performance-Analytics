# Design Thinking: Mudimeli Secondary School Grade 12 Performance Analytics

**Baseline year:** 2026 (Terms 1 to 3 available, Term 4 and final results to be added)
**Owner:** Khuliso, Mudimeli Village, Limpopo
**Data type:** Averages only (no individual learner marks)

> Data analytics is about asking questions and providing answers. Every phase below ends with the questions we ask and the shape of the answer we expect to deliver.

---

## 0. Project Overview

**Goal:** Use Grade 12 performance averages to show which subjects are doing well, how boys and girls compare, and how this can guide learners' career choices.

**Scope:** Grade 12, 2026, per term, per class, per subject, per gender.

**Data we hold (aggregated):**

| Field | Description |
|---|---|
| year | 2026 |
| term | 1, 2, 3, 4 |
| class | e.g. 12A, 12B, 12C |
| subject | e.g. Mathematics, Physical Sciences, Accounting |
| gender | F or M |
| learners | Number of learners in that group |
| avg_pct | Average mark (%) for that group |
| pass30_pct, pass50_pct | Optional: % of learners at or above 30% and 50% |

Store only the lowest level (class x subject x gender). Calculate subject totals and school totals with **weighted averages** (sum of avg_pct x learners, divided by sum of learners). Never average averages.

---

## 1. Empathise

**Purpose:** Understand who this analysis is for and what they need to know.

**People to engage:**
- Grade 12 learners (what do they want to study or do after matric?)
- Subject teachers and HODs (what do they already suspect about performance?)
- The principal and School Management Team
- Parents and the School Governing Body
- Career guidance or life orientation teacher

**How:** Short conversations, a 5-question learner survey, and a meeting with the principal.

**Questions we ask people:**
1. Which subjects do learners find hardest, and why?
2. Do teachers believe boys and girls perform differently in some subjects?
3. How do learners currently choose subjects and careers?
4. What decisions would the school make if it had clear performance evidence?
5. What worries people about sharing performance data?

**Answers we capture:** An empathy map (says, thinks, feels, does) for a learner, a teacher and the principal, plus a list of their top needs and fears.

**Output of this phase:** Empathy notes and a list of stakeholder needs.

---

## 2. Define

**Purpose:** Turn what we heard into a clear problem and clear analytical questions.

**Problem statement (draft):**
> Grade 12 learners at Mudimeli Secondary School choose subjects and careers without clear evidence of how subjects and groups actually perform. The school needs simple, trustworthy averages to see strengths, gaps and trends.

**How might we...**
- ...show which subjects are performing well and which need support?
- ...compare boys and girls fairly?
- ...use performance patterns to guide career choices?
- ...do all of this without exposing any learner?

**Core analytics questions:**

| # | Type | Question |
|---|---|---|
| Q1 | Descriptive | How many Grade 12 learners are in each subject, split by boys and girls? |
| Q2 | Descriptive | What is the average mark per subject in each term of 2026? |
| Q3 | Descriptive | What is the average per class and per subject? |
| Q4 | Comparative | How do boys and girls compare in each subject? Where is the gap largest? |
| Q5 | Comparative | Which classes outperform others in the same subject? |
| Q6 | Trend | Is performance improving or dropping from Term 1 to Term 3? |
| Q7 | Diagnostic | Which subjects dropped most between terms, and in which classes or gender? |
| Q8 | Relationship | Do subjects move together (e.g. Maths and Physical Sciences)? |
| Q9 | Career | Which subject strengths point toward which career fields? |
| Q10 | Predictive | If Term 1 to 3 trends continue, which subjects are at risk in the final exam? |
| Q11 | Prescriptive | Where should the school and tutors focus support first? |

**Success measures:**
- Every question above has a clear, written answer backed by a table or chart.
- The principal can use the findings in a term planning meeting.
- Learners can read a one-page career guide.

**Output of this phase:** Problem statement, "How might we" list, and the question table.

---

## 3. Ideate

**Purpose:** Explore ways to answer the questions and present the answers.

**Ideas for analysis:**
- Weighted average per subject per term, with boys and girls side by side
- Heatmap: class x subject average
- Gap chart: girls average minus boys average per subject
- Term trend lines per subject
- Correlation between subjects using class-level averages
- Subject-to-career mapping (see below)

**Ideas for delivery:**
- One-page principal summary
- Teacher view per subject
- Learner-friendly career guide poster
- Interactive dashboard (Power BI, or React + ASP.NET Core API + PostgreSQL)

**Subject-to-career mapping (starter, to be refined with a career guidance teacher):**

| Strong subjects | Possible career fields |
|---|---|
| Mathematics + Physical Sciences | Engineering, electrical, IT, technicians |
| Life Sciences + Physical Sciences | Health, nursing, pharmacy, agriculture |
| Accounting + Mathematics/Maths Literacy | Finance, auditing, business |
| Geography + Life Sciences | Environmental, mining, water and resource management |
| Languages + History | Education, law, journalism, public service |

**Questions we ask while ideating:**
1. Which idea answers the most questions with the least data?
2. Which idea is simplest for a teacher to understand?
3. Which idea can run again next year with new data?

**Output of this phase:** Shortlist of analyses and delivery formats.

---

## 4. Prototype

**Purpose:** Build a small working version to answer the questions.

**Step 1: Structure the data**
- Excel or Google Sheets capture using the table in Section 0, with dropdowns for term, class, subject and gender.
- Load into PostgreSQL table `school_averages`.

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
  pass50_pct  NUMERIC(5,2)
);
```

**Step 2: Answer each question**

**Q1. Learners per subject, boys and girls**
```sql
SELECT subject,
       SUM(learners) FILTER (WHERE gender = 'F') AS girls,
       SUM(learners) FILTER (WHERE gender = 'M') AS boys,
       SUM(learners) AS total
FROM school_averages
WHERE year = 2026 AND term = 1
GROUP BY subject
ORDER BY total DESC;
```
*Answer format:* "Mathematics has X learners (Y girls, Z boys). Subject A has the most learners."

**Q2. Weighted average per subject per term**
```sql
SELECT subject, term,
       ROUND(SUM(avg_pct * learners) / SUM(learners), 1) AS avg_pct
FROM school_averages
WHERE year = 2026
GROUP BY subject, term
ORDER BY subject, term;
```
*Answer format:* "The top three subjects in Term 3 are A, B and C. The lowest three are X, Y and Z."

**Q3 and Q5. Class x subject comparison**
```sql
SELECT class, subject,
       ROUND(SUM(avg_pct * learners) / SUM(learners), 1) AS avg_pct
FROM school_averages
WHERE year = 2026 AND term = 3
GROUP BY class, subject
ORDER BY subject, avg_pct DESC;
```
*Answer format:* "In Subject A, class 12B leads by N points over 12C."

**Q4. Boys vs girls gap per subject**
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
*Answer format:* "Girls outperform boys in A and B. Boys lead in C. The largest gap is in subject D at N points."

**Q6 and Q7. Term-to-term change**
```sql
WITH t AS (
  SELECT subject, term,
         SUM(avg_pct * learners) / SUM(learners) AS avg_pct
  FROM school_averages
  WHERE year = 2026
  GROUP BY subject, term
)
SELECT subject,
       ROUND(MAX(avg_pct) FILTER (WHERE term = 3)
           - MAX(avg_pct) FILTER (WHERE term = 1), 1) AS change_t1_to_t3
FROM t
GROUP BY subject
ORDER BY change_t1_to_t3;
```
*Answer format:* "Subject A dropped N points from Term 1 to Term 3. Subject B improved by M points."

**Q8. Subject relationships (Python)**
```python
import pandas as pd
df = pd.read_sql("""SELECT class, gender, subject, avg_pct
                    FROM school_averages WHERE year=2026 AND term=3""", conn)
pivot = df.pivot_table(index=["class", "gender"], columns="subject", values="avg_pct")
print(pivot.corr().round(2))
```
*Answer format:* "Maths and Physical Sciences move together (r = 0.8). Treat this as a pattern, not proof, because the groups are few."

**Q9. Career guidance:** Take each subject's average and the mapping table in Section 3. *Answer format:* "Learners strong in Subject A and B can consider fields X and Y. Subject C is a gap to close before pursuing field Z."

**Q10. Risk outlook:** Fit a simple trend (Term 1 to Term 3) per subject and project to Term 4. *Answer format:* "At current trend, subject A is likely to finish at N%. Subjects B and C are at risk of falling below 50%." State clearly this is an estimate from only three data points.

**Q11. Support priorities:** Rank subjects by (low average) + (large drop) + (large learner count). *Answer format:* "Priority 1: Subject A (low average, many learners). Priority 2: ..."

**Step 3: Visuals**
- Bar chart: average per subject
- Clustered bars: girls vs boys per subject
- Heatmap: class x subject
- Line chart: term trend per subject

**Output of this phase:** Working database, queries, charts, and a draft answer sheet.

---

## 5. Test

**Purpose:** Check that the answers are correct, fair and useful.

**Data quality checks:**
1. Do the learner counts match the school's enrolment figures?
2. Are all averages between 0 and 100?
3. Are there missing class, subject or term rows?
4. Do weighted school averages match the school's own term analysis report?

**Fairness and accuracy checks:**
- Flag any group with fewer than 5 learners. Do not draw conclusions from it and consider combining groups.
- Remember that averages hide spread. A subject with an average of 50% can have very strong and very weak learners.
- Differences of 1 to 2 points between boys and girls may be noise, not a real gap.
- Correlation is not causation.

**Feedback sessions:**
- Present the draft answers to the principal and one or two teachers.
- Ask: Does this match what you see in class? What surprises you? What is missing? Is anything unfair or misleading?

**Questions we ask at this stage:**
1. Which answers do stakeholders trust, and which do they doubt?
2. Which findings could be misread or used to blame teachers or learners?
3. What extra data would strengthen the weakest answers?

**Output of this phase:** Corrected analysis, list of limitations, and agreed final answers.

---

## 6. Implement and Share

**Deliverables:**
1. **Principal report (1 page):** Top and bottom subjects, boys vs girls, term trends, priority support list.
2. **Teacher subject sheets:** Each teacher sees their subject by class and gender.
3. **Learner career guide:** Simple poster linking strong subjects to career fields and further study options.
4. **Dashboard:** Re-usable for Term 4 and future years.

**Ethics and privacy (POPIA):**
- Get written permission from the principal (and circuit office if required).
- Use aggregated averages only. No names, no individual marks.
- Share results for improvement and guidance, never to rank or shame individual teachers or learners.

**Next steps:**
- Add Term 4 and final NSC results when released
- Backfill earlier years (2025, 2024, and back toward 2010) to show long-term trends
- Compare final results against Term 1 to 3 to test how reliable the term averages were
- Follow up later with where learners went after matric, to sharpen career guidance

---

## Master Question and Answer Log

Fill this in as the analysis progresses.

| # | Question | Method | Answer | Confidence (high, medium, low) | Action |
|---|---|---|---|---|---|
| Q1 | Learners per subject by gender | SQL Q1 | | | |
| Q2 | Average per subject per term | SQL Q2 | | | |
| Q3 | Average per class and subject | SQL Q3 | | | |
| Q4 | Boys vs girls gap | SQL Q4 | | | |
| Q5 | Class comparison | SQL Q3 | | | |
| Q6 | Improving or dropping | SQL Q6 | | | |
| Q7 | Biggest drops and where | SQL Q6 plus class split | | | |
| Q8 | Subject relationships | Python correlation | | | |
| Q9 | Career pointers | Subject-career map | | | |
| Q10 | Final exam risk | Trend projection | | | |
| Q11 | Support priorities | Ranking | | | |
