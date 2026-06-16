#  ClinicalPulse
### MySQL-powered Cardiovascular, Endocrine & Mental Health analytics using real CMS Medicare data with HIPAA-compliant synthetic patients.

---

# 📊 Data Sources & Privacy Statement

## Cardiovascular · Endocrine · Mental Health Clinical Database System

---

## Overview

This project uses a **hybrid data strategy** — real statistical data from U.S. government sources for clinical accuracy, and fully synthetic data for all patient-level records in strict compliance with HIPAA privacy requirements.

---

## 🌐 External Data Sources (Real Statistical Data)

### 1. Centers for Medicare & Medicaid Services (CMS)
- **Source:** [data.cms.gov](https://data.cms.gov)
- **Dataset:** Medicare Specific Chronic Conditions — State-Level Prevalence
- **Access:** Public REST API — no API key, no login, no registration required
- **License:** U.S. Federal Open Data under the [OPEN Government Data Act (2018)](https://www.congress.gov/bill/115th-congress/house-bill/4174)
- **What we use it for:**
  Real national prevalence rates for chronic conditions across the Medicare population. For example:
  - Hypertension: **58.3%** of Medicare beneficiaries
  - Type 2 Diabetes: **29.7%**
  - Heart Failure: **14.1%**
  - Major Depression: **14.2%**

  These percentages are used to **statistically weight** how conditions are assigned in our simulated patient population, so the database mirrors real-world clinical distribution rather than random assignment.

- **Referenced in:** SLU Cross-Disciplinary Health Resources Guide (libguides.slu.edu)

---

### 2. National Library of Medicine (NLM) — RxNorm API
- **Source:** [rxnav.nlm.nih.gov](https://rxnav.nlm.nih.gov)
- **Dataset:** RxNorm Drug Name Terminology
- **Access:** Public REST API — no API key, no login required
- **License:** Freely available, maintained by the U.S. National Library of Medicine
- **What we use it for:**
  Fetching official, verified generic drug names (e.g., resolving "Warfarin" to its NLM-standardized terminology) to populate the `Medications_Lookup` table with clinically accurate drug information.

---

## 🔒 Patient Data — Simulated Only (HIPAA Compliance)

> **All patient-level records in this database are entirely fictional.**
> No real patient data was collected, accessed, stored, or used at any point in this project.

### Why Simulated Data?

Real patient health records are protected under the **Health Insurance Portability and Accountability Act (HIPAA)**. Accessing, storing, or using identifiable patient data requires:

- A formal **Data Use Agreement (DUA)**
- **IRB (Institutional Review Board)** approval
- **De-identification** of all 18 HIPAA identifiers
- Secure data handling infrastructure

Since this project does not have those agreements in place, **we do not use any real patient data** — and we do not need to, because our goal is to demonstrate SQL database design and clinical analytics patterns, not to analyze real individuals.

---

### Patient Data Simulation Library: `Faker`

All patient-level data is generated using the **Python `Faker` library** (`pip install faker`), a well-established open-source tool for generating realistic but entirely fictional personal data.

**What Faker generates in this project:**

| Field | Example Output | Real? |
|---|---|---|
| Patient names | "Eleanor Vasquez" | ❌ Fictional |
| Date of birth | 1968-04-22 | ❌ Fictional |
| Email addresses | eleanor@example.com | ❌ Fictional |
| Phone numbers | (555) 867-5309 | ❌ Fictional |
| City / State | Springfield, MO | ❌ Fictional |
| Insurance IDs | MCR847291 | ❌ Fictional |
| SSNs | Generated then SHA-256 hashed — never stored in plain text | ❌ Fictional + encrypted |
| MRN (Medical Record Numbers) | CEM00000001 | ❌ Fictional |

**What is NOT simulated (uses real clinical reference values):**

| Element | Source |
|---|---|
| ICD-10 diagnosis codes | Real ICD-10-CM standard codes (e.g., E11.9, I10, F32.9) |
| Drug names and classes | Real drug names verified via NLM RxNorm API |
| Lab reference ranges | Real clinical reference ranges (e.g., HbA1c normal: 4.0–5.6%) |
| Condition prevalence weights | Real CMS Medicare population statistics |
| Clinical SOAP note templates | Based on real clinical documentation patterns |

---

## 🗂️ Summary Table

| Data Type | Source | Real or Simulated |
|---|---|---|
| Condition prevalence rates | CMS data.cms.gov | ✅ Real |
| Drug generic names | NLM RxNorm API | ✅ Real |
| ICD-10 codes | ICD-10-CM standard | ✅ Real |
| Lab reference ranges | Clinical standards | ✅ Real |
| Patient names | Faker library | 🔵 Simulated |
| Patient demographics | Faker library | 🔵 Simulated |
| Patient SSNs | Faker + SHA-256 hash | 🔵 Simulated + encrypted |
| Individual lab values | Python random (clinical ranges) | 🔵 Simulated |
| Individual vital signs | Python random (clinical ranges) | 🔵 Simulated |
| Clinical notes | Template-based generation | 🔵 Simulated |
| Prescriptions | Random assignment | 🔵 Simulated |

---

## ⚖️ HIPAA Acknowledgment

This project was developed with full awareness of HIPAA patient privacy requirements. The decision to use the `Faker` library for all patient-level data was a deliberate design choice to:

1. **Eliminate any risk** of accidental exposure of real patient information
2. **Comply with HIPAA** by ensuring no Protected Health Information (PHI) was ever collected or stored
3. **Enable open sharing** of the codebase without any data use restrictions
4. **Demonstrate realistic** clinical database patterns without compromising real individuals

Any resemblance of the simulated patient names, dates, or identifiers to real persons is entirely coincidental.

---

## 📚 References

- CMS Open Data: https://data.cms.gov
- NLM RxNorm API: https://rxnav.nlm.nih.gov
- Faker Python Library: https://faker.readthedocs.io
- SLU Cross-Disciplinary Health Resources: https://libguides.slu.edu/cross-disciplinary
- OPEN Government Data Act: https://www.congress.gov/bill/115th-congress/house-bill/4174
- HIPAA Privacy Rule: https://www.hhs.gov/hipaa/for-professionals/privacy/index.html
