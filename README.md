# ClinicalPulse
**MySQL-powered Cardiovascular, Endocrine & Mental Health analytics using real CMS Medicare data with HIPAA-compliant synthetic patients.**

**[View Full Notebook (HTML)](https://abhijeethganji99.github.io/ClinicalPulse/)** — rendered with all outputs, charts, and query results.

> **Related project:** [GlucoseIQ](../glucoseiq) — Diabetes-focused analytics using UCI Pima + Kaggle clinical data and 4 CMS Tier 1 APIs.

---

## Overview

ClinicalPulse is a full-stack clinical database system built in MySQL 8, demonstrating advanced SQL engineering across 14 tables, 4 triggers, 2 scheduled events, 3 stored procedures/functions, 3 views, and 30+ SQL techniques. The database focuses on three deeply co-morbid clinical categories — Cardiovascular, Endocrine, and Mental Health — mirroring real Medicare patient populations using CMS and NLM open data APIs.

**Why these three together?**
- 60% of diabetic Medicare patients also have hypertension
- Depression affects 30%+ of heart failure patients
- Obesity drives both cardiovascular risk and mental health outcomes

---

## Data Sources

| Source | API | What It Provides | Auth |
|--------|-----|-----------------|------|
| CMS Medicare Chronic Conditions | `data.cms.gov` | Prevalence of 18 chronic conditions by US state — real Medicare population benchmarks | Free · No login |
| NLM RxNorm | `rxnav.nlm.nih.gov/REST/drugs.json` | Drug generic names and classifications for cardiac, diabetic, and psychiatric formulary | Free · No login |

Both APIs are listed in the SLU Cross-Disciplinary Health Resources guide. Patient records are generated using Faker with HIPAA Safe Harbor compliance — no real patient identities.

### HIPAA Compliance Note

Patient-level data (names, DOBs, emails, phone numbers) is generated via Faker for learning purposes. Real data in this project is limited to:
- CMS condition prevalence percentages (aggregate, not individual)
- NLM drug names and classifications
- ICD-10 codes and CPT codes (public clinical standards)

SSNs are stored AES-encrypted (`ssn_encrypted VARBINARY(255)`) even in synthetic form, demonstrating production security patterns.

---

## Database Schema — 14 Tables

```
cardio_endo_mh_db
│
├── CORE LOOKUP
│   ├── Patients              — demographics, insurance, smoker/CVD family history flags
│   ├── Physicians            — cardiologists, endocrinologists, psychiatrists (self-referencing hierarchy)
│   ├── Diagnoses_Lookup      — 18 ICD-10 codes across 3 categories + real CMS prevalence %
│   └── Medications_Lookup    — cardiac, diabetic, psychiatric formulary (NLM-verified names)
│
├── CLINICAL EVENTS
│   ├── Allergies             — drug/food allergies with severity (critical for psych + cardiac meds)
│   ├── PatientDiagnoses      — diagnosis events with SCD Type 2 versioning
│   ├── Prescriptions         — medication orders with adherence_pct tracking
│   ├── Procedures            — cardiac caths, glucose monitoring, therapy sessions
│   ├── LabResults            — HbA1c, lipids, BNP, PHQ-9, troponin, INR, lithium levels
│   ├── VitalSigns            — BP, HR, BMI (generated column), SpO2, temperature
│   └── ClinicalNotes         — SOAP notes per category with full-text search index
│
├── RELATIONSHIPS
│   └── PatientPhysician      — bridge table: many patients ↔ many physicians
│
└── SYSTEM
    ├── AuditLog              — immutable JSON change trail (auto-populated by triggers)
    └── DeletedRecords        — soft-delete archive (auto-populated before hard deletes)
```

### Conditions Tracked

| Category | ICD-10 Codes | CMS Prevalence |
|----------|-------------|----------------|
| ❤️ Cardiovascular | Hypertension (I10), Heart Failure (I50.9), CAD (I25.10), Afib (I48.91), AMI (I21.9), Stroke (I63.9), PVD (I73.9) | 7.0% – 58.3% |
| 🩺 Endocrine | Type 2 Diabetes (E11.9), Hyperglycemia (E11.65), Hyperlipidemia (E78.5), Obesity (E66.9), Hypothyroidism (E03.9), Diabetic CKD (E11.22) | 14.8% – 49.8% |
| 🧠 Mental Health | Major Depression (F32.9), Anxiety (F41.1), Bipolar (F31.9), Schizophrenia (F20.9), Alcohol Use Disorder (F10.20) | 1.8% – 14.2% |

---

## SQL Concepts Demonstrated

### Database Objects (9 types)
| Object | Implementation |
|--------|---------------|
| Tables | 14 tables with FKs, ENUMs, composite PKs |
| Indexes | 7 composite indexes + 1 covering index + 1 full-text index |
| Stored Procedures | `admit_patient()` — transaction + error handler + OUT params; `get_patient_timeline()` — CURSOR loop into temp table |
| Stored Function | `calculate_patient_risk_score()` — DETERMINISTIC, weighted CVD/Endo/MH scoring |
| Triggers | 4 triggers — see section below |
| Events | 2 scheduled daily events |
| Views | 3 views with PII masking |
| Temporary Tables | `tmp_patient_risk_calc`, `tmp_timeline` in cursor procedure |
| Generated Column | `bmi` on VitalSigns — `703 * weight_lbs / (height_in * height_in)` |

### Triggers (4)
| Trigger | Fires On | What It Does |
|---------|----------|-------------|
| `trg_audit_diagnosis_insert` | AFTER INSERT on PatientDiagnoses | Writes JSON snapshot to AuditLog |
| `trg_audit_diagnosis_update` | AFTER UPDATE on PatientDiagnoses | Captures old + new JSON values for status/severity changes |
| `trg_flag_critical_lab` | BEFORE INSERT on LabResults | Auto-flags Critical/High/Low based on ref range thresholds |
| `trg_soft_delete_patient` | BEFORE DELETE on Patients | Archives patient JSON to DeletedRecords before hard delete |

### Scheduled Events (2)
| Event | Schedule | Action |
|-------|----------|--------|
| `evt_auto_resolve_diagnoses` | Daily | Marks past `resolved_date` diagnoses as Resolved |
| `evt_expire_prescriptions` | Daily | Sets status = Completed for past `end_date` prescriptions |

### Views (3) — Security Layer Pattern
| View | Exposes | Masks |
|------|---------|-------|
| `vw_patient_summary` | CVD/Endo/MH diagnosis counts, risk tier, prescription and procedure totals | Email (`LEFT(email,2)****@****.***`), phone (last 4 only) |
| `vw_critical_lab_alerts` | Flag=Critical or High labs with ordering physician | — |
| `vw_active_medications` | Active prescriptions with adherence %, monitoring flags | — |

### Query Techniques
```sql
-- CTEs (WITH)
WITH diagnosis_events AS (
    SELECT p.mrn, p.full_name, pd.diagnosed_date AS event_date, ...
    FROM PatientDiagnoses pd JOIN Diagnoses_Lookup dl ...
), rx_events AS (...), lab_events AS (...)
SELECT * FROM diagnosis_events
UNION ALL SELECT * FROM rx_events
UNION ALL SELECT * FROM lab_events
ORDER BY event_date

-- Recursive CTE — physician specialty hierarchy
WITH RECURSIVE physician_tree AS (
    SELECT id, full_name, specialty, supervisor_id, 0 AS lvl FROM Physicians
    WHERE supervisor_id IS NULL
    UNION ALL
    SELECT p.id, p.full_name, p.specialty, p.supervisor_id, pt.lvl+1
    FROM Physicians p JOIN physician_tree pt ON p.supervisor_id = pt.id
)
SELECT CONCAT(REPEAT('  ', lvl), full_name) AS physician, specialty FROM physician_tree

-- Window Functions — ranking labs
SELECT p.full_name, lr.test_name, lr.result_value,
    ROW_NUMBER()  OVER (ORDER BY lr.result_value DESC) AS row_num,
    RANK()        OVER (ORDER BY lr.result_value DESC) AS `rank`,
    DENSE_RANK()  OVER (ORDER BY lr.result_value DESC) AS dense_rank,
    NTILE(4)      OVER (ORDER BY lr.result_value DESC) AS quartile,
    PERCENT_RANK() OVER (ORDER BY lr.result_value)     AS pct_rank
FROM LabResults lr JOIN Patients p ON p.id = lr.patient_id

-- LAG / LEAD — lab trend tracking
SELECT lr.test_name, lr.result_value,
    LAG(lr.result_value)  OVER (PARTITION BY lr.patient_id, lr.test_name ORDER BY lr.test_date) AS prev,
    LEAD(lr.result_value) OVER (PARTITION BY lr.patient_id, lr.test_name ORDER BY lr.test_date) AS nxt,
    ROUND(lr.result_value - LAG(lr.result_value)
        OVER (PARTITION BY lr.patient_id, lr.test_name ORDER BY lr.test_date), 3) AS delta
FROM LabResults lr

-- Running total + 3-visit moving average
SELECT p.full_name, proc.procedure_date,
    SUM(proc.cost_usd) OVER (
        PARTITION BY proc.patient_id ORDER BY proc.procedure_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_cost,
    ROUND(AVG(proc.cost_usd) OVER (
        PARTITION BY proc.patient_id ORDER BY proc.procedure_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2)    AS moving_avg_3,
    FIRST_VALUE(proc.procedure_name) OVER (
        PARTITION BY proc.patient_id ORDER BY proc.procedure_date) AS first_procedure
FROM Procedures proc JOIN Patients p ON p.id = proc.patient_id

-- NTILE quintile + stored function inline
SELECT p.mrn, p.full_name,
    NTILE(5) OVER (ORDER BY calculate_patient_risk_score(p.id) DESC) AS quintile
FROM Patients WHERE is_active = 1

-- Full-Text Search on SOAP notes
SELECT p.full_name, cn.category, cn.note_date,
    MATCH(cn.full_text) AGAINST('chest pain shortness of breath' IN NATURAL LANGUAGE MODE) AS score
FROM ClinicalNotes cn JOIN Patients p ON p.id = cn.patient_id
WHERE MATCH(cn.full_text) AGAINST('chest pain shortness of breath' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC

-- Correlated subqueries — patient profile
SELECT p.id,
    (SELECT COUNT(*) FROM PatientDiagnoses pd
     JOIN Diagnoses_Lookup dl ON dl.id = pd.diagnosis_id
     WHERE pd.patient_id = p.id AND dl.category = 'Cardiovascular') AS cvd_count,
    (SELECT ROUND(AVG(adherence_pct),1) FROM Prescriptions rx
     WHERE rx.patient_id = p.id AND rx.is_active = 1)               AS avg_adherence,
    (SELECT 1 FROM PatientDiagnoses pd
     JOIN Diagnoses_Lookup dl ON dl.id = pd.diagnosis_id
     WHERE pd.patient_id = p.id AND dl.category = 'Mental Health'
     LIMIT 1)                                                         AS has_mh_dx
FROM Patients p

-- EXISTS check
-- AES encryption
INSERT INTO Patients (ssn_encrypted, ...) VALUES (AES_ENCRYPT('123-45-6789', 'key'), ...)

-- JSON columns + JSON_OBJECT in triggers
detail_json JSON  -- on LabResults table
JSON_OBJECT('patient_id', NEW.patient_id, 'status', NEW.status, 'severity', NEW.severity)

-- EXPLAIN for query optimization
EXPLAIN SELECT patient_id, test_name, result_value, flag
FROM LabResults WHERE patient_id = 1 AND test_name = 'HbA1c'

-- SCD Type 2 versioning
-- PatientDiagnoses.version INT + is_current TINYINT(1)
-- Old row: is_current=0 | New row: version+1, is_current=1
```

### Complete SQL Feature Checklist

| Category | Features Used |
|----------|--------------|
| Window Functions | ROW_NUMBER · RANK · DENSE_RANK · NTILE · LAG · LEAD · FIRST_VALUE · SUM OVER · AVG OVER · PERCENT_RANK · ROWS BETWEEN UNBOUNDED PRECEDING |
| CTEs | Standard CTEs · Recursive CTE (physician tree) · Multi-CTE UNION ALL |
| Set Operations | UNION ALL (unified clinical timeline) |
| Subqueries | Correlated subqueries · EXISTS · Scalar subqueries in SELECT |
| Schema Patterns | Star schema · Self-referencing table · Bridge table · Audit table · Soft deletes · SCD Type 2 · Polymorphic ENUM categories |
| Stored Logic | Stored procedures (CURSOR, OUT params, error handler, transactions) · Deterministic stored function · Triggers · Scheduled events |
| Security | Views as PII masking layer · AES_ENCRYPT for SSN · Column-level masking in views |
| Performance | Composite indexes · Covering index · Full-text index · EXPLAIN analysis |
| Modern MySQL 8 | JSON columns · JSON_OBJECT() · Generated/computed column (BMI) |
| DML Patterns | Bulk INSERT with executemany · Batch UPDATE · Temp tables · CASE WHEN · COALESCE |
| Search | MATCH...AGAINST full-text search on SOAP notes |

---

## Tech Stack

```
Language        Python 3.x
Database        MySQL 8 (local, bootstrapped via subprocess)
APIs            requests — CMS data.cms.gov + NLM rxnav.nlm.nih.gov
Patient Data    Faker (HIPAA Safe Harbor synthetic identities)
DB Layer        mysql-connector-python · SQLAlchemy · pymysql
Data            pandas · numpy
Visualization   matplotlib · seaborn
```

---

## Notebook Sections

| Section | Contents |
|---------|----------|
| 1 | Environment bootstrap — MySQL install, credentials, Python packages |
| 2 | Schema design — 14 tables with all constraints, indexes, generated column |
| 3 | Triggers (4) and Event Scheduler (2) |
| 4 | Stored procedures (2) and stored function (1) |
| 5 | Views (3) with PII masking and security layer pattern |
| 6 | Data loading — CMS Chronic Conditions API + NLM RxNorm API + synthetic patients |
| 7 | Advanced queries — CTEs, Recursive CTE, UNION ALL clinical timeline |
| 8 | Window functions — ranking, LAG/LEAD trends, running totals, moving averages |
| 9 | Full-text search, correlated subqueries, EXPLAIN, temp tables |
| 10 | Stored function demo — risk scores weighted by CVD/Endo/MH category |
| 11 | Visualization dashboard — 6 clinical panels + CMS prevalence comparison |
| 12 | Stored procedures in action — admit_patient(), get_patient_timeline() CURSOR |
| 13 | Executive summary report — KPIs across all 3 focus categories |
| 14 | Teardown and artifact listing |

---

## Outputs

| File | Contents |
|------|----------|
| `clinical_dashboard.png` | 6-panel chart — risk tiers, comorbidities, HbA1c, BNP, PHQ-9, lab flags |
| `cms_prevalence_comparison.png` | Our DB vs CMS national prevalence by CVD / Endocrine / Mental Health |
| `adherence_vitals.png` | Medication adherence by drug category + population BP trend |
| `cardio_endo_mh_report.txt` | Executive summary with KPIs across all categories |

---

## Setup

```bash
# Python packages
pip install mysql-connector-python sqlalchemy pymysql pandas numpy \
            matplotlib seaborn tabulate requests faker

# MySQL is installed and configured automatically in Section 1
# No Kaggle account or API key required — CMS and NLM are fully public
```

Run all cells top-to-bottom. The notebook handles MySQL install, DB creation, schema, seeding, and teardown automatically.

---

## Related Project

**GlucoseIQ** — End-to-end Diabetes Patient Analytics using real UCI Pima + Kaggle clinical data and 4 live CMS Medicare Tier 1 APIs. MySQL · Python · Scikit-learn · Plotly.
