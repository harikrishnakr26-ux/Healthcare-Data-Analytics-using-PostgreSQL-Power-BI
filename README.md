# Healthcare Data Analytics using PostgreSQL + Power BI

> **End-to-end data analytics project** — from raw healthcare CSV ingestion into PostgreSQL, through advanced SQL analysis, to a 3-page interactive Power BI dashboard covering hospital operations, financial intelligence, and clinical outcomes.

---

## Table of Contents

- [Project Overview](#project-overview)
- [End-to-End Workflow](#end-to-end-workflow)
- [Dataset Description](#dataset-description)
- [Schema](#schema)
- [SQL Concepts Demonstrated](#sql-concepts-demonstrated)
- [Business Queries](#business-queries)
- [Power BI Dashboard](#power-bi-dashboard)
- [Key Insights](#key-insights)
- [Tools Used](#tools-used)
- [Future Improvements](#future-improvements)
- [Project Structure](#project-structure)

---

## Project Overview

This project simulates a real-world healthcare data analyst workflow on a hospital dataset covering patient demographics, medical conditions, billing, insurance, admissions, and clinical test results. The pipeline moves from raw data ingestion in PostgreSQL through advanced SQL analysis, culminating in a 3-page Power BI dashboard built for hospital administrators, finance teams, and clinical quality reviewers.

**What makes this resume-worthy:**
- Full pipeline: raw CSV → PostgreSQL → Power BI
- Every SQL query addresses a named healthcare business problem
- Advanced SQL: Window Functions, CTEs, `PERCENTILE_CONT`, `LAG`, `STDDEV`, conditional pivots
- Power BI dashboard with 3 analytical views — Hospital Operations, Financial & Billing, Clinical Outcomes
- Queries are production-ready, documented, and framed with business context

---

## End-to-End Workflow

```
Raw CSV Dataset (healthcare_dataset.csv)
             │
             ▼
  PostgreSQL (Ingestion + Cleaning)
             │
             ├──► Advanced SQL Queries (Business Analysis)
             │
             ▼
  Power BI (Interactive Dashboard — 3 Pages)
             │
             ├── Page 1: Hospital Analysis
             ├── Page 2: Financial & Billing Intelligence
             └── Page 3: Clinical Outcomes & Quality Metrics
```

---

## Dataset Description

| Column | Description |
|---|---|
| `name` | Patient name |
| `age` | Patient age |
| `gender` | Male / Female |
| `blood_type` | Blood group (A+, O-, etc.) |
| `medical_condition` | Primary diagnosis (Cancer, Diabetes, Obesity, etc.) |
| `date_of_admission` | Date patient was admitted |
| `doctor` | Treating doctor name |
| `hospital` | Hospital name |
| `insurance_provider` | Insurance company (Aetna, Cigna, Medicare, etc.) |
| `billing_amount` | Total billed amount (USD) |
| `room_number` | Assigned room number |
| `admission_type` | Elective / Urgent / Emergency |
| `discharge_date` | Date of discharge |
| `medication` | Prescribed medication |
| `test_results` | Outcome of diagnostic test (Normal / Abnormal / Inconclusive) |

**Source:** Kaggle — Healthcare Dataset  
**Rows:** ~40,000 patient records | **Database:** PostgreSQL | **Visualisation:** Power BI

---

## Schema

```sql
CREATE TABLE healthcare (
    name                VARCHAR(255),
    age                 INT,
    gender              VARCHAR(10),
    blood_type          VARCHAR(5),
    medical_condition   VARCHAR(100),
    date_of_admission   DATE,
    doctor              VARCHAR(255),
    hospital            VARCHAR(255),
    insurance_provider  VARCHAR(100),
    billing_amount      NUMERIC(12, 2),
    room_number         INT,
    admission_type      VARCHAR(50),
    discharge_date      DATE,
    medication          VARCHAR(100),
    test_results        VARCHAR(50)
);
```

---

## SQL Concepts Demonstrated

| Concept | Used In |
|---|---|
| Common Table Expressions (CTEs) | Q1, Q2, Q3, Q5, Q6, Q7, Q8 |
| `RANK()` / `DENSE_RANK()` / `NTILE()` | Q1, Q3, Q6 |
| `LAG()` for month-over-month comparison | Q2 |
| Running total with frame clause | Q2 |
| `PERCENTILE_CONT … WITHIN GROUP` | Q5 |
| `STDDEV()` for billing variance | Q7 |
| Conditional aggregation (pivot) | Q4, Q6 |
| `EXTRACT` / `DATE_PART` for time analysis | Q2, Q8 |
| `NULLIF` for safe division | Q1, Q3, Q6 |
| `HAVING` clause | Q7 |
| Derived columns and computed metrics | Q1, Q3, Q4, Q5 |

---

## Business Queries

---

### Q1 — High-Value Patient Identification with Billing Rank

**Business Problem:** Identify the top-spending patients per medical condition to flag high-cost cases for insurance review, care management, and resource allocation.

```sql
WITH condition_billing AS (
    SELECT
        name,
        age,
        gender,
        medical_condition,
        insurance_provider,
        billing_amount,
        RANK() OVER (
            PARTITION BY medical_condition
            ORDER BY billing_amount DESC
        )                                                  AS billing_rank,
        ROUND(AVG(billing_amount) OVER (
            PARTITION BY medical_condition
        )::numeric, 2)                                     AS avg_condition_billing,
        billing_amount - AVG(billing_amount) OVER (
            PARTITION BY medical_condition
        )                                                  AS vs_condition_avg
    FROM healthcare
)
SELECT *
FROM condition_billing
WHERE billing_rank <= 5
ORDER BY medical_condition, billing_rank;
```

**Explanation:** Uses `RANK() OVER (PARTITION BY)` to rank patients within each medical condition by billing amount, while simultaneously computing the condition-level average using `AVG() OVER ()` in the same window — allowing a delta column that shows how far above the average each high-cost patient is. This pattern mirrors what a hospital finance team would use in a cost outlier report.

---

### Q2 — Monthly Patient Admission Trend with MoM Growth

**Business Problem:** Track whether the hospital is growing or declining in monthly admissions, and quantify the month-over-month change to detect seasonal demand spikes or drops.

```sql
WITH monthly_admissions AS (
    SELECT
        DATE_TRUNC('month', date_of_admission)             AS admission_month,
        COUNT(*)                                           AS total_patients,
        ROUND(AVG(billing_amount)::numeric, 2)             AS avg_billing
    FROM healthcare
    GROUP BY DATE_TRUNC('month', date_of_admission)
),
with_growth AS (
    SELECT
        admission_month,
        total_patients,
        avg_billing,
        LAG(total_patients) OVER (ORDER BY admission_month)
                                                           AS prev_month_patients,
        total_patients - LAG(total_patients) OVER (
            ORDER BY admission_month
        )                                                  AS mom_change,
        ROUND(
            100.0 * (total_patients - LAG(total_patients) OVER (
                ORDER BY admission_month)
            ) / NULLIF(LAG(total_patients) OVER (
                ORDER BY admission_month), 0
            ), 2
        )                                                  AS mom_growth_pct
    FROM monthly_admissions
)
SELECT * FROM with_growth
ORDER BY admission_month;
```

**Explanation:** Chains two CTEs — the first aggregates monthly admissions, the second applies `LAG()` to compute month-over-month absolute change and percentage growth. The `NULLIF` prevents division-by-zero on the first row. This is a standard time-series pattern used in operational reporting dashboards.

---

### Q3 — Length of Stay Analysis with Billing Efficiency Score

**Business Problem:** Determine which patients have disproportionately long stays relative to their billing, and rank hospitals by their average care efficiency — useful for operational benchmarking.

```sql
WITH los_data AS (
    SELECT
        name,
        hospital,
        medical_condition,
        admission_type,
        date_of_admission,
        discharge_date,
        billing_amount,
        (discharge_date - date_of_admission)               AS length_of_stay,
        ROUND(
            billing_amount / NULLIF((discharge_date - date_of_admission), 0),
        2)                                                 AS billing_per_day
    FROM healthcare
),
ranked AS (
    SELECT
        *,
        RANK() OVER (
            PARTITION BY hospital
            ORDER BY length_of_stay DESC
        )                                                  AS los_rank_in_hospital,
        ROUND(AVG(length_of_stay) OVER (
            PARTITION BY medical_condition
        ), 2)                                              AS avg_los_for_condition
    FROM los_data
)
SELECT *
FROM ranked
WHERE los_rank_in_hospital <= 10
ORDER BY hospital, los_rank_in_hospital;
```

**Explanation:** Computes length of stay as a derived column from two date fields, then calculates a billing-per-day efficiency metric. `RANK() OVER (PARTITION BY hospital)` identifies the longest-stay patients within each hospital, while a second window function provides the condition-level average LOS for benchmarking — this is exactly the kind of query a hospital ops analyst would run for a bed utilisation review.

---

### Q4 — Insurance Provider Billing Pivot by Admission Type

**Business Problem:** Compare average billing amounts across all five insurance providers broken down by admission type (Elective, Emergency, Urgent) — to detect pricing inconsistencies or negotiation leverage points.

```sql
SELECT
    insurance_provider,
    ROUND(AVG(CASE WHEN admission_type = 'Elective'   THEN billing_amount END)::numeric, 0)
                                                           AS avg_elective,
    ROUND(AVG(CASE WHEN admission_type = 'Emergency'  THEN billing_amount END)::numeric, 0)
                                                           AS avg_emergency,
    ROUND(AVG(CASE WHEN admission_type = 'Urgent'     THEN billing_amount END)::numeric, 0)
                                                           AS avg_urgent,
    ROUND(AVG(billing_amount)::numeric, 0)                 AS overall_avg,
    COUNT(*)                                               AS total_patients
FROM healthcare
GROUP BY insurance_provider
ORDER BY overall_avg DESC;
```

**Explanation:** A manual pivot using conditional `AVG(CASE WHEN …)` — a transferable pattern for any cross-tabulation report without needing a pivot tool. This mirrors the matrix visual on the Power BI Financial dashboard and would be the SQL backing a finance team's insurance negotiation analysis.

---

### Q5 — Abnormal Test Result Rate by Doctor with Outlier Detection

**Business Problem:** Identify doctors with statistically abnormal rates of abnormal test results — a clinical quality metric used in physician performance reviews and accreditation audits.

```sql
WITH doctor_outcomes AS (
    SELECT
        doctor,
        COUNT(*)                                           AS total_patients,
        SUM(CASE WHEN test_results = 'Abnormal' THEN 1 ELSE 0 END)
                                                           AS abnormal_count,
        ROUND(
            100.0 * SUM(CASE WHEN test_results = 'Abnormal' THEN 1 ELSE 0 END)
            / NULLIF(COUNT(*), 0), 2
        )                                                  AS abnormal_rate_pct
    FROM healthcare
    GROUP BY doctor
    HAVING COUNT(*) >= 10
),
stats AS (
    SELECT
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY abnormal_rate_pct) AS q1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY abnormal_rate_pct) AS q3
    FROM doctor_outcomes
)
SELECT
    d.doctor,
    d.total_patients,
    d.abnormal_count,
    d.abnormal_rate_pct,
    ROUND((s.q3 + 1.5 * (s.q3 - s.q1))::numeric, 2)      AS upper_fence,
    CASE
        WHEN d.abnormal_rate_pct > s.q3 + 1.5 * (s.q3 - s.q1)
        THEN 'Flag for Review'
        ELSE 'Within Normal Range'
    END                                                    AS quality_flag
FROM doctor_outcomes d, stats s
ORDER BY d.abnormal_rate_pct DESC;
```

**Explanation:** Applies the IQR statistical outlier method using `PERCENTILE_CONT … WITHIN GROUP` — typically a Python operation — entirely in SQL. Doctors whose abnormal result rates exceed the upper IQR fence are flagged automatically. This is the kind of query that supports a Clinical Quality Management report, demonstrating both statistical reasoning and healthcare domain understanding.

---

### Q6 — Medical Condition Revenue Contribution (Pareto Analysis)

**Business Problem:** Which medical conditions generate the most hospital revenue? Quantify each condition's share of total billing to identify where to focus resource investment and insurance negotiations.

```sql
WITH condition_revenue AS (
    SELECT
        medical_condition,
        COUNT(*)                                           AS total_cases,
        ROUND(SUM(billing_amount)::numeric, 2)             AS total_revenue,
        ROUND(AVG(billing_amount)::numeric, 2)             AS avg_revenue_per_case
    FROM healthcare
    GROUP BY medical_condition
),
pareto AS (
    SELECT
        medical_condition,
        total_cases,
        total_revenue,
        avg_revenue_per_case,
        ROUND(
            100.0 * total_revenue / SUM(total_revenue) OVER (),
        2)                                                 AS revenue_share_pct,
        ROUND(
            SUM(total_revenue) OVER (
                ORDER BY total_revenue DESC
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            ) * 100.0 / SUM(total_revenue) OVER (),
        2)                                                 AS cumulative_pct,
        RANK() OVER (ORDER BY total_revenue DESC)          AS revenue_rank
    FROM condition_revenue
)
SELECT * FROM pareto
ORDER BY revenue_rank;
```

**Explanation:** Extends a basic revenue aggregation into a full Pareto analysis using `SUM() OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` for cumulative revenue percentage. The output directly supports a "which conditions drive 80% of billing?" business question — a common executive-level insight request.

---

### Q7 — Billing Variance and Anomaly Detection by Hospital

**Business Problem:** Flag hospitals with unusually high billing variability — a signal of inconsistent pricing, billing errors, or fraud risk — for finance and compliance review.

```sql
WITH hospital_billing_stats AS (
    SELECT
        hospital,
        COUNT(*)                                           AS total_patients,
        ROUND(AVG(billing_amount)::numeric, 2)             AS avg_billing,
        ROUND(STDDEV(billing_amount)::numeric, 2)          AS billing_stddev,
        ROUND(MIN(billing_amount)::numeric, 2)             AS min_billing,
        ROUND(MAX(billing_amount)::numeric, 2)             AS max_billing,
        ROUND((MAX(billing_amount) - MIN(billing_amount))::numeric, 2)
                                                           AS billing_range
    FROM healthcare
    GROUP BY hospital
    HAVING COUNT(*) >= 20
),
ranked AS (
    SELECT
        *,
        ROUND(billing_stddev / NULLIF(avg_billing, 0) * 100, 2)
                                                           AS coefficient_of_variation_pct,
        NTILE(4) OVER (ORDER BY billing_stddev DESC)       AS variance_quartile
    FROM hospital_billing_stats
)
SELECT *
FROM ranked
ORDER BY billing_stddev DESC;
```

**Explanation:** Uses `STDDEV()` to measure billing spread per hospital and computes the Coefficient of Variation (CV = stddev / mean × 100) — a normalised measure of variability that allows fair comparison across hospitals of different sizes. `NTILE(4)` buckets hospitals into variance quartiles. The top quartile by CV would be prioritised in a billing audit.

---

### Q8 — Patient Readmission Risk Cohort by Age Group and Condition

**Business Problem:** Segment patients into age cohorts and score each cohort's risk profile based on admission type, average LOS, and abnormal test rate — to guide preventive care resource allocation.

```sql
WITH patient_data AS (
    SELECT
        CASE
            WHEN age BETWEEN 0  AND 18  THEN '0–18'
            WHEN age BETWEEN 19 AND 35  THEN '19–35'
            WHEN age BETWEEN 36 AND 50  THEN '36–50'
            WHEN age BETWEEN 51 AND 65  THEN '51–65'
            ELSE '65+'
        END                                                AS age_group,
        medical_condition,
        admission_type,
        billing_amount,
        (discharge_date - date_of_admission)               AS los_days,
        test_results
    FROM healthcare
)
SELECT
    age_group,
    medical_condition,
    COUNT(*)                                               AS total_patients,
    ROUND(AVG(billing_amount)::numeric, 2)                 AS avg_billing,
    ROUND(AVG(los_days), 1)                                AS avg_los_days,
    SUM(CASE WHEN admission_type = 'Emergency'  THEN 1 ELSE 0 END)
                                                           AS emergency_admissions,
    ROUND(
        100.0 * SUM(CASE WHEN admission_type = 'Emergency' THEN 1 ELSE 0 END)
        / NULLIF(COUNT(*), 0), 2
    )                                                      AS emergency_rate_pct,
    ROUND(
        100.0 * SUM(CASE WHEN test_results = 'Abnormal' THEN 1 ELSE 0 END)
        / NULLIF(COUNT(*), 0), 2
    )                                                      AS abnormal_rate_pct,
    RANK() OVER (
        PARTITION BY age_group
        ORDER BY
            100.0 * SUM(CASE WHEN test_results = 'Abnormal' THEN 1 ELSE 0 END)
            / NULLIF(COUNT(*), 0) DESC
    )                                                      AS risk_rank_in_group
FROM patient_data
GROUP BY age_group, medical_condition
ORDER BY age_group, risk_rank_in_group;
```

**Explanation:** Combines age-based `CASE` bucketing, multi-metric conditional aggregation (emergency rate + abnormal rate), and `RANK() OVER (PARTITION BY age_group)` to rank conditions within each age cohort by clinical risk. This type of cohort risk scoring is used by hospital care management teams to prioritise which patient segments need preventive intervention programs.

---

## Power BI Dashboard

The SQL analysis feeds into a 3-page interactive Power BI dashboard, each page designed for a different stakeholder — hospital administrators, finance teams, and clinical quality officers.

---

### Page 1 — Hospital Analysis (Monthly View)

![Image Alt](https://github.com/harikrishnakr26-ux/Healthcare-Data-Analytics-using-PostgreSQL-Power-BI/blob/8e123eefe79482e07a4f4edbbc0234e1cbbc6d74/Screenshot%202026-05-18%20142108.png)

This page gives hospital administrators a bird's-eye view of operational performance. Three sparkline KPI cards track total patients (40K), average billing intensity ($25.54K), and average length of stay (15.5 days) across the full dataset. The monthly enrollment trend line chart reveals patient volume patterns from 2019 to 2024, while a year selector matrix allows quick period comparisons. Condition prevalence is broken down by gender in a horizontal bar chart, the admission type distribution (Elective 33.6%, Urgent 33.6%, Emergency 32.8%) is shown via a donut chart, and a volume-by-age-group bar chart highlights that the 65+ cohort is the largest patient segment — critical for capacity planning and staffing decisions.

---

### Page 2 — Financial & Billing Intelligence

![Image Alt](https://github.com/harikrishnakr26-ux/Healthcare-Data-Analytics-using-PostgreSQL-Power-BI/blob/a8cd761a900ee8cb32290dd51a1388cde6b0f9cb/Screenshot%202026-05-18%20142639.png)

This page is built for finance teams and insurance analysts. Four headline KPIs anchor the view: total revenue of **$23.56M**, Arthritis as the highest-revenue condition for the selected period, Johnson Group as the top-billing hospital, and an insured-vs-cash ratio of **20.02%**. A bar chart ranks hospitals by total revenue with Johnson Group leading, and a scatter plot maps average LOS days against average billing per medical condition — exposing which conditions generate higher costs relative to stay duration. The insurance provider treemap and cross-tab matrix reveal notable billing variation: Cigna's Emergency admissions reach **$29,954** while UnitedHealthcare's Urgent tier sits at just **$21,987**, a spread that flags potential contract renegotiation opportunities. A medication cost bar chart on the right shows Aspirin and Penicillin as the most frequently prescribed medications for the filtered period.

---

### Page 3 — Clinical Outcomes & Quality Metrics

![Image Alt](https://github.com/harikrishnakr26-ux/Healthcare-Data-Analytics-using-PostgreSQL-Power-BI/blob/a8cd761a900ee8cb32290dd51a1388cde6b0f9cb/Screenshot%202026-05-18%20142652.png)

Designed for clinical quality officers and medical directors, this page focuses on test outcomes and physician performance. Four KPI cards open the view: Abnormal Results **34.2%**, Normal Results **33.8%**, Inconclusive **32.1%**, and a doctor count of **835** for the selected month. A stacked bar chart breaks down test result distribution across all six medical conditions, while a patient outcomes matrix cross-references age group against result type — the 65+ cohort records the highest abnormal count (84) for the filtered period. The doctor performance table benchmarks individual physicians across total patients, average billing, average LOS days, and abnormal percentage, with Jennifer Johnson and Michael Hernandez showing a 100% abnormal rate on their small caseloads, flagging them for quality review. Blood type distribution is shown via a donut chart (spread across 8 types ranging from 11.26% to 13.98%), admissions by day of week reveal relatively even weekday-to-weekend volume, and patients by room range bars show consistent distribution across the 101–500 room band.

---

## Key Insights

- Obesity generates the highest overall revenue across the dataset, making chronic-condition treatments a major healthcare revenue driver.

- Total healthcare revenue exceeds **$1.42B**, with Johnson PLC emerging as the top-performing hospital overall.

- Admission types are distributed almost equally across Emergency, Urgent, and Elective categories, indicating balanced operational demand across hospitals.

- Clinical test outcomes remain evenly split between Abnormal, Normal, and Inconclusive categories, reflecting a diverse patient health profile.

- The 65+ age group records the highest abnormal test counts, highlighting increased healthcare risk among elderly patients.

- Average billing across medical conditions remains relatively stable (~$25K–$26K), suggesting standardized treatment-cost structures.

- Average patient length of stay is approximately 15 days, though longer stays do not always correspond to higher billing amounts.

- Insurance providers contribute revenue relatively evenly, with Cigna, Medicare, Aetna, Blue Cross, and UnitedHealthcare all showing strong participation.

- Medication usage remains balanced across major drug categories, indicating consistent prescription trends across patients.

- Doctor-level abnormal-result percentages vary significantly, enabling provider-level performance benchmarking and operational analysis.

- Patient admissions and room occupancy remain consistently distributed throughout the week, indicating stable hospital utilization patterns.

---

## Tools Used

| Tool | Purpose |
|---|---|
| PostgreSQL | Data storage, cleaning, advanced SQL analysis |
| pgAdmin / DBeaver | Query execution and exploration |
| Power BI Desktop | Interactive 3-page dashboard |
| Excel | Initial data inspection and profiling |
| GitHub | Version control and portfolio hosting |

---

## Future Improvements

- Add a Python EDA notebook (pandas + matplotlib/seaborn) as a companion analysis layer
- Build an automated data ingestion script (CSV → PostgreSQL via Python)
- Publish the Power BI dashboard to Power BI Service for live web access
- Add a predictive readmission risk model using logistic regression (Python/scikit-learn)
- Extend with DAX measures for rolling 30/90-day metrics in Power BI

---

## Project Structure

```
healthcare-data-analytics/
│
├── data/
│   └── healthcare_dataset.csv
│
├── sql/
│   ├── schema.sql
│   └── queries.sql
│
├── assets/
│   ├── dashboard_hospital_analysis.png
│   ├── dashboard_financial_billing.png
│   └── dashboard_clinical_outcomes.png
│
├── powerbi/
│   └── healthcare_dashboard.pbix
│
└── README.md
```

---

## Author

**Harikrishna K R**  
Aspiring Data Analyst | SQL · PostgreSQL · Power BI · Python  
[GitHub Profile](https://github.com/harikrishnakr26-ux)
