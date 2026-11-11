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
| [`USDM_SoA_POC_v2.html`](USDM_SoA_POC_v2.html) | **v2 POC** — multi-study (OSB, Faro, Alexion, devices) + TF-IDF BC candidate matching |
| [`USDM_Workbench_Pitch.html`](USDM_Workbench_Pitch.html) | Product pitch deck for a USDM-native workbench built on P21 |
| [`LY900018_SDTM_Trial_Domains.xlsx`](LY900018_SDTM_Trial_Domains.xlsx) | SDTM trial domains (TA/TE/TV/TI/TS) generated from USDM |
| [`SUMMARY.md`](SUMMARY.md) | Architecture overview and key design decisions |
| [`TECHNICAL_SUMMARY.md`](TECHNICAL_SUMMARY.md) | Build details, data sourcing, implementation notes |
| [`CRF_SAS_MAPPING.md`](CRF_SAS_MAPPING.md) | SAS transformation logic: COSMoS CRF Spec → P21 form hierarchy |
| [`Faro_Health_USDM_Assessment.md`](Faro_Health_USDM_Assessment.md) | Assessment of Faro Health as a USDM authoring platform — data quality, BC coverage, Certara fit |
| [`USDM_Mapping_Approaches_Summary.md`](USDM_Mapping_Approaches_Summary.md) | USDM → CRF/SDTM mapping approaches, risks, and conceptual boundaries |
| [`USDM_Procedure_Mapping_UMLS.md`](USDM_Procedure_Mapping_UMLS.md) | Procedure → COSMoS BC mapping via UMLS: pipeline, licensing (CPT/AMA), and limitations |
| [`COSMoS_BC_MultiSpecialization_FHIR_Research.md`](COSMoS_BC_MultiSpecialization_FHIR_Research.md) | Research: COSMoS BC multi-specialization (CRF/SDTM/ADaM/FHIR) and FHIR bridge architecture |
| [`bc_id_enriched_p21_summary.md`](bc_id_enriched_p21_summary.md) | Using BC IDs and enriched P21 master standards to generate study-level outputs |
| [`DDF USDM Model Informative.png`](<DDF%20USDM%20Model%20Informative.png>) | USDM informative model diagram (DDF reference) |
| [`USDM_UML.png`](USDM_UML.png) | USDM v4.0 UML class diagram |
| [`cdisc_sdtm_dataset_specializations_latest.xlsx`](cdisc_sdtm_dataset_specializations_latest.xlsx) | CDISC SDTM Dataset Specializations (2026-07-14, 1475 specializations) — source for cosmos_sdtm_item |
| [`cdisc_crf_specializations_draft.xlsx`](cdisc_crf_specializations_draft.xlsx) | CDISC CRF Specializations (2026-05-14, 303 specializations) — source for cosmos_crf_item |
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
COSMoS CRF Specialization         (199 study-BC rows; dec_id, value_list, codelist_submission_value; source: cdisc_crf_specializations_draft.xlsx)
        ↓ crf_item → ItemDef
ODM 2.0 Data Collection           (ItemGroupDef[Type=Concept], nested IGs, VLM)
        ↓ WhereClauseDef + ValueListDef
SDTM Dataset Specialization       (375 study-BC rows; sdtm_variable = actual SDTM var; source: cdisc_sdtm_dataset_specializations_latest.xlsx → Define-XML v2.1)
```

### SQL Schema (18 tables)

```sql
-- USDM Layer 1 (7 tables)
usdm_study,      -- id, name, usdm_version, system_name, trial_phase, therapeuticAreas
usdm_epoch, usdm_encounter, usdm_activity,
usdm_sai, usdm_sai_activity, usdm_bc_surrogate

-- COSMoS Layer 2 (4 tables)
cosmos_bc, cosmos_dec, cosmos_coding, activity_bc

-- Layer 2→3 Mapping views (3 derived tables)
bc_mapping,       -- 1 row per BC; sdtm_vlm_group_id(s) + crf_group_id(s) + status
bc_crf_mapping,   -- BC → CRF: crf_grp_count as "N groups / N items"
bc_sdtm_mapping,  -- BC → SDTM: vlm_grp_count as "N groups / N items"

-- Specialization Layer 3 (2 tables — authoritative CDISC sources)
cosmos_crf_item,  -- 199 rows from cdisc_crf_specializations_draft.xlsx; 20 cols incl. dec_id
cosmos_sdtm_item, -- 375 rows from cdisc_sdtm_dataset_specializations_latest.xlsx; 18 cols; sdtm_variable = actual SDTM variable

-- ODM 2.0 Layer 4 (ALTER existing P21 tables)
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
| `cosmos_crf_item` | CRF Spec | 199 rows from CDISC CRF Specializations (2026-05-14); cols include dec_id, codelist_submission_value, value_list |
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

### PK timepoints are in the USDM — on sub-timelines

The **Treatment Phase** sub-timeline contains 15 minute-level SAIs (predose → 5min → 10min → 15min → ... → 240min) with PK (Glucagon), Plasma Glucose, VS and ECG assessed at each. The **Hypoinduction** sub-timeline has 5 SAIs for the insulin clamp procedure.

In a complete USDM instance these would be linked from Main Timeline SAIs via `SAI.sub_timeline_id`. In this file that field is null — sub-timelines exist as standalone objects. The Main Timeline SoA (shown in the POC) is correct for a protocol document; PK timepoints would appear in a separate sampling schedule derived from the Treatment Phase timeline.

### COSMoS CRF Specialization coverage
The CRF Specialization XLSX (`cdisc_crf_specializations_draft.xlsx` from COSMoS GitHub) covers **16 domains, 2073 rows total**. The POC filters to 199 rows for the 12 study BCs. 32 of 35 LY900018 BCs have direct CRF Spec matches. cosmos_crf_item detail panel shows: crf_group_id, crf_item, dec_id, prompt, question_text, data_type, length, significant_digits, mandatory, codelist, codelist_submission_value, value_list, sdtm_target_variable.

### SDTM Dataset Specialization coverage
The SDTM Dataset Specializations XLSX (`cdisc_sdtm_dataset_specializations_latest.xlsx`, 2026-07-14) covers 1475 unique specializations, 13922 rows total. The POC filters to 375 rows for the 12 study BCs. cosmos_sdtm_item `sdtm_variable` now contains the actual SDTM variable name (VSORRES, AETERM, LBORRES, VSTESTCD, etc.) sourced directly from this file. Detail panel shows: vlm_group_id, sdtm_variable, dec_id, codelist, codelist_submission_value, value_list, assigned_value, data_type, length, mandatory_variable, mandatory_value.

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
