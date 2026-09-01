# USDM SoA POC v2 — Addendum

> Addendum to the [USDM SoA POC](USDM_SoA_POC.html). Documents the two additions present in [`USDM_SoA_POC_v2.html`](USDM_SoA_POC_v2.html): Activity → CRF Form matching and SDTM Trial Design Domain tables.

---

## New Tables in v2

| Table | Layer | Rows | Source |
|---|---|---|---|
| `activity_form_match` | Form Library Match | 44 | Certara Proliferated CRF Form Library + COSMoS BCs |
| `sdtm_ta` | SDTM Trial Domains | 10 | LY900018_SDTM_Trial_Domains.xlsx |
| `sdtm_te` | SDTM Trial Domains | 5 | LY900018_SDTM_Trial_Domains.xlsx |
| `sdtm_tv` | SDTM Trial Domains | 30 | LY900018_SDTM_Trial_Domains.xlsx |
| `sdtm_ti` | SDTM Trial Domains | 36 | LY900018_SDTM_Trial_Domains.xlsx |
| `sdtm_ts` | SDTM Trial Domains | 23 | LY900018_SDTM_Trial_Domains.xlsx |

---

## 1. activity_form_match

### Purpose

Matches each USDM Activity (from `cosmos_activity`) to one or more candidate CRF forms from the **Certara Proliferated CRF Form Library**, ranking candidates and identifying the recommended match per activity.

### Columns

| Column | Description |
|---|---|
| `activity_id` | Foreign key to `cosmos_activity` |
| `activity_name` | Activity display name |
| `activity_category` | Category grouping (e.g. Baseline Assessments, Laboratory) |
| `form_id` | Matched form identifier from the form library |
| `form_name` | Matched form display name |
| `match_score` | Score string: `N/M` BC overlap for BC matches; `N% name` for word-overlap matches; blank for name-only |
| `match_method` | Match method used — see below |
| `recommended` | `Y` = top-ranked match for this activity; `N` = secondary candidate |
| `missing_bcs` | For BC-matched rows: pipe-separated list of BC codes from the activity not covered by the matched form (e.g. `C49677|C49678`) |
| `surr_coverage` | For surrogate-matched rows: `N/M surr.` — number of surrogate BCs matched via section/question corpus |

### Match Methods

| Method | Colour | Logic |
|---|---|---|
| `BC` | Green | CDISC C-code overlap between activity BCs and form section BCs. Score = matched / total activity BCs. |
| `Actv. Name` | Amber | Activity name keywords matched against form names (word token lookup). |
| `Form Name` | Blue | Word-overlap ≥ 40% between activity name and form name. |
| `Surr. Name` | Purple | Surrogate BC section name / question text / prompt corpus match. Used where no BC or name match is available. Secondary rows added where surrogates identify a better-fitting form than the BC match. |
| `None` | Grey | No match found. |

### Surrogate Matching Detail

Where an activity has BC surrogates (`bcSurrogateIds` in the USDM) rather than direct `biomedicalConceptIds`, a token-based match is performed against the form library corpus (section names + question text + prompts). The `surr_coverage` column reports how many of the activity's surrogate BCs are covered by a given form.

Example — Activity_20 "Clinical Lab Tests":
- Primary BC match: `LB-VERT` (24/25 BCs covered) — but LB-VERT is a generic 12-question vertical form with no analyte-specific sections, giving 0/16 surrogate coverage.
- Secondary surrogate rows added: `LB-CHEM` (10/16 surr.) and `LB-HEM` (6/16 surr.) — more specific forms identified via the surrogate corpus match.

---

## 2. SDTM Trial Design Domains

Five SDTM trial design domain tables are included as read-only reference data, sourced directly from `LY900018_SDTM_Trial_Domains.xlsx` (generated from the LY900018 USDM JSON using the CDISC DDF-RA SDTM mapping).

### Domains

| Domain | Label | Rows | Key USDM source |
|---|---|---|---|
| `sdtm_ta` | Trial Arms | 10 | `StudyDesign.arms`, `epochs`, `studyCells` |
| `sdtm_te` | Trial Elements | 5 | `StudyDesign.elements` + transition rules |
| `sdtm_tv` | Trial Visits | 30 | `ScheduleTimeline.instances` (SAI) × arm |
| `sdtm_ti` | Trial Inclusion/Exclusion | 36 | `StudyDesign.eligibilityCriteria` + `studyVersion.eligibilityCriterionItems` |
| `sdtm_ts` | Trial Summary | 23 | `studyVersion` + `StudyDesign` metadata fields |

### Scope Notes

- **TM (Trial Disease Milestones)** and **TD (Trial Disease Assessments)** are not included. Both are absent from the CDISC DDF-RA SDTM mapping (`sdtm_mapping.xlsx`) and are inapplicable to the non-oncology LY900018 study type.
- TV generation uses the Main Timeline only. Sub-timelines (e.g. PK sampling) are not represented.
- TS parameter coverage reflects what is present in the LY900018 USDM file; parameters with no USDM source are absent rather than blank-filled.

---

## UI Changes in v2

- **`activity_form_match` button** added to the table selector panel (green, starred: `activity_form_match ★`)
- **Five SDTM domain buttons** added (red styling): `SDTM.TA`, `SDTM.TE`, `SDTM.TV`, `SDTM.TI`, `SDTM.TS`
- **Row colouring** in `activity_form_match` reflects match method (green = BC, amber = Actv. Name, blue = Form Name, purple = Surr. Name, grey = None)
