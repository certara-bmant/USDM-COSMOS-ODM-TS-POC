# USDM → CRF / SDTM Mapping: Approaches, Risks, and Conceptual Boundaries

*Ben Mant · September 2026 · For internal discussion*

---

## The Core Problem

A USDM study file describes a protocol: what activities are scheduled, which Biomedical Concepts (BCs) each activity collects, and when. Converting that into real data collection artefacts — a CRF form and an SDTM define — requires a mapping step. There are two materially different ways to approach this, with different feasibility and fidelity trade-offs.

---

## Approach A: Direct Activity-to-Form Linkage (Annotation-Based)

An activity in the USDM schedule is matched directly to a P21 form — either exactly (where form IDs are known) or via fuzzy matching on activity name, BC label, or question text. The form-to-event association in P21 is then written via the ScheduleOfActivities sheet (`Event → Form`), which is the correct P21 mechanism.

**What this gives you:** a working CRF association without requiring COSMoS specialization data. Where the fuzzy match correctly identifies a BC, some semantic context is recovered. This approach has practical value when working with a bespoke form library not anchored by CDISC — it can facilitate a study-level schedule build without requiring a fully structured specialization chain. However, it requires manual confirmation: the activity-to-form match is a high-level merge and does not resolve at the granular item level that specialization provides. It is also important to note that not all USDM activities will have an associated form — administrative, logistical, or protocol-only activities have no CRF counterpart and should not be force-matched.

**Where it breaks down:** downstream SDTM linkage from P21 / CDASH is annotation-based rather than structurally enforced. Fields such as `sdtm_target_variable`, `sdtm_annotation`, and `mapping_instructions` carry SDTM intent as free text (e.g. `"VS.VSORRESU"`). These express a relationship by convention — there is no foreign key from the CRF item record to an SDTM variable record that a system can join on. A process attempting to act on this computationally (codelist subsetting, VLM construction, validation) must parse strings rather than follow a defined key relationship. The result is documentation-quality linkage: sufficient for human review, insufficient for automated artefact generation.

**Caveat — end-to-end linked models exist but are not feasible in P21 MDR.** Platforms such as Ryze implement a genuinely linked model where CRF items carry typed, traversable references to their downstream SDTM target objects — the link from form item to SDTM variable is a first-class relationship in the data model, not a text field. In principle this is what Approach A would need to be fully generative. However, P21 MDR does not currently support this structure: SDTM linkage on a P21 form item is stored as annotation text, not as a joinable reference to a variable record in the define. Any approach operating within P21 MDR is therefore bounded by this limitation; the annotation-quality ceiling is a platform constraint, not a design choice that can be engineered around within the current P21 data model.

---

## Approach B: COSMoS Specialization Layer (Key-Linked)

The COSMoS BC catalog, CRF Specialization (`cosmos_crf_item`), and SDTM Dataset Specialization (`cosmos_sdtm_item`) together form a structured, key-linked chain:

```
USDM Activity → bc_id → vlm_group_id → crf_group_id → cosmos_crf_item (CRF questions)
                                      └─────────────→ cosmos_sdtm_item (SDTM variables)
```

`dec_id` is the shared key that appears on both the CRF and SDTM specialization rows, creating a joinable relationship between the two sides rather than a text annotation. Critically, the specialization also encodes constraints that terminology alone cannot provide: which unit codelist applies to a specific analyte (`LBORRESU` for a glucose BC is a concentration unit subset; for haemoglobin it is a mass-concentration subset), and which test codes belong to which where-clause conditions for VLM.

**What this gives you:** machine-generatable CRF form items, SDTM variables, VLM rows, codelist subsets, and WhereClause definitions directly from confirmed BC selections — without human editorial judgement at each step.

---

## Risks and Difficulties of the Specialization Approach

- **Coverage is partial and evolving.** COSMoS CRF Specialization currently covers ~16 domains; SDTM Specialization covers ~1,475 specializations as of mid-2026. Gaps exist, particularly in laboratory sub-domains. Any build on top of this must surface "no match" states honestly rather than imply complete coverage.

- **Customers cannot rely solely on COSMoS.** For many sponsors, COSMoS provides a useful starting point but will not cover their full BC inventory. Proprietary endpoints, sponsor-specific assessments, and novel biomarkers are unlikely to have published COSMoS specializations. In practice, a complete specialization layer will need to be supplemented — or in some cases predominantly sourced — from the sponsor's own past submissions: historical SDTM defines, legacy variable metadata, and previously approved VLM structures. A tooling approach that treats COSMoS as the only specialization source will have significant coverage gaps in real studies.

- **The BC library is anchored entirely to NCI Thesaurus, which may be a weak backbone for full clinical coverage.** COSMoS BCs derive their identity from NCI concept codes. NCI Thesaurus is broad but is not exhaustive for clinical research specifics — novel biomarkers, highly specific analyte sub-types, sponsor-defined composite endpoints, and emerging therapeutic targets frequently lack an NCI concept code or are represented only at a level too coarse to anchor a meaningful specialization. Where no NCI code exists, a BC cannot fully participate in the COSMoS framework, and any tooling that requires a valid `bc_id` → NCI code chain as its entry point will fail silently for these concepts. This is a structural ceiling on COSMoS coverage, not a gap that will close as more specializations are authored — it is bounded by the NCI ontology itself.

- **Sponsor-specific BCs may need to be migrated from existing internal systems.** Many sponsors already operate internal BC-equivalent libraries — endpoint catalogs, standard variable sets, metadata registries, or legacy EDC assessment libraries — built before COSMoS existed and not aligned to its NCI-anchored model. Migrating these into a USDM-compatible BC structure requires either finding the correct NCI concept code for each internal concept (often not straightforward), accepting a `BcSurrogate` designation where no code exists, or maintaining a crosswalk between internal identifiers and `bc_id` values. The further a sponsor's internal library diverges from CDISC/NCI terminology, the more of this migration becomes bespoke mapping work rather than a lookup — and any unmigrated concepts will fall outside the specialization-driven generation path entirely until they are resolved.

- **CRF specialization requires additional alignment for bespoke form libraries.** COSMoS CRF Specializations are authored against CDASH — they assume a CDASH-centric form structure. Sponsors with established bespoke form libraries (custom sections, proprietary item IDs, non-CDASH question structures) will find that COSMoS `crf_group_id` and `crf_item` values do not map directly to their existing library items. Bridging this will require some form of alias or crosswalk mechanism — linking a sponsor's local form item identifier to its COSMoS/CDASH counterpart — before the specialization can drive generation against their existing library rather than producing a parallel, disconnected form structure.

- **BcSurrogates are a structural gap.** Where no SDTM Dataset Specialization exists for a concept, USDM encodes it as a `BcSurrogate`. These are excluded from specialization-driven generation and must be handled as explicit gaps, not errors.

- **A single bc_id can resolve to multiple specializations.** The same BC can appear in genuinely distinct specializations (different domains, different `vlm_group_id` values) — e.g. glucose in both LB and FA contexts. Activity context and BC category help disambiguate but are not guaranteed to be conclusive. User confirmation is required where ambiguity remains; a system that silently picks one is producing incorrect output.

- **The specialization is a point-in-time artefact.** Generated study artefacts are frozen at the specialization version used. COSMoS releases will change existing field values. Re-generation after a new release must be a deliberate, versioned act — not a silent overwrite.

- **P21 does not hold `dec_id` on native variable objects.** The link from a P21 variable back to its BC is not natively stored. Round-tripping — reading a generated define back and re-linking it to a specialization — requires a matching strategy that must be designed explicitly, not assumed.

---

## Approach C: Direct Conceptual-to-Downstream Object (Skipping the Implementation Layer)

A third approach — less explicitly named but present in practice — maps directly from the USDM BC or activity to a downstream P21 artefact (form item or SDTM variable) without routing through a specialization intermediary at all. The conceptual object and the downstream object are treated as directly equivalent: if the BC is "Systolic Blood Pressure", a VS form item and a `VSORRESU` variable are generated by direct assertion rather than by resolving a specialization chain.

This can appear attractive because it eliminates the dependency on COSMoS coverage and avoids the BC migration and alias problems described above. Where the mapping is made by a domain expert who already knows the right answer, the output may be correct. The problems are structural rather than immediately visible.

**What is lost by skipping the implementation layer:**

- **No codelist subsetting rationale.** The mapping produces `mmHg` for `VSORRESU` but carries no record of why — no `assigned_value` from a specialization, no `value_list` constraint. The output is correct for the person who built it; it is opaque to any downstream system or reviewer trying to understand or replicate the derivation.
- **No VLM structure.** WhereClause conditions — which test code pairs with which unit subset, which result range is valid for which analyte — cannot be derived without the specialization's `value_list` and `assigned_value` fields. VLM produced under this approach must be manually authored variable by variable, which is exactly the status quo this tooling is meant to improve on.
- **No principled handling of multiple valid implementations.** The same BC can legitimately map to different SDTM variables, unit subsets, or CRF layouts depending on context. The specialization layer encodes which interpretation applies; direct conceptual-to-output mapping picks one silently and provides no mechanism for a different sponsor or context to pick another.
- **Traceability is lost.** An artefact generated directly from a concept carries no audit trail linking it to a standard — no `dec_id`, no `vlm_group_id`, no specialization version. Regulatory reviewers, future maintainers, and downstream tooling cannot determine how the mapping was derived or whether it is consistent with any published standard.

**The USDM conflation is the mechanism that makes Approach C appear to work.** The way sponsors or tool vendors typically operationalise direct conceptual-to-output mapping is by embedding the implementation decisions into the USDM file itself — encoding `VSTESTCD = SYSBP`, unit values, or form field identifiers as properties of USDM activities or BCs. This is exactly the d4k pattern. The USDM then appears to "contain" the downstream artefact information, making the mapping trivial to execute. But the cost is that the USDM file is no longer a protocol description — it is a pre-resolved implementation, specific to one sponsor's choices, locked to one submission context, and unreusable by any party that would make different (equally valid) implementation decisions. The implementation layer has not been skipped; it has been folded into the protocol layer where it does not belong and cannot be separated out later.

---

## The Conflation Problem

USDM is a protocol-level standard; the specialization layer exists to resolve implementation decisions separately. Some community examples (notably certain d4k-derived files) embed implementation detail — SDTM variable names, codelist submission values, form field IDs — directly into USDM activities or BCs. This conflates the two layers: the protocol now implies a single downstream implementation, making the file sponsor-specific and non-reusable. It sidesteps the specialization coverage problem by hardcoding one answer rather than solving it.
