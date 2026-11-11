# Faro Health — USDM Platform Assessment

*Ben Mant · September 2026 · For internal discussion*

---

## Platform Positioning

Faro Health is a **protocol authoring and study startup tool**, not a traditional MDR. Their core value chain is:

1. Protocol authored (or AI-extracted from legacy documents) in **Faro Study Designer**
2. USDM-aligned JSON output generated
3. Schedule of activities pushed downstream to EDC systems (e.g. Veeva Vault EDC) via **activity matching / pointers**
4. The EDC system (Veeva or similar) then uses its own form library and MDR to build the study

The richness of CRF content, SDTM annotations, and form structure lives in the downstream EDC infrastructure. Faro's USDM is the **pointer**; the substance is in the EDC vendor's MDR.

Faro won the CDISC 2025 AI Innovation Challenge specifically for the *Protocol Library* category — digitising legacy protocol documents into structured USDM using AI extraction.

---

## What the USDM Export Actually Contains

Examination of BMS-supplied Faro-generated USDM files (ACP, BP, 331, CL005) confirms:

| Metric | Finding |
|---|---|
| BiomedicalConcept objects | **Zero** across all studies |
| Activity coding model | **100% Procedure-based** |
| Coding systems present | LOINC, CPT, IQVIA (proprietary), SPONSOR |
| BC specialization chain available | **No** |

Every activity carries procedure codes but no `bcIds`. The USDM BiomedicalConcept class is not materialised in the export at all.

### Coding Systems Observed

| System | Nature | P21-native? | SDTM-mappable? |
|---|---|---|---|
| LOINC | Public lab/clinical measurement standard | No | Domain-level only |
| CPT | US medical billing codes | No | No |
| IQVIA | Faro/IQVIA proprietary internal codes | No | No |
| SPONSOR | Sponsor-internal sequential codes (PROC1…) | No | No |

---

## The UMLS Hypothesis

The coding pattern — each activity simultaneously carrying LOINC **and** CPT **and** IQVIA codes — is consistent with a **UMLS Metathesaurus** backbone. UMLS maps multiple source vocabularies (LOINC, CPT, SNOMED, RxNorm, and others) to a single Concept Unique Identifier (CUI). If Faro maintains a UMLS-based internal concept backbone, it would naturally surface all vocabulary codes mapped to a given CUI — which is exactly what is observed.

This is distinct from COSMoS, which anchors on NCI codes with LOINC as a secondary attribute. The multi-terminology-per-activity pattern is more specific to UMLS than to COSMoS.

---

## "Inferring COSMoS Without Stating It"

Faro's CDISC membership spotlight states they use the **"Biomedical Concept BCL"** (BCL = CDISC Biomedical Concept Library / COSMoS catalog) as a foundational input to their data model. However, this BC identity is never surfaced in the USDM export as `BiomedicalConcept` objects. The internal BC resolution presumably lives inside Faro's platform — used for activity matching, specialization logic, and downstream EDC build — but does not propagate into the USDM JSON.

**Faro never explicitly names COSMoS** in public materials, but CDISC BCL and COSMoS refer to the same artifact. The gap between their claimed BC usage and what appears in the USDM export remains unexplained publicly.

---

## The Veeva Integration: Pointers, Not Substance

Faro's Veeva Vault EDC integration pushes the **schedule structure** (which activities happen at which visits) as a handoff. Veeva then matches those activities against its own form library to configure the EDC study. The procedure codes (LOINC, CPT, IQVIA) in the USDM act as activity identifiers for Veeva to match against — not as SDTM-resolved data.

This is consistent with what has been observed in Faro demos: activity matching to Veeva, creating study instances — schedule and visit structure transfer, not full CRF/SDTM metadata transfer.

---

## AI Extraction from Legacy Protocols

The CDISC AI Innovation Challenge win for the Protocol Library category explains much of the coding pattern seen in Faro USDM exports. When AI extracts a legacy protocol document:

- Labs already referenced as LOINC codes in the document → LOINC appears in USDM
- Procedures referenced as CPT codes in billing/operational context → CPT appears in USDM  
- IQVIA internal codes from Faro's own activity library → IQVIA appears in USDM

This is an artefact of AI extraction from legacy source material, not a hand-curated BC assignment. The PROC1, PROC2… SPONSOR codes similarly reflect activities that had no standardised code in the source document.

---

## Implications for USDM Workbench Consumption

### What works with Faro-generated USDM
- **SoA display**: fully derivable — activities, visits, schedule structure intact
- **TA, TE, TV, TI derivation**: unaffected — these domains come from arms, epochs, elements, encounters, criteria, not from BC/procedure coding
- **Form-event association (Approach A)**: possible via fuzzy name match from procedure name to P21 form library
- **Rough domain classification**: LOINC codes can suggest domain placement (labs → LB, ECG → EG) at domain level only

### What does not work without additional data
- **COSMoS specialization chain**: no `bc_id` to start from
- **SDTM variable-level derivation**: not possible
- **VLM, codelist subsetting**: not possible
- **IQVIA code resolution**: proprietary to Faro's infrastructure — no public crosswalk

### Recommended workbench handling
The tool should detect operating mode on import — check whether any activities carry non-empty `bcIds`. If none are present, declare **Procedure Mode** to the user: SoA and TDD derivation available; CRF and SDTM specialization generation requires either BC respin of the study or an explicit sponsor-provided crosswalk from procedure codes.

---

## Comparison to OpenStudyBuilder

| Dimension | Faro Health | OpenStudyBuilder |
|---|---|---|
| Internal BC model | UMLS-likely + CDISC BCL (inferred) | Activity Instance Classes with NCI codes (visible in UI) |
| USDM BC export | Not materialised | Not materialised |
| Coding in USDM | LOINC, CPT, IQVIA, SPONSOR | Procedure (bare) |
| Gap nature | Architectural / proprietary backbone | Implementation gap with visible roadmap fix |
| 2026 roadmap | Not public on BC export | "Much more information from OSB metadata" committed |
| MDR functionality | No — protocol authoring + startup | Protocol authoring + CRF library (OSB MDR) |

OSB's gap is more tractable: NCI concept IDs already exist internally and the 2026 roadmap explicitly commits to improving USDM BC export. Faro's gap may be more structural if their backbone is UMLS-based rather than NCI-anchored, as UMLS CUIs don't cleanly map to COSMoS `bc_id` values.

---

## Data Quality Warning: These May Not Be Real Study Exports

> **Conclusion: treat the BMS-supplied Faro files as demo stubs, not as representative examples of Faro's production output. No strategic conclusions should be drawn from their study-level content.**

Cross-study analysis of the four BMS files reveals a pattern of hardcoded defaults and structural contradictions that are inconsistent with real study exports:

| Field | Hardcoded value | Reality check |
|---|---|---|
| Intervention model | Always `SINGLE GROUP` (C82640) | 331-201 is a three-arm randomised trial; ACP-2566 title says "Parallel-Group" |
| Planned age range | Always 18–100 years | ACP-2566 is limited to females aged 5–20 |
| Indication code | Always C3201, code system = SPONSOR | Studies span PTSD, Rett syndrome, colorectal cancer, IgA nephropathy |
| StudyVersion approval date | Always 2024-01-01 | Version identifiers include "07 August 2020", "Dated 17 July 2015" |
| Sponsor details | Placeholder text throughout | — |
| Intervention role | Always `Experimental Intervention` | Placebos present but not coded as such |
| First visit of each epoch | Always Day -1 | Causes CL005 timeline to run backwards |

BP29262 is the most egregious case: the study title describes a randomised, double-blind comparison of vanucizumab and bevacizumab, but the USDM contains one arm, one intervention, and one "Placeholder study element." This cannot be an accidental omission — the tool simply did not populate the design.

**The most probable explanation** is that Faro's export produces a structural skeleton from AI-extracted protocol text, with certain fields initialised from template defaults. The sponsor or Faro's team is expected to review and correct before downstream use. For a demo or sponsor POC, that correction pass did not happen.

**What this means for the procedure-coding observation:** the finding that all activities are procedure-coded (LOINC, CPT, IQVIA, no `bcIds`) is still valid — that reflects the AI extraction engine's output regardless of whether the study-level metadata was corrected. But any analysis of arms, epochs, criteria, timing, or indication from these specific files should be treated as noise.

---

## Sources

- [Faro | CDISC Membership Spotlight](https://www.cdisc.org/membership/spotlight/faro)
- [Faro Wins CDISC 2025 AI Innovation Challenge](https://www.newswire.com/news/faro-wins-cdisc-2025-ai-innovation-challenge-for-protocol-library-22647080)
- [Faro Health Joins Veeva Product Partner Program](https://finance.yahoo.com/news/faro-health-joins-veeva-product-110000617.html)
- [Merck co-leads Faro AI $37.3M Series B](https://runtimewire.com/article/faro-ai-series-b-merck-clinical-development-agents)
- [Faro | Workflow Automation](https://farohealth.com/workflow-automation)
