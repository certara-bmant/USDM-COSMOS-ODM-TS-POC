# USDM MDR POC — Technical Summary

## How the POC Was Built

### Source Files

| File | Role |
|---|---|
| `EliLilly_NCT03421379_Diabetes (1).json` | Primary data source — LY900018 USDM v4.0 (from CDISC DDF-RA GitHub) |
| `usdm_output_V4.0 (1).json` | Original TJ301 UC study — used to identify BcSurrogate pattern |
| `odm 2.0/ODM-study.xsd` | Verified ItemGroupDef nesting, ItemGroupTypeType, WhereClauseDef |
| `odm 2.0/ODM-enumerations.xsd` | Confirmed Type=Concept/Form/Dataset/Section |
| COSMoS GitHub (`cdisc-org/COSMoS`) | CRF Specialization draft CSV; BC model YAML |

---

## USDM JSON → Data Tables

### Object graph structure

```
d['study']['versions'][0]
  .biomedicalConcepts[]     → cosmos_bc, cosmos_dec, activity_bc (link_type='BC')
  .bcSurrogates[]           → usdm_bc_surrogate, activity_bc (link_type='Surrogate')

d['study']['versions'][0]['studyDesigns'][0]
  .epochs[]                 → usdm_epoch
  .encounters[]             → usdm_encounter
  .activities[]             → usdm_activity  (.biomedicalConceptIds + .bcSurrogateIds)
  .arms[]                   → usdm_study_cell (arm dimension)
  .studyCells[]             → usdm_study_cell
  .elements[]               → usdm_study_element
  .scheduleTimelines[0]     → Main Timeline (15 SAIs)
  .scheduleTimelines[1]     → Hypoinduction sub-timeline (5 SAIs)
  .scheduleTimelines[2]     → Treatment Phase sub-timeline (15 SAIs)
  .scheduleTimelines[0].instances[]   → usdm_sai
  .scheduleTimelines[0].timings[]     → timing_day, timing_type on usdm_sai
```

**Important:** `bcSurrogates` lives on `studyVersion`, not `studyDesign`.

### Linked-list traversal

Epochs, encounters, activities and SAIs are ordered via linked lists. SAIs use `defaultConditionId` as the next-pointer:

```python
all_next = {i['defaultConditionId'] for i in instances if i['defaultConditionId']}
first    = [i for i in instances if i['id'] not in all_next][0]
ordered  = []
cur = first
while cur:
    ordered.append(cur)
    cur = id_to_inst.get(cur['defaultConditionId'])
```

### Multiple timelines

The LY900018 study has 3 timelines. `mainTimeline=True` identifies the primary visit schedule. Sub-timelines (Hypoinduction, Treatment Phase) contain intra-day time-points and would be linked via `SAI.sub_timeline_id` (null in this file — the sub-timelines exist as standalone objects, not yet branched from the main SAIs).

### Timing day labels

The Main Timeline timings are not directly matched to SAI names by label (several share the same label e.g. "-1 Day" appears twice — once for Period 1 and once for Period 2). Day labels were manually mapped by SAI name and position:

```python
day_labels = {
    'SCREENING': 'Day -28',
    'P1_DAY_MINUS1': 'Day -1',
    'DAY_1_RANDOM': 'Day 1 (Rand)',
    'P1_PRE_INFUSION': 'Day 1 (Pre)',
    ...
    'FOLLOW_UP': 'Day 28+',
}
```

---

## BiomedicalConcept vs BcSurrogate

### Two entity types for the same role

When USDM links a protocol activity to a measured concept, it uses one of two entity types:

**BiomedicalConcept** — full reference to a published standard:
- `reference: /mdr/specializations/sdtm/packages/{date}/datasetspecializations/{TESTCD}` (SDTM DS path)
- `reference: /mdr/bc/packages/{date}/biomedicalconcepts/{NCItID}` (COSMoS BC path)
- `code.standardCode.code` = CDISC CT code for the TESTCD value (e.g. C49678 = RESP)
- `properties[]` = SDTM variable codes (VSTESTCD, VSORRES, VSORRESU, VSDTC etc.)
- Lives on `studyVersion.biomedicalConcepts`

**BcSurrogate** — placeholder for unpublished concepts:
- `reference: "None set"` — no CDISC standard reference exists yet
- No `code` or `properties` — just name, label, description
- Lives on `studyVersion.bcSurrogates`
- Referenced by activities via `bcSurrogateIds` (alongside `biomedicalConceptIds`)

### Activity BC count

Activity `bc_count` must sum both:
```python
bc_count = len(a.get('biomedicalConceptIds',[])) + len(a.get('bcSurrogateIds',[]))
```

In LY900018:
- Clinical Lab Tests: 25 BCs + 16 surrogates = 41
- Clinical Serology Tests: 0 BCs + 4 surrogates = 4
- Vital Signs: 4 BCs + 0 surrogates = 4

### SQL model

```sql
CREATE TABLE usdm_bc_surrogate (id, name, label, description, reference, type);
-- activity_bc covers both:
CREATE TABLE activity_bc (activity_id, bc_or_surr_id, name, code, ref_id, link_type);
-- link_type: 'BC' | 'Surrogate'
```

---

## BC Code Clarification

The `code` on a USDM BiomedicalConcept is the **CDISC CT code for the SDTM TESTCD value**, not a generic NCIt concept ID. Examples from LY900018:

| BC Name | BC code | Decodes to | Meaning |
|---|---|---|---|
| Respiratory Rate | C49678 | RESP | VSTESTCD value for respiratory rate |
| Systolic Blood Pressure | C25298 | SYSBP | VSTESTCD value for SBP |
| Diastolic Blood Pressure | C25299 | DIABP | VSTESTCD value for DBP |
| Pulse | C49676 | PULSE | VSTESTCD value for pulse |
| Temperature | C174446 | TEMP | VSTESTCD value for temperature |
| Hematocrit of Blood | C64796 | HCTBLD | LBTESTCD value for haematocrit |
| HbA1c | C64849 | HBA1CBLD | LBTESTCD value for HbA1c |

These are not the same as NCIt concept IDs for the physiological measurements (e.g. C3826 = Respiratory Rate the concept). The USDM model links at the SDTM implementation level.

---

## COSMoS CRF Specialization → ODM 2.0

Source: `export/cdisc_crf_specializations_draft.csv` (COSMoS GitHub, package_date 2026-06-30). 2073 rows across 16 domains including VS (112 rows), EG (243 rows), LB (976 rows). CSV and XLSX are identical and in sync with CDISC Library.

| COSMoS CRF field | ODM 2.0 target | Notes |
|---|---|---|
| `crf_group_id` | `ItemGroupDef/@OID` | |
| `crf_item` / `variable_name` | `ItemDef/@OID` / `@Name` | |
| `question_text` | `ItemDef/Question/TranslatedText` | Full-form |
| `prompt` | `ItemDef/Question/TranslatedText` | Grid/table |
| `order_number` | `ItemRef/@OrderNumber` | |
| `mandatory_variable` | `ItemRef/@Mandatory` | |
| `data_type` | `ItemDef/@DataType` | |
| `codelist` | `ItemDef/CodeListRef/@CodeListOID` | |
| `value_list` / `value_display_list` | `CodeList/EnumeratedItem` | |
| `sdtm_target_variable` | SDTM annotation | |
| `selection_type` | No ODM 2.0 equivalent | Vendor extension |
| `prepopulated_term/code` | No ODM 2.0 equivalent | Vendor extension |

---

## ODM 2.0 XSD Verification

### ItemGroupDef nesting (from ODM-study.xsd)
```xml
<xs:group name="ItemGroupDefGroup">
    <xs:sequence>
        <xs:element ref="ItemGroupRef" minOccurs="0"/>
        <xs:element ref="ItemRef"      minOccurs="0"/>
    </xs:sequence>
</xs:group>
```

### ItemGroupTypeType (from ODM-enumerations.xsd)
```xml
<xs:enumeration value="Concept"/>
<xs:enumeration value="Dataset"/>
<xs:enumeration value="Form"/>
<xs:enumeration value="Section"/>
```

### WhereClauseDef — compound conditions
```xml
<xs:element ref="RangeCheck" minOccurs="1" maxOccurs="unbounded"/>
```
Multiple RangeChecks per WhereClause → AND logic.
Example: `VSTESTCD EQ SYSBP AND VSPOS EQ SITTING` → separate conditioned ItemDef for supine vs. seated SBP.

---

## HTML Architecture

Single self-contained file (~100KB). No server required. Handsontable (~2MB) lazy-loaded from cdnjs only when Data Tables tab is first opened.

### Static SoA generation

The SoA table is generated in Python and embedded as static HTML. This avoids all rendering latency:

```python
for grp_label, act_ids in GROUPS:
    tbody += f'<tr class="rh-grp"><td colspan="{n+1}">{grp_label}</td></tr>'
    for aid in act_ids:
        bc_count = len(a.get('biomedicalConceptIds',[])) + len(a.get('bcSurrogateIds',[]))
        bc_tag = f' <span class="bc">BC×{bc_count}</span>' if bc_count else ''
        for v in sai_visits:
            has = aid in v['actIds']   # v['actIds'] = Python set from SAI.activityIds
            ...
```

### Treatment assignment row

The SoA thead contains a `<tr class="rh-tx">` row (dark green) showing which arm receives which treatment in each epoch. Spans are driven by epoch_spans from the parsed data:
- Period 1: LY-G = LY900018 · G-LY = Glucagon
- Period 2: LY-G = Glucagon · G-LY = LY900018 (sequence reversed)

### Lazy Handsontable
```javascript
function initTables() {
    var js = document.createElement('script');
    js.src = 'https://cdnjs.cloudflare.com/.../handsontable.full.min.js';
    js.onload = function() { hotLoaded = true; showTable('usdm_study', ...); };
    document.head.appendChild(js);
}
```
Called only on first click of the Data Tables tab.

---

## Limitations and Next Steps

| Area | Current state | Next step |
|---|---|---|
| BC reference | SDTM DS path (TESTCD codes). No COSMoS BC path refs in LY900018 | Confirm if any studies use COSMoS BC path; align on canonical ref type |
| BcSurrogates | 20 surrogates in this file — common lab analytes not yet in CDISC library | Monitor CDISC library releases; migrate surrogates to BCs as DSes are published |
| CRF Specialization | 16 domains incl. VS/EG/LB (2073 rows, 2026-05-14). CSV = XLSX = CDISC Library. | Monitor for new domains (FT, SU etc. rows may expand) |
| Sub-timeline linking | Hypoinduction + Treatment Phase are standalone; SAI.sub_timeline_id = null | Wire SAIs to sub-timelines when USDM data is complete |
| USDM encounter mapping | P2_INFUSION/TREATMENT/DISCHARGE reference P1 encounters (data issue) | Fix in source USDM JSON; update SoA accordingly |
| USDM import | Manually parsed for POC | Build generic importer against USDM v4.0 API/DDF spec |
| ODM 2.0 migration | ALTER columns shown | Full P21 migration with ValueListDef/WhereClauseDef tables |


> **SAS mapping detail:** See `CRF_SAS_MAPPING.md` for the full SAS transformation logic, step-by-step dataset descriptions, and output sheet specifications.

---

## SDTM Trial Summary Domain Generation

### Python generation approach

The five SDTM Trial Design domains are generated directly from the USDM JSON using the SDTM IG 3.4 mapping specification. No SAS required. Output: `LY900018_SDTM_Trial_Domains.xlsx`.

### TA — Trial Arms

The crossover element sequence per arm is derived from StudyCells and explicitly mapped:
```python
arm_epoch_elem = {
    'LY-G': ['StudyElement_1','StudyElement_2','StudyElement_4','StudyElement_3','StudyElement_5'],
    'G-LY': ['StudyElement_1','StudyElement_3','StudyElement_4','StudyElement_2','StudyElement_5'],
}
for arm in sd['arms']:
    for i, (epoch, elem_id) in enumerate(zip(epochs_ord, arm_epoch_elem[arm_name]), 1):
        # TAETORD = i, ETCD = elem.name, EPOCH = epoch.label
```

### TV — Trial Visits

VISITNUM comes from SAI position in the linked-list sequence. VISITDY from `scheduleTimelines[0].timings`, matched by label to SAI name:
```python
day_map = {}
for t in tl['timings']:
    if t['type']['decode'] == 'Fixed Reference': day_map[t['label']] = 1
    elif t['type']['decode'] == 'Before': day_map[t['label']] = 1 - days
    else: day_map[t['label']] = 1 + days
```
Both arms get the same visit rows since the schedule is shared (ARMCD repeated per arm).

### TI — Eligibility Criteria

`eligibilityCriteria` (on studyDesign) has no `nextId` chain — sorted by integer `identifier` field instead. Text comes from `eligibilityCriterionItem.text` (rich HTML) via `criterionItemId` FK, with HTML stripped:
```python
ecs_sorted = sorted(sd['eligibilityCriteria'], key=lambda ec: int(ec['identifier']))
for ec in ecs_sorted:
    item = ei_map[ec['criterionItemId']]
    text = re.sub(r'<[^>]+>', ' ', item['text']).strip()
```

### TS — TSSEQ per TSPARMCD (SDTM IG 3.4 §5.5)

TSSEQ is **not** a global sequence. It resets to 1 for each new TSPARMCD:
```python
parm_seq = defaultdict(int)   # counter per TSPARMCD

def add_ts(parmcd, parm, val, ...):
    parm_seq[parmcd] += 1     # resets naturally: first call = 1, second = 2
    rows.append([STUDYID, 'TS', parm_seq[parmcd], grpid, parmcd, ...])
```
Parameters with multiple values in LY900018: TRT (×2 — LY900018 + GlucaGen), OBJSEC (×3), INDIC (×2).

**TSGRPID** is populated for TRT records to group related treatment parameters — each intervention gets a TSGRPID (1 for LY900018, 2 for GlucaGen). This would also link DOSE, DOSU, DOSFRQ records if those were available in the USDM.

### TSVALCD / TSVCDREF

CDISC CT codes populated where known:
- REGID TSVALCD: C54693 (ClinicalTrials.gov); TSVCDREF: CDISC CT
- TPHASE TSVALCD: C15600 (Phase III); TSVCDREF: CDISC CT
- STYPE TSVALCD: C98388 (Interventional Study)
- RANDOM TSVALCD: C25301 (Yes)

### Known gaps from USDM JSON

- StudyCells in this JSON have no `elementId` populated — element-to-arm mapping was manually derived from arm names and element descriptions
- `StudyArm.label` is empty — `name` used as ARMCD
- Dose/route/frequency details (DOSE, DOSU, DOSFRQ, ROUTE) not available in this USDM instance; TSGRPID reserved but unpopulated for those parameters
