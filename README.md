# USDM MDR POC

> A proof-of-concept Metadata Repository built on **CDISC USDM v4.0**, demonstrating structured digital protocol representation, Schedule of Activities, Biomedical Concept integration, and SDTM trial domain generation.

[![USDM](https://img.shields.io/badge/USDM-v4.0-blue)](https://www.cdisc.org/ddf)
[![ICH M11](https://img.shields.io/badge/ICH-M11-blue)](https://www.ich.org/page/multidisciplinary-guidelines)
[![COSMoS](https://img.shields.io/badge/COSMoS-2026--05--14-purple)](https://github.com/cdisc-org/COSMoS)
[![ODM](https://img.shields.io/badge/ODM-2.0-green)](https://www.cdisc.org/standards/data-exchange/odm)
[![SDTM IG](https://img.shields.io/badge/SDTM_IG-3.4-orange)](https://www.cdisc.org/standards/foundational/sdtmig)

---

## Overview

This POC demonstrates how a P21 Metadata Repository — currently backed by SQL with Handsontable grid views and an ODM 1.3 data model — can be extended to support CDISC USDM v4.0 as the canonical protocol layer.

**Source study:** LY900018 — EliLilly intranasal glucagon crossover study (NCT03421379), Type 2 diabetes, sourced from the [CDISC DDF-RA reference repository](https://github.com/cdisc-org/DDF-RA/tree/release-4-0-1).

---

## Files

| File | Description |
|---|---|
| [`USDM_SoA_POC.html`](USDM_SoA_POC.html) | **Interactive POC** — open in any browser, no server required |
| [`LY900018_SDTM_Trial_Domains.xlsx`](LY900018_SDTM_Trial_Domains.xlsx) | SDTM trial domains (TA/TE/TV/TI/TS) generated from USDM |
| [`SUMMARY.md`](SUMMARY.md) | Architecture overview and key design decisions |
| [`TECHNICAL_SUMMARY.md`](TECHNICAL_SUMMARY.md) | Build details, data sourcing, implementation notes |
| [`CRF_SAS_MAPPING.md`](CRF_SAS_MAPPING.md) | SAS transformation logic: COSMoS CRF Spec → P21 form hierarchy |
| [`usdm_output_V4.0 (1).json`](usdm_output_V4.0%20(1).json) | TJ301 UC study USDM v4.0 JSON (original reference) |
| `odm 2.0/` | ODM 2.0 XSD schema files |
| `USDM-IG_v4.pdf` | USDM Implementation Guide v4.0 |

---

## Quick Start

1. Open **`USDM_SoA_POC.html`** in a browser — the Schedule of Activities renders instantly with no dependencies
2. Click **Data Tables** to explore the SQL schema (loads Handsontable from CDN on first click)
3. Open **`LY900018_SDTM_Trial_Domains.xlsx`** to see the generated TA/TE/TV/TI/TS domains

---

## Architecture

```
USDM v4.0 Protocol Layer          (SoA, SAI, Epochs, Arms, StudyCells)
        ↓ Activity.biomedicalConceptIds / bcSurrogateIds
COSMoS BC Catalog                 (35 BCs + 20 BcSurrogates in LY900018)
        ↓ bc_id → crf_group_id
COSMoS CRF Specialization         (2073 rows, 16 domains — VS/EG/LB/AE/CM/DM/DS/EC…)
        ↓ crf_item → ItemDef
ODM 2.0 Data Collection           (ItemGroupDef[Type=Concept], nested IGs, VLM)
        ↓ WhereClauseDef + ValueListDef
SDTM Dataset Specialization       (VLM → Define-XML v2.1)
```

### SQL Schema (11 tables)

```sql
-- USDM (7 tables)
usdm_study, usdm_epoch, usdm_encounter, usdm_activity,
usdm_sai, usdm_sai_activity, usdm_bc_surrogate

-- COSMoS (4 tables)
cosmos_bc, cosmos_dec, cosmos_coding, activity_bc

-- ODM 2.0 (ALTER existing P21 tables)
ALTER TABLE item_group_def ADD (type, bc_id, parent_ig_oid);
ALTER TABLE item_def ADD (bc_id, dec_id, sdtm_target, value_list_oid);
CREATE TABLE odm_value_list_def (...);
CREATE TABLE odm_where_clause (...);
```

---

## POC Tabs

### 1. Schedule of Activities
ICH M11-style SoA table built from USDM data:
- **4 header rows:** Epoch band · Treatment by arm · SAI name (SAI.name drives columns, not Encounter.name) · Day (from USDM timing objects)
- **Activity grouping** in 9 protocol-defined categories
- **BC×N badge** shows number of BiomedicalConcept + BcSurrogate links per activity
- **Arm row:** Crossover sequence — LY-G gets LY900018 in Period 1, Glucagon in Period 2; G-LY reversed

### 2. Architecture Analysis
Five-layer model with callouts, SQL schema, CRF Specialization → P21 Form mapping, Activity→Form link query, and SDTM trial domain generation table.

### 3. Data Tables
15+ SQL tables as live Handsontable grids including:

| Table | Layer | Notable content |
|---|---|---|
| `usdm_sai` | USDM | 15 SAIs with `timing_day`, `timing_type` from USDM timing objects |
| `usdm_bc_surrogate` | USDM | 20 BcSurrogates (lab analytes, serology — no SDTM DS published yet) |
| `cosmos_bc` | COSMoS | 35 BCs — all SDTM DS refs; bc_code = CDISC CT TESTCD value |
| `activity_bc` | USDM/COSMoS | 56 rows; `link_type` = BC or Surrogate |
| `cosmos_crf_item` | CRF Spec | VS/EG/LB/AE/CM/DM/DS/EC examples from XLSX |
| `odm_value_list` | ODM 2.0 | ValueListDef with compound WhereClauseRef |
| `odm_where_clause` | ODM 2.0 | Compound AND conditions (VSTESTCD=SYSBP AND VSPOS=SITTING) |

### 4. Study Model
Summary cards parsed from USDM JSON.

---

## SDTM Trial Domains

Generated directly from the USDM JSON — no manual SDTM authoring:

| Domain | USDM source | Rows |
|---|---|---|
| TA — Trial Arms | StudyCells × Arm × Epoch × Element | 10 |
| TE — Trial Elements | studyDesign.elements + transition rules | 5 |
| TV — Trial Visits | SAI sequence + timing objects | 30 (15 visits × 2 arms) |
| TI — Inclusion/Exclusion | eligibilityCriteria + criterionItems | 36 |
| TS — Trial Summary | Phase, objectives, population, interventions | 23 |

**TSSEQ note (SDTM IG 3.4 §5.5):** TSSEQ resets to 1 for each TSPARMCD — it is a sequence number *within* a parameter, not a global counter. TRT×2 (LY900018, GlucaGen) → TSSEQ 1, 2. OBJSEC×3 → TSSEQ 1, 2, 3.

---

## Key Technical Points

### ScheduledActivityInstance is the SoA column header
`SAI.name` (V1, V1.1, V2…) drives column headers — **not** `Encounter.name` (E1, E2…). The SoA is a pivot of `usdm_sai_activity`: SAI.name as columns, activity.name as rows.

### BiomedicalConcept codes are CDISC CT TESTCD values
In this USDM file, all BCs reference SDTM Dataset Specializations. The `code` is the CDISC CT code for the VSTESTCD/LBTESTCD value (e.g. C49678 = RESP), not a generic NCIt concept ID.

### BcSurrogate
When no SDTM DS exists, USDM uses `BcSurrogate` with `reference = "None set"`. Activities reference surrogates via `bcSurrogateIds` alongside `biomedicalConceptIds`. 20 surrogates in LY900018 cover additional haematology (WBC, MCH, MCHC), LFTs (ALT, AST, ALP, GGT), BUN, insulin, plasma glucose, and serology.

### COSMoS CRF Specialization coverage
The CRF Specialization XLSX (`cdisc_crf_specializations_draft.csv/xlsx` from COSMoS GitHub `export/`) covers **16 domains, 2073 rows** — including VS (112), EG (243), LB (976), AE, CM, DM, DS, EC, FT, IE, MH, PR, QS, SC, SU. CSV and XLSX are identical and in sync with the CDISC Library. 32 of 35 LY900018 BCs have direct CRF Spec matches.

---

## Standards References

| Standard | Version | Role |
|---|---|---|
| [USDM](https://www.cdisc.org/ddf) | 4.0 | Protocol & SoA model |
| [ICH M11](https://www.ich.org) | — | Structured digital protocol |
| [COSMoS](https://github.com/cdisc-org/COSMoS) | 2026-05-14 | BC catalog + CRF Specialization |
| [ODM](https://www.cdisc.org/standards/data-exchange/odm) | 2.0 | Data collection definitions |
| [SDTM IG](https://www.cdisc.org/standards/foundational/sdtmig) | 3.4 | Trial domain generation |
| [CDASH IG](https://www.cdisc.org/standards/foundational/cdash) | 2.1 | CRF Specialization base standard |
