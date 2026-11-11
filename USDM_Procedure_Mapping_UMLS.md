# USDM Procedure Mapping via UMLS — Approach, Licensing & Limitations

*Ben Mant · September 2026 · For internal discussion*

---

## The Core Problem

USDM exports from tools like Faro Health and OpenStudyBuilder surface procedures at the **activity level** — a single row represents a clinical test order (e.g. "Chemistry Panel (Serum)", "Vital Signs", "Hematology / CBC"). COSMoS Biomedical Concepts operate at the **variable level** — one BC per measured endpoint (Albumin, Systolic Blood Pressure, Hemoglobin). A direct name-match or TF-IDF similarity between a procedure name and a BC name fails for any panel or composite procedure, because the procedure name deliberately obscures the analyte detail that COSMoS requires.

UMLS resolves this gap. It is the only public resource that explicitly encodes the **composition of clinical panels** — mapping a single procedure code (CPT, LOINC) to its constituent analytes — and simultaneously cross-references those analytes to SNOMED, LOINC, and NCI codes that anchor COSMoS BCs.

---

## Proposed Mapping Pipeline

```
USDM usdm_procedure row
        │
        ├─ procedure has CPT code?
        │     Yes ──► UMLS Metathesaurus: CUI lookup by CPT code
        │                     │
        │               Is CUI a panel / composite?
        │                     │
        │               Yes ──► Retrieve component atoms (UMLS /content/current/CUI/{cui}/atoms)
        │                     │    → each component has its own CUI + CPT/LOINC
        │                     │
        │               No  ──► Treat as single analyte, proceed to step below
        │
        ├─ procedure has LOINC code?
        │     Yes ──► UMLS CUI lookup by LOINC; LOINC panels also expand to parts
        │
        └─ procedure name only (OSB)?
              ──► TF-IDF / fuzzy match against COSMoS short_name + synonyms
                  (current approach; inherits the granularity mismatch for panels)

For each resolved analyte CUI:
        │
        ├─ Cross-reference to NCI code (UMLS source = NCI)
        │
        └─ Match NCI code against COSMoS bc_id
              │
              ├─ Direct hit  ──► Confirmed BC assignment (confidence: EXACT)
              └─ No hit      ──► Fall back to TF-IDF on analyte name
                                  (confidence: INFERRED)
```

---

## Why UMLS Works Here

The UMLS Metathesaurus aggregates over 200 source vocabularies under Concept Unique Identifiers (CUIs). For clinical panels, the MSH (MeSH), CPT, and LOINC sources all encode compositional relationships:

| Panel | CPT | Component analytes (CPT) |
|---|---|---|
| Comprehensive Metabolic Panel | 80053 | Albumin (82040), ALT (84460), ALP (84075), AST (84450), Bilirubin total (82247), BUN (84520), Calcium (82310), CO₂ (82374), Chloride (82435), Creatinine (82565), Glucose (82947), K (84132), Na (84295), Total Protein (84155) |
| CBC with differential | 85025 | WBC (85048), RBC (85041), Haemoglobin (85018), Haematocrit (85014), MCV, MCH, MCHC, Platelets (85049), Differential (85007) |
| Thyroid Panel (TSH) | 84443 | TSH — single analyte; no expansion needed |
| Urinalysis (non-auto) | 81001 | pH, specific gravity, protein, glucose, ketones, bilirubin, blood, nitrite, leucocyte esterase |

Each component analyte CPT code has a UMLS CUI, and each CUI cross-references to LOINC and NCI codes. COSMoS BCs carry `bc_id` values from NCI (e.g. C105585 = Serum Albumin Measurement), making the join tractable.

Faro's multi-code pattern (LOINC + CPT + IQVIA per activity) is consistent with UMLS being Faro's internal backbone: the Metathesaurus naturally surfaces all source vocabulary codes for a CUI, which is exactly what appears in the Faro USDM export.

---

## UMLS API Access — Licensing Considerations

### What a UMLS licence covers

UMLS is produced by the US National Library of Medicine (NLM) and is **free to use but requires registration and acceptance of a licence agreement**. The licence covers access to the Metathesaurus, Semantic Network, and SPECIALIST Lexicon — including the REST API (UTS), full bulk downloads, and derivative works built on top of them.

The NLM licence is permissive for research and commercial software development: it does not restrict the number of queries, charge royalties, or prohibit embedding UMLS-derived mappings in a commercial product. Attribution to NLM is required.

### The critical complication for Certara / Pinnacle 21

UMLS is a **container licence only**. The Metathesaurus aggregates over 200 source vocabularies, each of which carries its **own separate licence**. A UMLS API key grants access to query those vocabularies through the NLM interface, but it does not grant the right to redistribute or surface their content in a commercial product. The sub-licences that matter most for procedure mapping are:

| Source | Required for panel mapping | Licence model | Cost |
|---|---|---|---|
| **CPT (AMA)** | Yes — Faro uses CPT codes; panels expand via CPT component lists | AMA Data File Agreement — negotiated commercial licence | Annual fee; typically $X,000s/year for software vendors |
| **LOINC** | Yes — Faro uses LOINC codes; LOINC Part expansion available | Regenstrief LOINC Licence — free, attribution required | None |
| **SNOMED CT** | Incidentally (cross-references) | SNOMED International + national release licence | Free in most countries via national licence; US: free via NLM |
| **NCI Thesaurus** | Yes — COSMoS bc_id values are NCI codes | US government public domain | None |
| **MeSH** | Background cross-reference only | US government public domain | None |
| **MedDRA** | Not needed for procedure-to-BC mapping | ICH/MSSO licence | Restricted to licence holders; Certara/P21 likely already licenced |
| **RxNorm** | Not needed | US government public domain | None |

### CPT licensing — the key risk

The AMA treats CPT codes as a proprietary data product. **Displaying, storing, or transmitting CPT codes in a commercial software product requires an AMA Data File Licence**, regardless of whether the codes were sourced from UMLS, a public API, or a sponsor's USDM file. This applies to:

- Showing CPT codes in a workbench UI (even read-only)
- Storing CPT codes in a workbench database or output file derived from USDM input
- Using CPT codes as lookup keys for panel expansion logic

Certara and Pinnacle 21 almost certainly already hold AMA licences for existing P21 Community / Enterprise features (SDTM domain coverage, lab reference ranges, etc.). The question is whether the existing licence scope covers USDM workbench use, or whether a licence amendment is needed. This needs legal/procurement confirmation before any CPT-driven panel expansion goes to production.

**LOINC, NCI, and MeSH have no equivalent restriction.** A mapping pipeline that resolves procedures to COSMoS BCs using only LOINC and NCI codes (skipping CPT display and CPT-based expansion) could be deployed with zero licence cost beyond NLM UMLS registration.

### Practical licensing path for a USDM workbench

The lowest-risk approach is a two-tier design:

**Tier 1 — LOINC + NCI only (no additional licence cost):** Use LOINC codes present in the USDM input to drive panel expansion via the LOINC Parts link table (freely available from Regenstrief). Map component analyte LOINC codes → NCI codes → COSMoS bc_id. This covers all Faro LOINC-coded procedures cleanly.

**Tier 2 — CPT expansion (AMA licence required):** For procedures where only a CPT code is present (no LOINC equivalent), use UMLS CUI lookup on the CPT code to expand to component analytes, then map to COSMoS. Gated behind AMA licence confirmation.

IQVIA/SPONSOR codes fall outside both tiers and require manual curation regardless of licensing.

### UTS API limits

The UMLS REST API is available at `https://uts-ws.nlm.nih.gov`. Key parameters:

- **Authentication:** API key issued on UMLS account creation (free, no approval wait for most users)
- **Rate limits:** No formal published cap; NLM requests "reasonable use"; bulk panel lookups should use the UMLS full download (MRCONSO, MRREL tables) rather than the live API
- **Panel expansion query:** `GET /content/current/CUI/{cui}/relations?relationLabel=has_part` — returns component CUIs for panel concepts
- **Code-to-CUI lookup:** `GET /search/current?string={code}&sabs=CPT` or `sabs=LNC`

---

## Known Limitations

### 1. Sponsor-specific panel codes are not in UMLS

Faro USDM exports contain `IQVIA` and `SPONSOR` codes (e.g. `PROC1`, `PROC2`, `NP113`, `T9207`) that have no representation in UMLS or any public vocabulary. These are Faro's internal activity library identifiers, not standardised codes. For these entries:

- UMLS CUI lookup returns no match
- The procedure name (if sufficiently specific) may still yield a TF-IDF BC candidate, but with lower confidence
- Panel expansion is impossible — the component analytes are unknown without access to Faro's internal activity library or the original protocol document

This affects a meaningful proportion of Faro activity rows. In the BMS-331 export, approximately 30–40% of procedure code rows carry `IQVIA` or `FARO00` (placeholder) CPT codes with no external mapping available. These require manual curation or a direct data request to Faro.

### 2. CPT panel definitions change over time

CPT code compositions are revised annually by the AMA. Panel 80053 (CMP) has a fixed analyte list, but other panels have changed composition between editions. UMLS reflects a specific release date; a workbench using the UMLS API should record the UMLS version used for traceability.

### 3. LOINC panel parts require the LOINC Parts system

LOINC has its own panel expansion mechanism (`LOINC_PARTS`) separate from UMLS. For LOINC panel codes (e.g. `57021-8` = CBC with differential), the LOINC Parts link table is more reliable than UMLS for panel composition. Both paths lead to the same analyte LOINC codes, but the LOINC Parts route is simpler and doesn't require UMLS traversal.

### 4. COSMoS BC coverage gaps

COSMoS does not have BCs for every clinical analyte. Sponsor-specific biomarkers (proteomics, pharmacogenomics, RNA), niche psychiatric scales, and procedure-administrative activities (Drug Accountability, IRT Randomization) have no COSMoS BC and cannot be mapped regardless of UMLS resolution. These remain permanently in the "no match" category and should be flagged for manual SDTM annotation or suppressed from COSMoS-based derivation.

### 5. Activity vs. variable granularity mismatch for OSB

OpenStudyBuilder exposes procedure names without external codes. For bare names like "Glucose Metabolism" or "Lipids", the activity groups multiple analytes without naming them. UMLS lookup requires a code entry point; without CPT or LOINC, the only option is fuzzy text matching at the activity level, which returns a high-level BC (e.g. "Lipid Panel" rather than individual HDL, LDL, Total Cholesterol BCs). OSB's roadmap to export richer metadata (including NCI codes from its internal Activity Instance Classes) would close this gap when delivered.

---

## Recommended Workbench Handling

For a production USDM workbench consuming procedure-coded studies, the suggested logic is:

1. **CPT code present → UMLS panel expansion → analyte-level COSMoS BC matching** (highest confidence)
2. **LOINC code present → LOINC Parts expansion → analyte-level COSMoS BC matching** (equivalent)
3. **IQVIA / SPONSOR code only → procedure name TF-IDF → BC candidate list** (medium confidence; flag for review)
4. **No code, procedure name only (OSB) → TF-IDF → BC candidate list** (medium confidence; flag for review)
5. **No code, generic procedure name → no match** (flag as "requires manual curation")

All mappings should carry a `confidence` level and a `mapping_source` field (UMLS_PANEL / LOINC_PARTS / TFIDF / MANUAL) for downstream traceability. The COSMoS `bc_id` assignments that emerge from steps 1–2 can be treated as near-certain; steps 3–4 require human sign-off before being used to drive SDTM variable derivation.

---

## Sources

- [UMLS Licence Agreement — NLM](https://www.nlm.nih.gov/research/umls/knowledge_sources/metathesaurus/release/license_agreement.html)
- [UTS REST API Documentation](https://documentation.uts.nlm.nih.gov/rest/home.html)
- [AMA CPT Licence Information](https://www.ama-assn.org/practice-management/cpt/ama-cpt-licensing-overview)
- [LOINC Licence — Regenstrief Institute](https://loinc.org/license/)
- [SNOMED CT Licence — SNOMED International](https://www.snomed.org/snomed-ct/get-snomed)
- [COSMoS BC Catalog — CDISC](https://www.cdisc.org/cosmos)
- CPT 80053 Comprehensive Metabolic Panel: [UMLS CUI C0519825](https://uts.nlm.nih.gov/uts/umls/concept/C0519825)
