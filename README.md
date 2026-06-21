# 🏥 Healthcare Business Intelligence Capstone

A full end-to-end Business Intelligence project analysing inpatient hospital performance across **operational**, **clinical**, and **financial** domains over a 24-month period (January 2023 – December 2024).

Built with **PostgreSQL**, **pgAdmin**, and **Tableau Desktop**, from raw data to executive-ready dashboards.

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Tech Stack](#tech-stack)
- [Project Architecture](#project-architecture)
- [Phase 1 — Database Setup](#phase-1--database-setup)
- [Phase 2 — Data Import](#phase-2--data-import)
- [Phase 3 — Data Validation](#phase-3--data-validation)
- [Phase 4 — SQL Views for Readmission Analysis](#phase-4--sql-views-for-readmission-analysis)
- [Phase 5 — Tableau Connection & Data Source Setup](#phase-5--tableau-connection--data-source-setup)
- [Phase 6 — Calculated Fields](#phase-6--calculated-fields)
- [Phase 7 — Dashboard 1: Operational Performance](#phase-7--dashboard-1-operational-performance)
- [Phase 8 — Dashboard 2: Clinical Performance](#phase-8--dashboard-2-clinical-performance)
- [Phase 9 — Dashboard 3: Financial Performance](#phase-9--dashboard-3-financial-performance)
- [Key Findings](#key-findings)
- [Recommendations](#recommendations)
- [Deliverables](#deliverables)
- [Project Reflections](#project-reflections)

---

## Project Overview

This capstone project simulates a real-world BI engagement where a tertiary hospital is experiencing operational pressure: prolonged patient stays, rising readmission rates, bed occupancy concerns, and declining revenue. The goal was to investigate these concerns using data, build interactive dashboards for executive stakeholders, and deliver evidence-based recommendations.

**Scope:** Inpatient encounters only, filtered to January 2023 – December 2024.

---

## Problem Statement

Hospital leadership raised five key concerns:

- Prolonged inpatient stays exceeding expected benchmarks
- Increasing bed occupancy pressure and reduced patient throughput
- Delayed patient turnover affecting departmental capacity
- Rising 30-day readmission rates suggesting discharge planning gaps
- Growing financial strain despite sustained patient activity

The task was to investigate these concerns using hospital data and produce dashboards that could inform executive decision-making.

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| **PostgreSQL** | Relational database for all hospital data |
| **pgAdmin** | GUI for PostgreSQL — table creation, data import, SQL execution |
| **Tableau Desktop** | Data visualisation and dashboard development |
| **SQL (PostgreSQL dialect)** | Window functions, CTEs, views for readmission analysis |

---

## Project Architecture

```
Raw CSV Files
      │
      ▼
PostgreSQL Database (hospital_data)
      │
      ├── encounters        (936,099 rows)
      ├── patients          (21,866 rows)
      ├── payers            (10 rows)
      ├── financial_summary (936,099 rows)
      └── procedures        (2,441,702 rows)
      │
      ├── vw2_readmissions_30d       (SQL View — row-level readmission flags)
      └── vw_readmission_by_diagnosis (SQL View — aggregated readmission rates)
      │
      ▼
Tableau Desktop
      │
      ├── Dashboard 1: Operational Performance
      ├── Dashboard 2: Clinical Performance
      └── Dashboard 3: Financial Performance
```

The schema follows a **star model** centred on the `encounters` table. All dimension tables (`patients`, `payers`) and fact tables (`financial_summary`, `procedures`) link back to `encounters` via `encounter_id` or `patient_id`/`payer_id`.

---

## Phase 1 — Database Setup

### Thought Process

Before touching any data, I needed a clean relational database. I chose PostgreSQL for its robustness and direct Tableau connectivity. Rather than using an intermediary tool like DBeaver, I used **pgAdmin** — the official PostgreSQL GUI — which allowed me to create tables and import data all in one place.

### Steps

1. Opened pgAdmin and created a new database named `hospital_data`
2. Opened the Query Tool and ran `CREATE TABLE` statements in dependency order to respect foreign key constraints

**Table creation order (important — foreign keys must be respected):**

```sql
-- 1. patients (no dependencies)
CREATE TABLE patients (
    id                   VARCHAR(50) PRIMARY KEY,
    birth_date           DATE,
    death_date           DATE,
    ssn                  VARCHAR(20),
    drivers              VARCHAR(20),
    passport             VARCHAR(20),
    prefix               VARCHAR(10),
    first                VARCHAR(50),
    middle               VARCHAR(50),
    last                 VARCHAR(50),
    suffix               VARCHAR(10),
    maiden               VARCHAR(50),
    marital              VARCHAR(10),
    race                 VARCHAR(50),
    ethnicity            VARCHAR(50),
    gender               VARCHAR(10),
    birthplace           VARCHAR(100),
    address              VARCHAR(100),
    city                 VARCHAR(50),
    state                VARCHAR(50),
    county               VARCHAR(50),
    fips                 VARCHAR(10),
    zip                  VARCHAR(10),
    lat                  NUMERIC(10,6),
    lon                  NUMERIC(10,6),
    healthcare_expenses  NUMERIC(12,2),
    healthcare_coverage  NUMERIC(12,2),
    income               NUMERIC(12,2)
);

-- 2. payers (no dependencies)
CREATE TABLE payers (
    id                       VARCHAR(50) PRIMARY KEY,
    name                     VARCHAR(100),
    ownership                VARCHAR(50),
    address                  VARCHAR(100),
    city                     VARCHAR(50),
    state_headquartered      VARCHAR(50),
    zip                      VARCHAR(10),
    phone                    VARCHAR(20),
    amount_covered           NUMERIC(15,2),
    amount_uncovered         NUMERIC(15,2),
    revenue                  NUMERIC(15,2),
    covered_encounters       INTEGER,
    uncovered_encounters     INTEGER,
    covered_medications      INTEGER,
    uncovered_medications    INTEGER,
    covered_procedures       INTEGER,
    uncovered_procedures     INTEGER,
    covered_immunizations    INTEGER,
    uncovered_immunizations  INTEGER,
    unique_customers         INTEGER,
    qols_avg                 NUMERIC(6,4),
    member_months            NUMERIC(10,2)
);

-- 3. encounters (references patients and payers)
CREATE TABLE encounters (
    encounter_id      VARCHAR(50) PRIMARY KEY,
    patient_id        VARCHAR(50),
    payer_id          VARCHAR(50),
    start_date        TIMESTAMP,
    stop_date         TIMESTAMP,
    encounter_class   VARCHAR(50),
    diagnosis_code    VARCHAR(20),
    diagnosis         VARCHAR(255),
    subspecialty      VARCHAR(100),
    department        VARCHAR(100),
    total_claim_cost  NUMERIC(12,2),
    payer_coverage    NUMERIC(12,2),
    FOREIGN KEY (patient_id) REFERENCES patients(id),
    FOREIGN KEY (payer_id)   REFERENCES payers(id)
);

-- 4. financial_summary (references encounters)
CREATE TABLE financial_summary (
    encounter_id         VARCHAR(50) PRIMARY KEY,
    total_charges        NUMERIC(12,2),
    insurance_payments   NUMERIC(12,2),
    patient_payments     NUMERIC(12,2),
    adjustments          NUMERIC(12,2),
    outstanding_balance  NUMERIC(12,2),
    FOREIGN KEY (encounter_id) REFERENCES encounters(encounter_id)
);

-- 5. procedures (references encounters, one-to-many)
CREATE TABLE procedures (
    encounter_id           VARCHAR(50),
    procedure_code         VARCHAR(20),
    procedure_description  VARCHAR(255),
    reasoncode             VARCHAR(20),
    procedure_diagnosis    VARCHAR(255),
    FOREIGN KEY (encounter_id) REFERENCES encounters(encounter_id)
);
```

> **Note:** `start_date` and `stop_date` were set as `TIMESTAMP` rather than `DATE` to accommodate the time components present in the source data.

---

## Phase 2 — Data Import

### Thought Process

With tables created, I imported each CSV using pgAdmin's built-in Import/Export tool. The import order follows the same dependency chain as table creation.

### Steps

For each table:
1. Right-click the table in pgAdmin → **Import/Export Data**
2. Set Format: `csv`, Header: `ON`, Delimiter: `,`
3. Browse to the source CSV file
4. Click OK

**Import order:**
1. `patients.csv`
2. `payers.csv`
3. `encounters.csv`
4. `financial_summary.csv`
5. `procedures.csv`

---

## Phase 3 — Data Validation

After importing, I ran a quick row count check to confirm all data loaded correctly:

```sql
SELECT 'patients'          AS table_name, COUNT(*) AS row_count FROM patients
UNION ALL
SELECT 'payers'            AS table_name, COUNT(*) AS row_count FROM payers
UNION ALL
SELECT 'encounters'        AS table_name, COUNT(*) AS row_count FROM encounters
UNION ALL
SELECT 'financial_summary' AS table_name, COUNT(*) AS row_count FROM financial_summary
UNION ALL
SELECT 'procedures'        AS table_name, COUNT(*) AS row_count FROM procedures
```

**Results:**

| Table | Row Count |
|-------|-----------|
| patients | 21,866 |
| payers | 10 |
| encounters | 936,099 |
| financial_summary | 936,099 |
| procedures | 2,441,702 |

✅ `encounters` and `financial_summary` match exactly, confirming one financial record per encounter.

### Encounter Class Check

To understand the scope of inpatient data (the focus of this project), I checked the encounter class distribution:

```sql
SELECT encounter_class, COUNT(*) AS count
FROM encounters
GROUP BY encounter_class
ORDER BY count DESC
```

| Encounter Class | Count |
|----------------|-------|
| ambulatory | 655,569 |
| outpatient | 164,169 |
| urgentcare | 56,213 |
| emergency | 43,536 |
| **inpatient** | **16,612** |

Only **16,612 inpatient encounters**, the rest are outpatient. All analysis was filtered to this subset.

---

## Phase 4 — SQL Views for Readmission Analysis

### Thought Process

Most charts were built directly in Tableau by connecting to raw tables. However, readmission analysis requires a **self-join** (comparing each encounter against subsequent encounters for the same patient) this can't be done natively in Tableau's drag-and-drop interface. I solved this by pre-building two PostgreSQL views using **window functions**.

I used the `LEAD()` window function instead of a self-join because it is more efficient, cleaner to read, and easier to audit.

### View 1: Row-Level Readmission Flags

This view adds a `readmission_flag` column to every inpatient encounter. A flag of `1` means the patient was readmitted within 30 days of discharge.

```sql
CREATE VIEW vw2_readmissions_30d AS
WITH inpatient_encounters AS (
    SELECT
        encounter_id,
        patient_id,
        start_date  AS admission_date,
        stop_date   AS discharge_date,
        department,
        diagnosis,
        LEAD(start_date) OVER (
            PARTITION BY patient_id
            ORDER BY start_date
        ) AS next_admission_date
    FROM encounters
    WHERE encounter_class = 'inpatient'
      AND start_date >= '2023-01-01'
      AND start_date <  '2025-01-01'
)
SELECT
    encounter_id,
    patient_id,
    department,
    diagnosis,
    admission_date,
    discharge_date,
    next_admission_date,
    next_admission_date - discharge_date AS days_to_readmission,
    CASE
        WHEN next_admission_date IS NOT NULL
         AND next_admission_date - discharge_date <= INTERVAL '30 days'
         AND next_admission_date - discharge_date >  INTERVAL '0 days'
        THEN 1
        ELSE 0
    END AS readmission_flag
FROM inpatient_encounters
```

**How `LEAD()` works here:**
- `PARTITION BY patient_id` — evaluates each patient independently
- `ORDER BY start_date` — orders their encounters chronologically
- `LEAD(start_date)` — for each encounter, looks ahead to the next encounter's start date for the same patient
- The `> INTERVAL '0 days'` condition explicitly excludes same-day readmissions

**Verification:**
```sql
SELECT
    ROUND(SUM(readmission_flag)::NUMERIC / COUNT(*) * 100, 2) AS readmission_rate_pct,
    SUM(readmission_flag) AS readmitted_encounters,
    COUNT(*)              AS total_encounters
FROM vw2_readmissions_30d
```

Result: **6.87%** readmission rate | 131 readmitted | 1,907 total

### View 2: Readmission Rate by Diagnosis

This view aggregates the row-level flags from View 1 by diagnosis:

```sql
CREATE VIEW vw_readmission_by_diagnosis AS
SELECT
    diagnosis,
    COUNT(*)                                        AS total_encounters,
    SUM(readmission_flag)                           AS readmitted_encounters,
    ROUND(
        SUM(readmission_flag)::NUMERIC
        / NULLIF(COUNT(*), 0) * 100, 2
    )                                               AS readmission_rate_pct
FROM vw2_readmissions_30d
WHERE diagnosis IS NOT NULL
GROUP BY diagnosis
HAVING COUNT(*) >= 10
ORDER BY readmission_rate_pct DESC
```

> The `HAVING COUNT(*) >= 10` threshold removes diagnoses with too few encounters to produce statistically meaningful rates.

---

## Phase 5 — Tableau Connection & Data Source Setup

### Thought Process

Rather than writing SQL queries for every chart, I connected Tableau **directly to the raw PostgreSQL tables** and built everything using Tableau's drag-and-drop interface. SQL views were only used for the two readmission charts.

### Connection Steps

1. Tableau Desktop → **Connect to a Server** → **PostgreSQL**
2. Server: `localhost`, Port: `5432`, Database: `hospital_data`
3. Dragged tables onto the canvas using **Relationships** (not joins) to avoid row duplication — especially important because `procedures` has multiple rows per encounter

**Relationships defined:**
- `encounters` → `patients` on `encounters.patient_id = patients.id`
- `encounters` → `payers` on `encounters.payer_id = payers.id`
- `encounters` → `financial_summary` on `encounters.encounter_id = financial_summary.encounter_id`
- `encounters` → `procedures` on `encounters.encounter_id = procedures.encounter_id`

### Data Source Filters Applied

Two filters were applied at the data source level so they automatically applied to all sheets:

1. `encounter_class = 'inpatient'`
2. A calculated field `In 24 Month Window`:

```
[Start Date] >= #2023-01-01# AND [Start Date] <= #2024-12-31#
```

> I initially tried Tableau's built-in relative date filter ("Last 24 months") but it was pulling December 2022 data because the dataset goes back to year 2000. The hardcoded date range was more reliable.

---

## Phase 6 — Calculated Fields

These calculated fields were created in Tableau to support the analysis:

| Field Name | Formula | Purpose |
|------------|---------|---------|
| `ALOS` | `AVG(DATEDIFF('day', [Start Date], [Stop Date]))` | Average Length of Stay in days |
| `In 24 Month Window` | `[Start Date] >= #2023-01-01# AND [Start Date] <= #2024-12-31#` | Date filter |
| `Month Year Label` | `DATENAME('month', [Start Date]) + " " + STR(YEAR([Start Date]))` | Readable month-year axis label |
| `Month Sort` | `YEAR([Start Date]) * 100 + MONTH([Start Date])` | Numeric sort key for chronological ordering |
| `Avg Daily Census` | `SUM(DATEDIFF('day', [Start Date], [Stop Date])) / COUNTD(DATETRUNC('day', [Start Date]))` | Estimated patients in hospital per day |
| `Bed Occupancy Rate` | `SUM(DATEDIFF('day', [Start Date], [Stop Date])) / (COUNTD(DATETRUNC('day', [Start Date])) * 500)` | Occupancy based on assumed 500-bed capacity |
| `Revenue per Encounter` | `SUM([Total Charges]) / COUNTD([Encounter Id])` | Average charge per inpatient encounter |
| `Readmission Rate %` | `[Readmission Rate Pct] / 100` | Converts PostgreSQL percentage to Tableau decimal for formatting |
| `Diagnosis Short` | `IF [Diagnosis] = 'Dependent drug abuse (disorder)' THEN 'Dependent Drug Abuse' ELSEIF ...` | Abbreviated diagnosis labels for chart readability |

> **Note on `Month Year Label`:** Tableau's continuous date axis was collapsing months from different years (Jan 2023 and Jan 2024 both showing as "January"). Using a string-based label with a numeric sort field solved this cleanly.

---

## Phase 7 — Dashboard 1: Operational Performance

### KPI Cards

| KPI | Value | Field |
|-----|-------|-------|
| Total Inpatient Encounters | 1,907 | `COUNTD([Encounter Id])` |
| Average ALOS | 4.9 Days | `ALOS` calculated field |
| Average Daily Census | 14.0 | `Avg Daily Census` calculated field |
| Bed Occupancy Rate | 2.79% | `Bed Occupancy Rate` calculated field |

### Charts Built

**1. Encounter & ALOS Trend (Dual-Axis)**
- Columns: `Month Year Label` (sorted by `Month Sort`)
- Rows: `COUNTD(Encounter Id)` as bars, `ALOS` as line
- Dual axis with independent scales
- Shows monthly volume and average stay length side by side

**2. Department Workload (Bar Chart)**
- Rows: `Department`
- Columns: `COUNTD(Encounter Id)`
- Colour: `ALOS` with Orange-Blue Diverging palette
- Sorted descending by encounter volume

**3. Department ALOS (Bar Chart with Reference Line)**
- Rows: `Department`
- Columns: `ALOS`
- Reference line: Constant at **4.9 days** (hospital average), labelled "Hospital Avg ALOS: 4.9 days"
- Colour: Red-Blue Diverging palette centred at 4.9 — bars above average appear red

**4. Top Procedures (Bar Chart)**
- Rows: `Procedure Description` — filtered to Top 10 by `COUNTD(Encounter Id)`
- Columns: `COUNTD(Encounter Id)`
- Colour: `ALOS` with diverging palette

---

## Phase 8 — Dashboard 2: Clinical Performance

### KPI Cards

| KPI | Value | Source |
|-----|-------|--------|
| Total Unique Patients | 1,156 | `COUNTD([Patient Id])` |
| 30-Day Readmission Rate | 6.87% | `vw2_readmissions_30d` view |
| Readmitted Encounters | 131 | `SUM([Readmission Flag])` |

### Charts Built

**5. Top Diagnoses by Volume (Bar Chart)**
- Rows: `Diagnosis Short` — Top 10 by encounter count
- Columns: `COUNTD(Encounter Id)`
- Labels showing exact counts

**6. Diagnoses vs ALOS (Scatter Plot — Quadrant Analysis)**
- Columns: `ALOS`
- Rows: `COUNTD(Encounter Id)`
- Mark size: `COUNTD(Encounter Id)` (larger circles = more encounters)
- Reference lines: Constant at ALOS = 4.9 (vertical, red dashed) and Average on Y-axis (horizontal, blue dashed)
- Quadrant annotations added manually: "High Burden Zone", "High Volume, Efficient Zone", "Complex, Rare Cases"

**7. Department ALOS vs Encounter Volume (Scatter Plot — Quadrant Analysis)**
- Same structure as above but at department level
- Labels: `Department` on Detail shelf
- Clearly shows Internal Medicine alone in the "Highest Burden Zone"

**8. Readmission Rate by Diagnosis (Bar Chart)**
- Data source: `vw_readmission_by_diagnosis` view
- Rows: `Diagnosis Short` — filtered to readmission rate > 0
- Columns: `Readmission Rate %`
- Colour: Red-Blue Diverging centred at average readmission rate
- Reference line: Average readmission rate across all diagnoses
- Non-Small Cell Lung Carcinoma (24.76%) appears prominently in deep red

---

## Phase 9 — Dashboard 3: Financial Performance

### KPI Cards

| KPI | Value | Source |
|-----|-------|--------|
| Total Charges | $43.32M | `SUM([Total Charges])` |
| Insurance Payments | $41.23M | `SUM([Insurance Payments])` |
| Patient Payments | $2.09M | `SUM([Patient Payments])` |
| Insurance % | 95.18% | Calculated field |
| Patient % | 4.82% | Calculated field |

> **Note on filters:** Financial dashboards use the same inpatient + date filters as Dashboards 1 and 2.

### Charts Built

**9. Total Inpatient Charges Trend (Line Chart)**
- Columns: `Month Year Label` (sorted by `Month Sort`)
- Rows: `SUM([Total Charges])`
- Reference line: Average monthly charges ($1.80M)
- Shows clear declining trend from $2.9M (Jan 2023) to $1.1M (Dec 2024)

**10. Insurance vs Patient Payments Trend (Dual-Axis Line Chart)**
- Insurance Payments on primary axis, Patient Payments on secondary axis
- Independent (unsynchronised) axes to show the scale difference clearly
- Average reference lines on both axes
- Custom text box legend added (Tableau auto-legend was removed for cleanliness)

**11. Department Financial Performance (Bar Chart)**
- Rows: `Department`, sorted descending by total charges
- Columns: `SUM([Total Charges])`
- Colour: `Revenue per Encounter` calculated field
- Tooltip: Insurance Payments, Patient Payments, Total Charges, Revenue per Encounter

---

## Key Findings

### Operational
- **Internal Medicine** handles **85.5%** of all inpatient encounters (1,630 of 1,907) with an ALOS of **5.2 days**, exceeding the hospital average of 4.9 days
- Only 3 departments have inpatient activity: Internal Medicine, Surgery, and O.G
- Monthly encounter volume is volatile (65–96 per month) with no clear trend

### Clinical
- **Dependent Drug Abuse** (437 encounters) and **Non-Small Cell Lung Carcinoma** (420 encounters) together account for **45% of all inpatient admissions**
- The hospital-wide **30-day readmission rate is 6.87%** (131 of 1,907 encounters)
- **Non-Small Cell Lung Carcinoma has a 24.76% readmission rate**, 1 in 4 patients returns within 30 days
- This single diagnosis accounts for **104 of 131 total readmissions (79.4%)**
- Dependent Drug Abuse has **zero readmissions** despite being the highest-volume diagnosis

### Financial
- Total inpatient charges over 24 months: **$43.32M**
- Insurance covers **95.18%** of all inpatient payments
- Monthly charges **declined 62%** from peak ($2.9M, Jan 2023) to December 2024 ($1.1M)
- Internal Medicine generates **$39.8M (91.9%)** of total inpatient charges

---

## Recommendations

| Ref | Recommendation | Priority |
|-----|---------------|----------|
| R1 | Establish Internal Medicine Capacity Review Committee | IMMEDIATE |
| R2 | Implement Structured Discharge Planning Protocols | HIGH |
| R3 | Review Oncology Procedure Scheduling (inpatient vs. day-case) | MEDIUM |
| R4 | Implement Lung Carcinoma Post-Discharge Surveillance Programme | IMMEDIATE |
| R5 | Develop Substance Abuse to Community Care Transition Pathway | MEDIUM |
| R6 | Establish Cardiac Readmission Prevention Protocols | HIGH |
| R7 | Investigate Root Cause of Declining Inpatient Charges | IMMEDIATE |
| R8 | Reduce Insurance Dependency Through Payer Diversification | MEDIUM |
| R9 | Integrate Financial & Operational Performance Monitoring Dashboard | HIGH |

---

## Deliverables

| Deliverable | Description |
|-------------|-------------|
| `hospital_data` PostgreSQL database | Five relational tables with full hospital dataset |
| `vw2_readmissions_30d` | Row-level readmission view using LEAD() window function |
| `vw_readmission_by_diagnosis` | Aggregated readmission rates by diagnosis |
| Tableau Workbook (`.twbx`) | Three dashboards — Operational, Clinical, Financial |
| Executive Report (`.docx`) | Full written analysis with findings and recommendations |
| Presentation Deck (`.pptx`) | 9-slide executive presentation |

---

## Project Reflections

### What I Learned

**On data engineering:**
- Foreign key dependency order matters when creating tables, i learnt to always create parent tables before child tables
- Tableau's Relationships model (vs. Joins) prevents row duplication when connecting one-to-many tables like `procedures`
- Window functions (`LEAD()`) are more elegant and performant than self-joins for sequential event analysis

**On Tableau:**
- Data source filters applied to "All Using This Data Source" provide a clean, consistent filter across all sheets without manual repetition
- Continuous date axes can collapse years,  a calculated string field (`Month Year Label`) paired with a numeric sort field (`Month Sort`) solves this reliably
- `financial_summary` fields weren't appearing in Measure Values because they live in a secondary table, dragging them directly from the Data pane was the workaround
- Diverging colour palettes centred at the benchmark value (e.g., 4.9 days for ALOS) are more informative than single-colour encoding

**On analysis:**
- The hospital average ALOS (4.9 days) and per-department average ALOS are different things, the overall average is weighted by volume, not a simple mean of department averages
- Readmission rate concentration matters as much as the overall rate, a 6.87% overall rate masked a 24.76% rate for a single diagnosis
- Absence of readmissions (Drug Abuse: 0%) is not necessarily good news, it may reflect disconnection from community care rather than successful treatment

### Design Decisions

- **Direct table connections over Custom SQL** for most charts - keeps the workbook maintainable and avoids Tableau's unrelated-tables join errors
- **SQL views only for readmission** — the self-referential logic required for readmission detection genuinely cannot be replicated in Tableau's data source pane
- **Hardcoded date filter** over relative date filter — more predictable behaviour with historical datasets
- **Insight-driven chart titles** over generic labels — e.g., "Internal Medicine Exceeds the Hospital Average ALOS of 4.9 Days" instead of "Department ALOS"

---

## Repository Structure

```
├── sql/
│   ├── create_tables.sql             # All five CREATE TABLE statements
│   ├── vw2_readmissions_30d.sql      # Row-level readmission view
│   └── vw_readmission_by_diagnosis.sql  # Aggregated readmission view
├── tableau/
│   └── Healthcare_BI_Capstone.twbx   # Packaged Tableau workbook
├── report/
│   └── Healthcare_BI_Executive_Report.docx
├── presentation/
│   └── Healthcare_BI_Capstone_Presentation.pptx
├── dashboards/
│   ├── dashboard_1_operational.png
│   ├── dashboard_2_clinical.png
│   └── dashboard_3_financial.png
└── README.md
```

---

*Built as part of a Healthcare Business Intelligence Capstone | January 2023 – December 2024 Analysis Window*
