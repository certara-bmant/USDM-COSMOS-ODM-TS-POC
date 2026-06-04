# USDM MDR POC — Summary

## Overview

A proof-of-concept (POC) for a P21 Metadata Repository built on CDISC USDM v4.0, demonstrating how to represent an ICH M11-compliant Structured Digital Protocol. The POC uses **LY900018 (NCT03421379)** — an EliLilly crossover study of intranasal glucagon vs. standard glucagon for insulin-induced hypoglycaemia in Type 2 diabetes patients, sourced from the CDISC DDF-RA reference examples repository.

**POC file:** `USDM_SoA_POC.html` — open in any browser, no server required.

---

## Study

- **Study:** LY900018 — Glucagon Crossover Study (NCT03421379), EliLilly
- **Design:** 2-period crossover — LY-G (LY900018 → Glucagon) and G-LY (Glucagon → LY900018)
- **Indication:** Type 2 diabetes; insulin-induced hypoglycaemia rescue
- **Epochs:** Screening · Period 1 · Wash Out · Period 2 · Follow-Up
- **Visits:** 15 SAIs on Main Timeline (includes intra-day sub-schedule on Day 1)
- **Activities:** 34 protocol activities across 9 groups
- **BiomedicalConcepts:** 35 full BCs (all SDTM Dataset Specialization refs) + 20 BcSurrogates
- **Sub-timelines:** Hypoinduction (5 nodes) + Treatment Phase (15 intra-day nodes, 5 min–240 min)

---

## POC Tabs

### 1. Schedule of Activities
ICH M11-style protocol SoA:
- Four header rows: Epoch band · Treatment by arm (LY-G vs G-LY) · SAI name · Day
- Activities grouped by protocol category
- BC×N badge shows number of BC + surrogate links per activity
- Clinical Lab Tests: 25 BCs + 16 surrogates = 41 total assessments
- Clinical Serology: 4 BcSurrogates (HBsAg, HCV, HIV, Syphilis — no published SDTM DS yet)
- Arm note: crossover — Period 1 arm assignment reverses in Period 2

### 2. Architecture Analysis
Five-layer model:

| Layer | Standard | Role |
|---|---|---|
| 1 | USDM v4.0 | Protocol & Schedule — SAI pivot, StudyCells for crossover arms |
| 2 | COSMoS | Leaf BC catalog (CDISC Library); BcSurrogate for unpublished concepts |
| 3 | COSMoS CRF Spec (draft) | Pre-built form metadata → ODM 2.0 |
| 4 | ODM 2.0 | Data collection — Concept ItemGroups + VLM + compound WhereClause |
| 5 | SDTM Dataset Specialization | VLM → Define-XML v2.1 |

Key callout: **BcSurrogate** — when no SDTM DS exists yet, USDM uses a lightweight placeholder (`reference = "None set"`). Once CDISC publishes a DS, the surrogate is replaced with a full BC.

### 3. Data Tables
18 SQL tables as live Handsontable grids:

| Table | Layer | Key content |
|---|---|---|
| usdm_study | 1 | Study version metadata (LY900018) |
| usdm_epoch | 1 | 5 epochs; linked list |
| usdm_encounter | 1 | 7 encounters; multiple SAIs share P1 DAY 1 |
| usdm_activity | 1 | 34 activities with category and bc_count (includes surrogates) |
| usdm_sai | 1 | 15 SAIs with timing_day; intra-day structure visible |
| usdm_study_cell | 1 | 2 arms × 5 epochs = 10 cells (crossover) |
| usdm_study_element | 1 | 5 epoch-level elements |
| usdm_sai_activity | 1 | Junction table — SoA pivot source |
| **usdm_bc_surrogate** | 1 | **20 BcSurrogates** — lab analytes and serology with `reference = "None set"` |
| cosmos_bc | 2 | 35 unique BCs; all SDTM DS refs; CDISC CT TESTCD codes |
| cosmos_dec | 2 | BC properties (SDTM variable codes) |
| cosmos_coding | 2 | LOINC/SNOMED/CDISC CT per BC |
| **activity_bc** | 1/2 | **56 rows** (36 BC + 20 surrogate); `link_type` column |
| cosmos_crf_item | 3 | CRF Spec draft — AE/CM/DM/DS/EC |
| odm_item_group_def | 4 | Nested Concept IGs; corrected CDISC CT bc_ids |
| odm_item_def | 4 | Topic vars (VSTESTCD) + result vars with ValueListRef |
| odm_value_list | 4 | ValueListDef with compound WhereClauseRef |
| odm_where_clause | 4 | RangeChecks; compound AND conditions |

### 4. Study Model
Summary cards: 15 SAIs, 34 activities, 5 epochs, 2 arms, 35 BCs, 20 surrogates, 2 sub-timelines.

---

## Key Architecture Points

### USDM SAI is the SoA pivot
`ScheduledActivityInstance.name` (SCREENING, P1_DAY_MINUS1…) is the column header. The SoA is a pivot of `usdm_sai_activity`. `SAI.sub_timeline_id` would enable arm-specific branching but is null here — both arms share one schedule.

### Crossover arms via StudyCells
`StudyCells (2 arms × 5 epochs = 10)` define which treatment element each arm receives in each epoch. The schedule is identical across arms; the treatment sequence is what differs.

### BiomedicalConcept vs BcSurrogate
- **BiomedicalConcept** (`biomedicalConceptIds`): full reference to an SDTM Dataset Specialization (`/mdr/specializations/sdtm/…`) or COSMoS BC (`/mdr/bc/packages/…`). BC code = CDISC CT TESTCD value (e.g. C49678 = RESP, not the NCIt physiological concept).
- **BcSurrogate** (`bcSurrogateIds`): `reference = "None set"` — CDISC has not yet published a DS. 20 surrogates cover additional haematology (RBC count, MCV, MCH, MCHC, WBC), LFTs (ALT, AST, ALP, GGT, bilirubin, total protein), BUN, insulin, plasma glucose, and serology (HBsAg, HCV, HIV, Syphilis).
- Both live on `studyVersion`, not `studyDesign`.

### Multiple timelines
Main Timeline (15 SAIs) covers the high-level visit schedule. Two sub-timelines capture detail: **Hypoinduction** (5 nodes for insulin clamp) and **Treatment Phase** (15 intra-day nodes at 5, 10, 15, 20, 25, 30, 40, 50, 60, 90, 120, 240 min post-dose). Sub-timelines would be linked via `SAI.sub_timeline_id`.

### COSMoS coverage gaps
CRF Specialization (2073 rows, 16 domains including VS/EG/LB — see domain table above). Lab BCs (ALT, AST, WBC, BUN etc.) are surrogates in this USDM file.

---

## P21 Implementation Path

1. **Phase 1 — Protocol/SoA:** Import USDM JSON → 7 USDM tables + `usdm_bc_surrogate` → SoA as Handsontable pivot of `usdm_sai_activity`
2. **Phase 2 — BC Catalog:** Sync COSMoS from CDISC Library API; hold surrogates locally; migrate to full BCs as CDISC publishes DSes
3. **Phase 3 — Forms:** Import COSMoS CRF Specializations → ODM 2.0 Concept ItemGroups/ItemDefs
4. **Phase 4 — VLM:** `ValueListDef + WhereClauseDef` for multi-test domains; compound conditions (e.g. VSTESTCD=SYSBP AND VSPOS=SITTING)

---

## Files

| File | Description |
|---|---|
| `USDM_SoA_POC.html` | Interactive POC — open in browser |
| `SUMMARY.md` | This document |
| `TECHNICAL_SUMMARY.md` | Build details and data sourcing |
| `usdm_output_V4.0 (1).json` | TJ301 UC study USDM JSON (original source) |
| `EliLilly_NCT03421379_Diabetes (1).json` | LY900018 diabetes study USDM JSON (current POC source) |
| `USDM-IG_v4.pdf` | USDM Implementation Guide v4.0 |
| `USDM_CT (1).xlsx` | USDM Controlled Terminology |
| `odm 2.0/` | ODM 2.0 XSD schema files |

---

## CRF Specialization → P21 Form Generation

The COSMoS CRF Specialization can be directly transformed into P21's three-level form hierarchy without manual CRF authoring for standard domains.

### Three-level mapping

| CRF Spec field(s) | P21 level | ODM 2.0 type | Notes |
|---|---|---|---|
| `domain` | Form | ItemGroupDef[Type=Form] | One form per SDTM domain (AE, CM, DM…) |
| `crf_group_id` + `short_name` | Section | ItemGroupDef[Type=Section] | e.g. AE_DENORMALIZED, CMFREE_NORMALIZED |
| `crf_item` + `question_text` | Question/Item | ItemDef + ItemRef | Items ending RESU/RES flagged as VLM targets |
| `data_type`, `length`, `significant_digits` | Item attributes | @DataType, @Length, @SignificantDigits | `decimal` with digits → `float` in P21 |
| `codelist` + `value_list` + `value_display_list` | Codelist/Terms | CodeList + EnumeratedItem | Sub-codelists per field; unit sub-codelists for RESU items |
| `sdtm_target_variable` | SDTM target | Alias/annotation | Domain-qualified: AE.AETERM |

### VLM generation

WhereClause is appropriate only for **normalized** implementations (`implementation_option = 'Normalized'`). Denormalized forms have one row per concept — no VLM required.
- `value_list` populated → comparator = `IN`
- `prepopulated_term` set → comparator = `EQ`

WhereClause is built across all fields in a `crf_group_id` with AND logic to form a compound condition.

### Activity → Form link via bc_id

Because `cosmos_crf_item.bc_id` matches the USDM BiomedicalConcept code, the bridge from protocol activity to generated form is fully traceable:

```
USDM Activity
  → activity_bc.bc_code              (via biomedicalConceptIds)
      → cosmos_crf_item.bc_id         (CRF Spec lookup)
          → cosmos_crf_item.domain    → P21 Form
          → cosmos_crf_item.crf_group_id → P21 Section
          → cosmos_crf_item.crf_item  → P21 Question
```

This allows P21 to auto-generate a form skeleton for any activity whose BCs have a published CRF Specialization. In LY900018: Vital Signs (SYSBP, DIABP, PULSE, TEMP), Informed Consent, and HbA1c could generate forms today. Clinical Lab Tests and Serology require VS/LB CRF Specializations (not yet published).

### Form library link (higher-level merge)

Where a P21 MDR already has a form library, the link from activity to form does not need to go through the granular BC→CRF Specialization chain. A simpler direct join can be maintained at the **activity level**:

```sql
CREATE TABLE activity_form (
    activity_id   VARCHAR,   -- USDM Activity ID
    form_id       VARCHAR,   -- P21 Form ID (e.g. AE, VS, LB)
    form_name     VARCHAR,   -- Human-readable form name
    link_source   VARCHAR    -- 'crf_spec' | 'manual' | 'library'
);
```

This allows:
- **Auto-populate** from the bc_id→CRF Spec chain where coverage exists
- **Manual override** for activities where BCs are missing or surrogates are used
- **Library reuse** where a sponsor has pre-authored forms not derived from CRF Spec

The two approaches are complementary:
- BC-level merge: granular, traceable, driven by USDM data — shows *which specific BCs* a form collects
- Form-level merge: pragmatic, works even without BC links — shows *which forms* an activity uses

For P21's Handsontable view, the `activity_form` table could be an editable junction — auto-populated where possible, manually maintained elsewhere.

### COSMoS CRF Specialization vs. CDISC eCRF Portal

There are two distinct CDISC sources for form metadata — it is important not to conflate them:

| Source | Format | Domains | Access | Use for MDR |
|---|---|---|---|---|
| **COSMoS CRF Specialization** (GitHub draft CSV) | Machine-readable 35-column schema | AE, CM, DM, DS, EC | Public (draft) | Direct programmatic import via SAS or API |
| **CDISC eCRF Portal** (Knowledge Base) | CDASH-based HTML/ODM 1.3 templates | VS, EG (3 variants), LB, and many more | Member-only (Gold/Platinum) | Manual download; format conversion needed |

VS (Vital Signs) and EG (ECG — Local Reading, Central Reading, Central Reading with Investigator Assessment) are available on the [CDISC eCRF Portal](https://www.cdisc.org/kb/ecrf/vital-signs) as CDASH-based templates, but have **not yet been published in the machine-readable COSMoS CRF Specialization format**. LB (Laboratory) is absent from both the draft CSV and the eCRF Portal COSMoS format. The COSMoS CRF Specialization format is expected to expand to additional domains as the standard matures.


### Coverage today vs. future

| Activity | BC/Surrogate links | CRF Spec available | Auto-generate form? |
|---|---|---|---|
| Informed Consent | BC: CONSENT | ✓ DS domain | Yes |
| Vital Signs | 4 BCs (VS domain) | ✓ VS — SBP, DBP, Pulse, Temp all in CRF Spec | Yes — SYSBP_DENORMALIZED, DIABP_DENORMALIZED etc. |
| Clinical Lab Tests | 25 BCs + 16 surrogates | ✓ LB — 77 BCs, 976 rows in CRF Spec | Yes — most lab analytes covered |
| Adverse Events | BC: AE (C41331) | ✓ AE domain | Yes |
| Concomitant Meds | BC: CMFREE | ✓ CM domain | Yes |
| HbA1c | BC: HBA1CBLD (C64849) | ✓ LB — HBA1CBLD_DENORMALIZED in CRF Spec | Yes |

Note: BcSurrogates (`reference = "None set"`) cannot drive form generation until they are promoted to full BCs with a published SDTM DS reference.

---

## SDTM Trial Summary Domains from USDM SoA

The USDM SoA provides all the data required to generate the five SDTM Trial Design domains directly from the JSON — no manual SDTM authoring needed.

**Output file:** `LY900018_SDTM_Trial_Domains.xlsx` (5 sheets)

| Domain | Source in USDM | LY900018 rows | Notes |
|---|---|---|---|
| TA — Trial Arms | StudyCells → Arm × Epoch → StudyElement sequence | 10 | TAETORD = element order within arm; crossover sequence explicit per arm |
| TE — Trial Elements | studyDesign.elements; transitionStartRule/transitionEndRule | 5 | TESTRL/TEENRL from USDM TransitionRule.text |
| TV — Trial Visits | ScheduledActivityInstance sequence + timing objects | 30 (15 visits × 2 arms) | VISITNUM from SAI position; VISITDY from ScheduleTimeline.timings |
| TI — Inclusion/Exclusion | eligibilityCriteria + eligibilityCriterionItems (HTML stripped) | 36 | IECAT from category.decode; sorted by identifier number |
| TS — Trial Summary | studyPhase, objectives, population, interventions, identifiers | 23 | See TSSEQ note |

### TSSEQ (SDTM IG 3.4 §5.5)

TSSEQ is a sequence number **within each TSPARMCD**, not a global running counter. This is different from --SEQ in subject-level domains.

- If TSPARMCD = `OBJSEC` has 3 secondary objectives → TSSEQ = 1, 2, 3
- If TSPARMCD = `TRT` has 2 treatment arms → TSSEQ = 1, 2
- Single-value parameters always have TSSEQ = 1

**TSGRPID** ties related parameters together across rows — for example, TRT and DOSE records for the same treatment arm would share a TSGRPID value. In LY900018: TRT TSSEQ 1 (LY900018) and TRT TSSEQ 2 (GlucaGen) each have their own TSGRPID.

### USDM → SDTM field mapping highlights

- `StudyArm.name` → TA.ARMCD; `StudyArm.description` → TA.ARM
- `StudyElement.name` → TA.ETCD and TE.ETCD; `transitionStartRule.text` → TE.TESTRL
- `SAI.name` (V1, V1.1…) used to derive TV.VISITNUM sequence; ScheduleTimeline.timings → TV.VISITDY
- `EligibilityCriterion.name` (INC1…EXC15) → TI.IETESTCD; `EligibilityCriterionItem.text` (HTML stripped) → TI.IETEST
- `StudyVersion.titles.text` → TS TITLE; `StudyIdentifier.text` → TS STUDYID/REGID; objectives → TS OBJPRIM/OBJSEC
