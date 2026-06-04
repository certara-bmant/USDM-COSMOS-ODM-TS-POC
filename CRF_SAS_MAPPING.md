# CRF Specialization → P21 Form: SAS Mapping Logic

**Script:** `crfdss_to_form.sas`  
**Input:** `cdisc_crf_specializations_draft.xlsx` (COSMoS GitHub — `export/` folder)

> **CRF Specialization coverage (2026-05-14):** 16 domains, 2073 rows, 303 unique specializations. VS (112 rows, 13 BCs), EG (243 rows, 24 BCs), LB (976 rows, 77 BCs), plus AE, CM, DM, DS, EC, FT, IE, MH, PR, QS, SC, SU, MB. The CSV and XLSX in the GitHub `export/` folder have identical content and are kept in sync with the CDISC Library. For large-scale or production use, the CDISC Library API supports filtering by domain or bc_id directly.  
**Outputs:** `crf_form.xlsx` (Forms/Sections/Questions/Units/Codelists/Terms/FormSpec) and `crf_vlm.xlsx` (P21 VLM template)

---

## Overview

Transforms the flat 35-column COSMoS CRF Specialization into P21's three-level form hierarchy and a VLM (Value Level Metadata) sheet ready for import. The output matches P21's EDC form specification format.

---

## Step 1 — Data Cleaning (`dss_` dataset)

Before mapping, the CRF Spec is cleaned for P21 compatibility:

```sas
/* Items ending RESU/RES are VLM targets (result variables) */
if substr(crf_item, length(trim(crf_item))-3, 4) = 'RESU' or
   substr(crf_item, length(trim(crf_item))-2, 3) = 'RES'
   then vlm_target = 'Y';

/* decimal with significant_digits but length ≠ 200 → float */
if significant_digits ne '' and length ne '200' then data_type = 'float';

/* length=200 implies free-text — significant_digits not applicable */
if length = '200' then significant_digits = '';
```

**Checks produced (`chks` dataset):**
- `significant_digits` set on a non-float item where length ≠ 200 → should be float
- `significant_digits` set where length = 200 → digits not required for free text

---

## Step 2 — Codelist Sub-setting (`cls` dataset)

Three sub-codelist naming patterns are generated from rows where `value_list` or `prepopulated_term` is populated:

| Pattern | Condition | ID format | Name format |
|---|---|---|---|
| Unit codelists | Item ends `RESU` | `{codelist_submission_value}_{vlm_group_id}_OR` | `Unit, subset for {short_name} - Original` |
| Value codelists (with CT) | Has `codelist_submission_value`, not RESU | `{codelist_submission_value}_{crf_group_id}_{crf_item}` | `Subset for {short_name} - {crf_item} {codelist_submission_value}` |
| Value codelists (no CT) | No `codelist_submission_value`, not RESU | `{crf_item}_{crf_group_id}` | `Subset for {short_name} - {crf_item}` |

`value_list` and `value_display_list` are split on `;` to generate individual term rows.

---

## Step 3 — WhereClause Generation (`wc` / `wc_` datasets)

WhereClause is generated only for **normalized** implementation options on non-result items:

```sas
/* Scope: normalized sources, non-VLM-target items only */
if implementation_option = 'Normalized' and vlm_target = '';

/* Comparator selection */
if value_list ne ''      then comparator = 'IN';
if prepopulated_term ne '' then comparator = 'EQ';

/* WhereClause text */
if comparator = 'EQ' then
    wc = crf_item || ' ' || comparator || ' ' || assigned_value;
if comparator = 'IN' then
    wc = crf_item || ' ' || comparator ||
         ' ("' || tranwrd(value_list, ';', '","') || '")';
```

Conditions within a `crf_group_id` are combined with AND:
```sas
combinedwc = catx(' and ', combinedwc, wc);
```

The combined where clause is then merged back to the VLM target items.

---

## Step 4 — P21 Output Sheets

### Forms sheet (`form_final`)

Source: distinct `domain` values.

| Column | Source | Notes |
|---|---|---|
| ID | `domain` | e.g. AE, CM, DM, DS, EC |
| name | `domain` | Same as ID |
| sdtm_target_domain | `domain` | SDTM domain for the form |

Maps to: **ODM 2.0** `ItemGroupDef[Type=Form]`

---

### Sections sheet (`sections_final`)

Source: distinct `crf_group_id` per form.

| Column | Source | Notes |
|---|---|---|
| ID | `crf_group_id` | e.g. AE_DENORMALIZED, CMFREE_NORMALIZED |
| name | `short_name` | Human-readable label |
| form | `domain` | Parent form ID |
| mandatory | Hardcoded | `No` |
| repeating | Hardcoded | `No` |
| order | Auto-incremented per form | Reset to 1 for each new form |

Maps to: **ODM 2.0** `ItemGroupDef[Type=Section]`

---

### Questions sheet (`questions_final`)

Source: one row per `crf_item` within each section.

| Column | Source | Notes |
|---|---|---|
| Form | `domain` | |
| Section | `crf_group_id` | |
| short_name | `short_name` | BC concept label |
| Order | Auto-incremented per section | |
| ID | `crf_item` | e.g. AETERM, AESEV |
| Question_Text | `question_text` | Full question wording |
| Prompt | `prompt` | Grid/table column label |
| Data_Type | `data_type` (cleaned) | `decimal`→`float` applied |
| Length | `length` | |
| Digits | `significant_digits` (cleaned) | Blank for free-text |
| Mandatory | `mandatory_variable` | `Y`→`Yes`, `N`→`No` |
| Codelist | `codelist` (NCI C-code or sub-codelist ID) | From cls join |
| Measurement_Units | `value_list` (for RESU items, `,`-separated) | |
| SDTM_Target | `sdtm_target_variable` (domain-qualified) | See qualification logic below |
| sdtm_annotation | `sdtm_annotation` | Raw annotation string |
| prepopulated_term | `prepopulated_term` | |

**SDTM target qualification:**
```sas
/* Single target */
sdtm_target = trim(left(form)) || '.' || sdtm_target;

/* Multi-target (comma-separated) */
sdtm_target = tranwrd(sdtm_target, ',', ',' || trim(left(form)) || '.');
sdtm_target = trim(left(form)) || '.' || sdtm_target;
/* Result: AE.AETERM,AE.AEMODIFY for multi-target items */
```

Maps to: **ODM 2.0** `ItemDef` + `ItemRef`

---

### Units sheet (`units`)

Source: `cls` rows where name contains `"Unit,"`.

| Column | Source |
|---|---|
| id | `term` (the unit value itself) |
| unit | `decoded_value` (or `term` if no decode) |

---

### Codelists sheet (`codelist_final`)

Source: distinct sub-codelists from `cls`.

| Column | Source | Notes |
|---|---|---|
| id | `id` from cls | Generated sub-codelist ID |
| name | `name` from cls | Human-readable label |
| type | Hardcoded | `text` |

---

### Terms sheet (`terms_final`)

Source: `cls` terms, deduplicated by `codelist` + `recommended_term`.

| Column | Source |
|---|---|
| codelist | `id` from cls |
| recommended_term | `term` |
| display_term | `decoded_value` (or `term` if blank) |
| order | Row order |

---

## Step 5 — VLM Output (`crf_vlm.xlsx`)

Merges VLM targets with their combined WhereClause and sub-codelist IDs. Output columns match the P21 VLM template:

| P21 VLM column | Source | Notes |
|---|---|---|
| Order | Auto-incremented | |
| Dataset | `domain` | |
| Variable | `sdtm_variable` | |
| Variant | — | Blank |
| Where_Clause | `combinedwc` | Combined AND conditions from Step 3 |
| Label | `short_name` | |
| Data_Type | `data_type` (cleaned) | |
| Length | `length` | |
| Significant_Digits | `significant_digits` (cleaned) | |
| Mandatory | `mandatory_value` | |
| Codelist | Sub-codelist ID (from cls join) or `codelist_submission_value` | |
| Origin | Hardcoded | `Collected` |

---

## Key Design Decisions

- **Normalized only for VLM**: WhereClause generation is restricted to `implementation_option = 'Normalized'` rows. Denormalized forms have one concrete row per concept — no conditional logic needed.
- **RESU/RES detection**: Items ending in these suffixes are result variables — they become VLM targets, not condition sources.
- **Unit extraction**: Items ending `RESU` produce a measurement unit sub-codelist from `value_list`, enabling unit validation in the EDC.
- **Sub-codelist deduplication**: Duplicate term entries (same term + value_list) are removed. Near-duplicates with differing scope get a `newid` based on variable + value to avoid collisions.
