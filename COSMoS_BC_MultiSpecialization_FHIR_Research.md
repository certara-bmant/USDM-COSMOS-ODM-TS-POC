# COSMoS BC Multi-Specialization & FHIR Architecture — Research Report

**Date:** July 2026  
**Scope:** How CDISC COSMoS handles BC → multiple specializations (CRF, SDTM, ADaM, FHIR); FHIR bridge status; whether "everything via SDTM" is official; USDM model constraints.

---

## Executive Summary

The COSMoS framework is **architecturally designed** for a single BC to serve as a standards-agnostic conceptual anchor that multiple downstream specializations (SDTM, CRF/CDASH, FHIR, OMOP, ADaM) can independently link to. However, today **only SDTM Dataset Specializations are published**, with CRF specializations in draft. FHIR, ADaM, and OMOP specializations are on the roadmap but do not yet exist as formal COSMoS artifacts.

The FHIR bridge currently lives in a **separate HL7/CDISC Joint Mapping IG** (v1.0, 2024), which routes FHIR → SDTM/CDASH — not directly to analytical datasets. Your skepticism about "everything via SDTM" is well-founded: the current practical reality is SDTM-centric, driven by FDA regulatory submission requirements rather than a principled architectural choice.

The **USDM v4 model** (`BiomedicalConcept.reference: String [0..1]`) cannot natively hold multiple specialization references — this is a real model gap that hasn't been addressed in v4.

---

## 1. COSMoS Architecture Intent: Multiple Specializations Per BC

### The Conceptual Design

The COSMoS framework explicitly separates:
- **Conceptual layer** — the BC itself, standards-agnostic, anchored in NCI Thesaurus (NCIt) codes
- **Implementation layer** — Dataset Specializations, each binding the BC to a specific data standard

Per the CDISC BC documentation: *"The data model is flexible and can accommodate other standards (e.g., HL7 FHIR, etc.)"*

The architecture vision is that **many different data standards can refer to the same Biomedical Concepts** — making the framework scalable to bring data together from disparate sources. The planned specialization types explicitly identified include:

| Specialization Type | Status (July 2026) |
|---|---|
| SDTM Dataset Specialization | **Published** (~1,400+ specializations) |
| CRF Specialization (CDASH-aligned) | **Draft** — 303 files, 190 BCs |
| FHIR Dataset Specialization | **Planned — not yet implemented** |
| ADaM Dataset Specialization | **Planned — not yet implemented** |
| OMOP Dataset Specialization | **Planned/community discussion** |

Sources confirming planned FHIR and CDASH/CRF specializations:
- 2023 CDISC Interchange BC presentation ("Looking to Future" slide): *"CDASH dataset specializations — FHIR dataset specializations"* listed as upcoming activities
- CDISC 360i 2025 progress report: 360i Build team generated "eCRF specifications, ODM.xml, SDTM annotated CRFs, SDTM dataset specializations" — noting gaps remain for FHIR/ADaM

### How Multiple Specializations Would Work Per BC

The architecture is: one BC → N specializations, each a separate artifact with its own schema, linked by `bc_id` / `biomedicalConceptId`. There is no constraint at the COSMoS data model level preventing a BC from having simultaneous SDTM + CRF + FHIR + ADaM specializations. Each specialization maps the BC's Data Element Concepts (DECs) to the variables/elements of that standard.

For example, for **HbA1c (C111207)**:
- SDTM specialization: `vlm_group_id=HBA1CHGBBLD` → LB domain, mandatory: LBORRES, LBORRESU, LBSPEC
- CRF specialization: `vlm_group_id=HBA1CHGBBLD` → CDASH items, question text, mandatory fields
- Future FHIR specialization: would map to `Observation.code (LOINC 4548-4)` + `Observation.valueQuantity`

The `vlm_group_id` already serves as the cross-link between CRF and SDTM specializations. A future FHIR specialization would extend this pattern.

---

## 2. FHIR Support: What Exists vs. What's Planned

### Existing: FHIR-CDISC Joint Mapping IG v1.0

Published jointly by HL7 International BR&R Work Group and CDISC (2024). This is a **separate standard, not part of COSMoS**. Key characteristics:

- **Direction:** FHIR R4 → CDISC (SDTM/CDASH)
- **Scope (v1.0):** Adverse Events (AE), Concomitant Medications (CM), Demographics (DM), Lab (LB), Medical History (MH), Procedures (PR), Vital Signs (VS)
- **Scope (v2.0 planned):** SU, SC, HO, MB, DI, EC/EX, MI, TU, TR, RP, RS domains
- **Nature:** Informative (not normative). Deviations "may be appropriate or even common" — study-specific and regulatory-context adjustments needed
- **Format:** Tabular mapping tables organized by CDISC domain (CDISC first, FHIR second)

### Critical Limitation: Conceptual Mismatch

The FHIR-CDISC IG caveats page is explicit about key gaps:

1. **Terminology clash**: EHRs use SNOMED/ICD/LOINC; CDISC uses MedDRA/NCI-CT. Mapping between them is not straightforward.
2. **Concept gaps**: Variables like `--PRESP`/`--OCCUR` (pre-specification/occurrence) have no FHIR equivalent — they only exist when data is collected on a sponsor's CRF, not passively in an EHR.
3. **RELREC** (relationship records) are out of scope — FHIR ID-based relationships don't translate cleanly to SDTM relational links.
4. **Some CDISC variables simply won't exist in EHR data** — `MHEVDTYP` (event date type: DIAGNOSIS, RELAPSE, etc.) requires investigator judgment not captured by EHR timestamps.

### Planned: COSMoS FHIR Dataset Specializations

These are on CDISC's roadmap but not yet released. When they arrive, they would work like SDTM specializations: a FHIR specialization file per BC, mapping DECs to FHIR resource/path/code, enabling metadata-driven FHIR data extraction linked to the same conceptual BC. The Medium/Chowdhury (2023) article explicitly describes this pattern — using **mCODE** (Minimal Common Oncology Data Elements) as an example FHIR IG that would provide the FHIR specialization for oncology BCs.

---

## 3. Is "Everything via SDTM" the Official Position?

### The Short Answer: No, But It's the Current Reality

**Official position**: The BC framework is explicitly standards-agnostic at the conceptual layer. CDISC says the model is "flexible and can accommodate other standards." The vision is for BCs to serve as the hub connecting CDASH, SDTM, ADaM, FHIR, OMOP, and any other standard.

**Current reality**: 
- All 360i automation work flows through SDTM as the intermediate layer
- FDA requires CDISC (SDTM) format for regulatory submissions, including RWD
- The FHIR-CDISC IG routes FHIR → SDTM, not to final analytical data
- ADaM specializations don't exist — ADaM is still generated from SDTM

### Why "Everything via SDTM" Is Architecturally Dubious

Your instinct is correct. The CDISC community is aware of this tension:

1. **Not all EHR data becomes SDTM.** EHR data used for eligibility screening, safety monitoring, or real-world evidence often doesn't need SDTM tabulation — it needs analytical-ready data (OMOP/FHIR-native). Forcing FHIR → SDTM → ADaM is an extra transformation step that adds complexity and can lose information.

2. **Certara/Abolafia's analysis (2024)**: Raises the explicit question of whether CDISC standards can accommodate all study designs, or whether OMOP or FHIR would be better native submission formats for RWD. Notes: "Should the paradigm shift from 'how do we fit non-interventional data into CDISC?' to 'what should be the future submission standard?'"

3. **The FHIR-ADaM direct path**: Some research (noted in the CDISC KB FHIR article) explores producing ADaM datasets directly from FHIR without going through SDTM — this would enable direct analysis of EHR-sourced data without the SDTM conversion step. Not mainstream but active research direction.

4. **The FDA December 2025 guidance**: Moves toward accepting de-identified RWD with audit trail captured during the study — this opens a door to non-SDTM pathways being regulatory-acceptable for device studies at minimum.

---

## 4. USDM v4 Model Constraint: `BiomedicalConcept.reference`

### The Definitive Finding

From the USDM-IG v4 data dictionary (section 12):

| Attribute | Type | Cardinality | Definition |
|---|---|---|---|
| `reference` | string | (implicit 0..1) | *"A citation to an authoritative source for a biomedical concept."* |

The UML diagram (page 21) confirms `String [0..1]` — a single optional string.

### What This Means

A USDM v4 BC instance can point to **one** specialization reference. DDF examples use this field to reference the SDTM Dataset Specialization URL. There is no first-class mechanism to simultaneously reference CRF + SDTM + FHIR specializations on a single BC.

### Available Workarounds Within v4

1. **`notes: CommentAnnotation [0..*]`** — Multiple free-text notes on the BC. Could hold additional specialization URLs, but unstructured and non-machine-readable.
2. **`BiomedicalConceptProperty.code → AliasCode.standardCodeAliases [0..*]`** — At the DEC level, multiple code aliases can carry SDTM/CDASH/LOINC/FHIR codes. This is the closest native multi-standard support, but it's at the property level, not BC level.
3. **External resolution**: Keep the USDM `reference` pointing to the CDISC Library API endpoint for the BC (`/mdr/bc/biomedicalconcepts/C111207`), and let the CDISC Library serve all specializations. The resolution happens externally, not in the USDM model itself.

### The Model Gap

For a UI that wants to show "this BC has SDTM spec X, CRF spec Y, FHIR spec Z," the USDM v4 model offers no structured container for this. This is a genuine architectural limitation — future USDM versions would need something like:

```json
"specializations": [
  { "type": "SDTM", "reference": "https://library.cdisc.org/.../BILISERPL" },
  { "type": "CRF", "reference": "https://library.cdisc.org/.../crf/BILISERPL" },
  { "type": "FHIR", "reference": "https://build.fhir.org/ig/HL7/fhir-cdisc-mapping/lb.html" }
]
```

This construct doesn't exist in v4.

---

## 5. Implications for the BC-to-Form Coverage Check Tool

### Architecture Recommendation

Given the above findings, the BC Coverage Check tool design should:

1. **Operate outside the USDM `reference` constraint** — the tool is a coverage checker, not a USDM model validator. It can resolve all specializations from COSMoS GitHub/CDISC Library by `bc_id`, regardless of what the USDM `reference` field contains.

2. **Model the specialization hierarchy explicitly**:
   ```
   BC (bc_id)
   ├── SDTM Dataset Specialization (vlm_group_id)  ← published
   ├── CRF Specialization (crf_group_id ↔ vlm_group_id)  ← draft
   ├── FHIR Specialization  ← planned (not yet available)
   └── ADaM Specialization  ← planned (not yet available)
   ```

3. **For FHIR coverage**: Until COSMoS FHIR specializations exist, the FHIR-CDISC Joint Mapping IG v1.0 provides domain-level (not BC-level) FHIR mappings. A coverage check against FHIR would need to operate at a higher granularity (domain/variable) rather than per-BC until FHIR specializations are published.

4. **"Not SDTM" flag**: The tool should acknowledge BCs/data that may not follow the SDTM path — e.g., PRO/ePRO data, observational study data, or EHR-sourced variables that will be submitted via RWD pathways. These may only have CRF or FHIR specializations in future, not SDTM.

---

## Sources

- [CDISC Biomedical Concepts](https://www.cdisc.org/cdisc-biomedical-concepts)
- [CDISC 360i Initiative](https://www.cdisc.org/standards/cdisc-360i)
- [COSMoS GitHub Repository](https://github.com/cdisc-org/COSMoS)
- [FHIR to CDISC Joint Mapping IG v1.0 — Overview](https://build.fhir.org/ig/HL7/fhir-cdisc-mapping/overview.html)
- [FHIR to CDISC Joint Mapping IG — Caveats](https://build.fhir.org/ig/HL7/fhir-cdisc-mapping/caveats.html)
- [FHIR to CDISC IG — CDISC page](https://www.cdisc.org/standards/real-world-data/fhir-cdisc-joint-mapping-implementation-guide-v1-0)
- [CDISC KB: FHIR in Clinical Research — EHR to Analysis](https://www.cdisc.org/kb/articles/use-fhir-clinical-research-electronic-medical-records-analysis)
- [CDISC KB: FHIR for RWE](https://www.cdisc.org/kb/articles/use-fast-healthcare-interoperability-resources-fhir-generation-real-world-evidence-rwe)
- [2023 CDISC Interchange BC Presentation (PDF)](https://www.cdisc.org/sites/default/files/2023-10/2023_cdisc_interchange_bc_presentation_draft.pdf)
- [Building FAIR Data Products from Clinical Studies with CDISC Biomedical Concepts — Tamer Chowdhury, Medium](https://medium.com/@tamer.chowdhury/cosmos-biomedical-concepts-an-ontology-to-create-fair-data-products-from-sdtm-87b4f9b77899)
- [Future Clinical Study Design: CDISC, FHIR, OMOP, or Hybrid Model? — Certara](https://www.certara.com/blog/future-clinical-study-design-cdisc-fhir-omop-or-hybrid-model/)
- [CDISC Roadmap](https://www.cdisc.org/cdisc-roadmap)
- [PhUSE 2024: CDISC, FHIR, OMOP, or Hybrid Model](https://www.lexjansen.com/phuse-us/2024/re/PAP_RE03.pdf)
