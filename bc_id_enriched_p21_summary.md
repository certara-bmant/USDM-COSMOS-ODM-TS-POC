# Using BC IDs and Enriched P21 Master Standards to Generate Study-Level Outputs

## Overview

The P21 enriched metadata model anchors every piece of study metadata — CRF questions, SDTM variables, value-level rows, codelists — to a **Biomedical Concept (BC)**. A BC is identified by a `bc_id` (an NCI C-code, e.g. `C49677` for Systolic Blood Pressure) and a `vlm_group_id` (a human-readable label matching the BC in context, e.g. `SYSBP`).

When a study team selects a set of BCs as the observations for their study, those selections become the filter key used to **cut** enriched master standards down to study-specific outputs. Importantly, BC matching is performed **per activity** — the USDM structures BCs within Activities (e.g. Chemistry, Urinalysis, Vital Signs), and the activity context can inform and narrow the `vlm_group_id` selection where a `bc_id` maps to multiple options. For example, a glucose measurement within a Chemistry activity may resolve to a different `vlm_group_id` than the same BC appearing within a different activity context.

1. **Study CRF Form Spec** — cut from the **Master Form Spec** by filtering Sections on `vlm_group_id`, then further scoped by `crf_group_id` to select the appropriate layout group
2. **Study SDTM Define** — cut from the **Master SDTM Standard** using `vlm_group_id` on Value Level Metadata (VLM), cascading to variables, codelists, and terminology

Both operations follow the same pattern: the master standard is the source of truth; the BC selection is the filter.

---

## The Master Standards

Three enriched master standards underpin the generation process. Each is authored once and maintained centrally; studies draw from them rather than authoring content from scratch.

**Master SDTM Standard**
The SDTM Implementation Guide enriched with COSMoS VLM. Every Value Level Metadata row carries a `bc_id` and `vlm_group_id`, linking each observation type (e.g. a systolic BP result row in VS) to its biomedical concept. The standard covers all datasets, all variables, and the full value-level detail for every supported BC.

**Master Terminology**
The controlled terminology enriched with BC-specific subsets. In addition to the full standard codelists (e.g. VSRESU, VSTESTCD), the master terminology includes pre-built subsets scoped to individual BCs or BC groups (e.g. a unit subset containing only the valid units for blood glucose). These subsets are resolved from CDISC dataset specializations and stored in the master so that study generation does not need to derive them at runtime.

**Master Form Spec**
The CRF standard enriched with `bc_id`, `vlm_group_id`, and `crf_group_id` on each Section. The form spec is organised hierarchically (Events → Forms → Sections → Questions → Codelists / Units) with each Section mapped to a specific BC. The `crf_group_id` groups related Sections within a Form — for example, all vital sign measures that share a common CRF layout (e.g. a repeated panel with one row per measurement) would share a `crf_group_id`, distinguishing them from Sections that appear as standalone blocks. All question-level metadata — SDTM targets, pre-populated values, response codelists, units, and DEC IDs — is authored in the master and inherited by any study that selects the corresponding BC.

---

## 1. Study CRF Form Spec — Filtering by VLM Group ID on Sections

### Selection Sequence for CRF Form Spec

Generating the study form spec involves two sequential user decisions:

**Step A — Select vlm_group_id (BC selection, per activity)**
The study team selects the BCs they need. Each `vlm_group_id` identifies the clinical concept (e.g. SYSBP, GLUCBLD). This selection is driven by the USDM study input and is resolved **per Activity** — the Activity context (e.g. Chemistry, Urinalysis, Vital Signs) can narrow or disambiguate the `vlm_group_id` where a `bc_id` has multiple possible mappings. A single `bc_id` may map to more than one `vlm_group_id` (e.g. a glucose BC appearing in both an LB and an FA specialization); the user selects which apply.

**Step B — Select crf_group_id (where required)**
This step follows the same pattern as Step A, one level down. Just as a single `bc_id` can map to multiple `vlm_group_id`s, a single `vlm_group_id` may appear under more than one `crf_group_id` in the master form spec. The `crf_group_id` identifies the layout group — the panel or table within a Form that the Section belongs to. For example, SYSBP might appear in a supine vital signs group and a standing vital signs group, each with a different `crf_group_id`. Where a `vlm_group_id` maps to only one `crf_group_id`, this step is automatic; user input is only required when there is more than one match.

The `crf_group_id` also governs how Sections are assembled in the output. Sections sharing a `crf_group_id` are rendered as a single panel or repeated block in the CRF tool — for instance, all vital sign measures collected in a single row-per-measurement table share one `crf_group_id`. If a study selects only a subset of BCs from a group (e.g. SYSBP and DIABP but not PULSE), those Sections are included with their shared group identifier preserved so the panel layout remains valid.

### Cut Operation

Given selected `vlm_group_id`s and the resolved `crf_group_id` for each:

1. **Filter Sections** — retain only Sections whose `vlm_group_id` is in the selection
2. **Resolve `crf_group_id`** — where a `vlm_group_id` maps to multiple groups, use the user-selected `crf_group_id` to disambiguate *(user input required in these cases)*
3. **Cascade to Forms** — retain only Forms that contain matched Sections
4. **Cascade to Questions** — retain only Questions belonging to matched Sections
5. **Cascade to Codelists and Terms** — retain only Codelists referenced by matched Questions
6. **Cascade to Units** — retain only Units referenced in matched Question measurement unit fields

```
Master Form Spec (all BCs, all crf_group_ids)
        │
        │  Step A: filter vlm_group_id ∈ selected (per activity)
        ▼
Matched Sections
        │
        │  Step B: resolve crf_group_id per BC
        │  (user input where multiple groups match)
        ▼
Study CRF Form Spec
  ├── Forms      (containing matched Sections)
  ├── Sections   (one per selected BC × crf_group_id)
  ├── Questions  (with SDTM targets, pre-pop values, DEC IDs)
  ├── Codelists  (only those referenced by matched Questions)
  ├── Terms      (only terms within matched Codelists)
  └── Units      (only units appearing in matched Questions)
```

### Additional Thoughts — Activity-Driven Form Selection

The master form spec may contain multiple Forms that each address an overlapping set of BCs — for example, a combined Height and Weight form (used at screening) and a standalone Weight-only form (used at each subsequent visit). Both Forms contain a Weight Section, but they serve different activities in the study schedule.

Where the USDM structures BCs within Activities (e.g. Anthropometry, Chemistry), the Activity context is the most principled basis for selecting the best-matched Form. Rather than requiring the user to manually choose among candidate Forms, the system can compare the set of BCs associated with a USDM Activity against the set of BCs covered by each candidate Form, and select the Form whose BC coverage most closely matches the Activity — preferring specificity (a Weight-only form for a weight-only Activity) over breadth.

This means the Form selection is ideally resolved automatically from the USDM rather than as a separate manual step, with user intervention reserved only for cases where no clear best match exists.

---

## 2. Study SDTM Define — Filtering by VLM Group ID on Value Level Metadata

### Cut Operation

**Step 1 — Filter VLM rows and variable-level specializations**

Two distinct patterns exist depending on whether the BC has VLM targets in the master standard:

*Pattern A — BC with VLM targets (e.g. VS, LB, EG)*
Retain VLM rows whose `vlm_group_id` is in the selected set. Each row represents one observation type scoped by a where clause (e.g. `VSTESTCD EQ SYSBP`). The where clause is the mechanism that ties a variable's value-level row to a specific BC.

*Pattern B — BC with no VLM targets (e.g. AE, CM, EX)*
Some domains do not use value-level scoping — every collected record is of the same type, so there is no where clause and no VLM row. In these cases the BC maps to variables rather than to value-level rows.

A key consideration here is that many SDTM variables in these domains — while they carry an NCI code (e.g. AEHLT, C117380) — do not correspond to a conceptual data element in the COSMoS BC model. Variables like AEHLT, AEHLTCD, AEHLGT exist purely as artefacts of the SDTM implementation (in this case, encoding the MedDRA hierarchy) and have no independent DEC representation in the USDM. A `dec_id` lookup therefore cannot reliably identify these variables, as there is no concept-level anchor for them.

The CDISC dataset specializations are consequently the appropriate route for Pattern B: they capture the full SDTM implementation mapping for a `vlm_group_id`, including implementation-level variables that have no conceptual DEC counterpart. The specializations are used to identify which variables are associated with a given `vlm_group_id`, with `dec_id` reserved only for variables that genuinely correspond to a collected conceptual data element.

The result in all Pattern B cases is a flat variable list for the dataset with no value-level rows — each variable appears once in the Variables sheet with any assigned values or codelists applied uniformly.

**Multiple specializations per BC**

A single BC (`bc_id`) can have more than one `vlm_group_id` in the master standard — for example, a glucose measurement may have specializations for both the Findings (LB) and the Findings About (FA) domains, each with a distinct `vlm_group_id`. Selecting the appropriate `vlm_group_id` is itself the specialization selection: the user chooses which `vlm_group_id`(s) to include for a given BC, and the cut proceeds from there. No separate specialization-resolution step is needed beyond the BC/VLM group selection.

**Step 2 — Identify required Datasets and Variables**

- Datasets: the unique set of datasets appearing in the filtered VLM rows and variable-level specializations
- Variables: pulled from the master standard for each included dataset using three sources:
  - **Required and Expected variables** — all variables classified as Required or Expected in the master standard are automatically included for every dataset in scope, regardless of whether they appear in VLM rows
  - **VLM-referenced variables** — variables that appear in filtered VLM rows or variable-level specializations
  - **Where clause variables** — variables promoted because they appear in Where Clauses (e.g. `VSTESTCD` in `VSTESTCD EQ SYSBP`), required in the define to support the value-level scoping even if they have no VLM row of their own

Variable metadata (label, data type, length, mandatory, codelist, origin, role) is pulled from the master standard for each included variable.

**Step 3 — Subset Codelists and Terminology**

The BC selection determines which terms appear in the study terminology. Three rules apply, in order:

| Rule | Trigger | Result |
|---|---|---|
| **Assigned value** | VLM row has a fixed `assignedValue` (e.g. units fixed to mmHg) | A codelist containing that single term |
| **Where clause terms** | Variable appears in Where Clauses (e.g. `VSTESTCD EQ SYSBP`) | Codelist includes only the specific terms referenced across all selected BCs |
| **Full codelist** | Variable references a codelist with no assignment or where clause constraint | All terms from the master terminology codelist |

Where a BC-specific subset codelist exists in the master terminology (pre-built from specializations), that subset is used directly rather than filtering the parent codelist at generation time.

Note: the master terminology exists as a separate object in P21. Only terms that are explicitly assigned, constrained by where clauses, or part of assigned codelists are attached to the study define — the full terminology is not included.

```
Master SDTM Standard (all datasets, all VLM rows)
        │
        │  filter: vlm_group_id ∈ selected
        ▼
Filtered VLM rows
        │
        │  cascade: unique datasets + variables (+ promoted where-clause vars)
        ▼
Study Variables + Datasets  ←  metadata from master standard
        │
        │  resolve: assigned / where-clause / full codelist rules
        ▼
Study Terminology  ←  terms + NCI codes from master terminology
```

### Alternate Data Flows by BC Type

```
USDM BC Selection  (bc_id → vlm_group_id)
              │
              │ Does vlm_group_id have VLM targets?
              │
       ┌──────┴──────┐
      YES             NO
       │               │
       ▼               ▼
  Pattern A       Pattern B: dataset specializations
  (VS, LB, EG, …) Use specializations to map
                   vlm_group_id → variable list
  Filter VLM rows  (preferred route — covers both
  by vlm_group_id  conceptual and implementation-only
       │           variables such as AEHLT, AEHLTCD)
       │                    │
       ▼                    ▼
  Where-clause        (no VLM rows)
  scoping per BC            │
       └──────────┬─────────┘
       │                               │
       ▼                               ▼
  Value Level rows          Flat variable list
  in study define           in study define
       │                               │
       └───────────────┬───────────────┘
                       │
                       ▼
          Required + Expected variables
          added from master standard
                       │
                       ▼
          Codelist + Terminology cut
          (assigned / where-clause / full)
                       │
                       ▼
               Study SDTM Define
```

---

## 3. The Shared Pattern

Both outputs follow the same conceptual model:

```
USDM Study Input
  └── BC Selection  (bc_id)
              │
              │  resolve: bc_id → vlm_group_id
              │
     ┌────────┴────────┐
     │                 │
     ▼                 ▼
Master Form Spec    Master SDTM Standard
     │              (enriched with COSMoS VLM)
     │  filter by        │
     │  vlm_group_id     │  filter by vlm_group_id
     ▼                   ▼
Matched Sections    Filtered VLM rows
     │
     │  filter by crf_group_id
     │  [user input where multiple groups match]
     ▼
Study CRF          Study SDTM Define
Form Spec          (datasets, variables, VLM,
                    terminology — from master)
```

The `bc_id` is the USDM input. The `vlm_group_id` is resolved downstream from it and acts as the shared filter key against both master standards — linking the selected BC to its CRF representation (Sections in the master form spec) and its SDTM representation (VLM groups in the master SDTM standard).

A single BC selection drives both outputs consistently. The master standards encode all domain expertise; the study is a selection over them, with no bespoke authoring required.

---

## 4. Summary of Roles

| Master Standard | Enrichment | Used to Generate |
|---|---|---|
| Master SDTM Standard | COSMoS VLM with `bc_id` / `vlm_group_id` on every VLM row | Study datasets, variables, value-level metadata |
| Master Terminology | Full standard codelists + pre-built BC-specific subsets | Study codelists and terms (assigned, constrained, or full) |
| Master Form Spec | `bc_id` / `vlm_group_id` / `crf_group_id` on every Section | Study CRF forms, sections (grouped by `crf_group_id`), questions, units, response codelists |
