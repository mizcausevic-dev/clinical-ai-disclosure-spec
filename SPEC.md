# Clinical AI Disclosure v0.1 — Specification

**Status:** Draft
**Version:** 0.1.0
**Editor:** Miz Causevic
**License:** AGPL-3.0 (this document, schema, and examples). Implementations are unrestricted.

RFC 2119 keywords (**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**) apply throughout.

---

## 1. Scope

This specification defines a JSON document format for **Clinical AI Cards** — machine-readable disclosures published by vendors of AI systems used in healthcare settings: clinical decision support (CDS), diagnostic AI, patient-facing triage, molecule design, scribing, prior-auth automation, and other regulated medical applications.

A Clinical AI Card is consumed by:

- **Hospital procurement teams and Chief Medical Information Officers (CMIOs)** evaluating AI vendors before contracting
- **Compliance officers** auditing HIPAA, 21st Century Cures Act, and state privacy compliance
- **The FDA, EMA, and notified bodies** reviewing software-as-medical-device (SaMD) postings
- **Insurance and risk teams** pricing professional liability when AI is in the care pathway
- **Patients and clinicians** wanting to know what an AI tool will and will not do
- **The agents themselves** (via the [Kinetic Gain Protocol Suite](https://github.com/mizcausevic-dev/kinetic-gain-protocol-suite))

A Clinical AI Card is **the healthcare-vertical sibling** of [AI Tutor Cards](https://github.com/mizcausevic-dev/ai-tutor-card-spec). Both inherit the disclosure philosophy of [Agent Cards](https://github.com/mizcausevic-dev/agent-cards-spec); both add the *regulatory* fields the general-purpose spec deliberately omits. A Clinical AI Card **SHOULD** reference its underlying Agent Card via `agent_card_uri` so a reviewer can chain through to general capabilities.

## 2. Terminology

- **Clinical AI system** — any AI software product used in patient care, research, or operational healthcare contexts.
- **Care setting** — where the system is deployed: inpatient / outpatient / ICU / ED / home / telehealth / research.
- **Decision support level** — how much agency the AI exercises: `informational` (provides data, no recommendation), `advisory` (recommends, clinician decides), or `autonomous` (acts on its own). Autonomy in clinical settings has FDA implications — see §6.3.
- **SaMD** — Software as Medical Device, per IMDRF and FDA definitions. Class I-IV based on healthcare-decision impact + healthcare-situation severity.
- **PHI** — Protected Health Information under HIPAA.
- **BAA** — Business Associate Agreement, the HIPAA contract required when a vendor processes PHI on behalf of a covered entity.
- **CDS Hooks** — HL7 standard for triggering decision-support callouts at clinically-relevant moments (medication order, problem add, encounter open).
- **SMART on FHIR** — the OAuth2 / OpenID Connect profile + FHIR data model that lets third-party apps run inside EHRs (Epic, Cerner, etc.).
- **De-identification (Safe Harbor / Expert Determination)** — the two HIPAA-acceptable methods of stripping PHI from a dataset.

## 3. Design philosophy

Three principles drive the design:

### 3.1 Regulatory disclosure as machine-readable evidence

Hospital procurement today reads vendor whitepapers, HIPAA attestations, and (when applicable) FDA 510(k) summaries as PDFs. **A Clinical AI Card is the same content in a format an LMS, an EHR app gallery, or an automated procurement tool can read in milliseconds.** The fields map directly to controls in CMS Conditions of Participation, 21st Century Cures Act information-blocking provisions, and HIPAA security-rule audits.

### 3.2 Autonomy ↔ device-status coupling

The single most important conditional rule in this spec: **a system that acts autonomously in a clinical role is, by FDA definition, a medical device.** §6.3 enforces this via schema (`decision_support_level=autonomous` ⇒ `is_medical_device=true`). Vendors who try to ship "autonomous AI" without device clearance are publicly identified by their own Card.

### 3.3 Bias audits as a first-class field

For high-stakes systems (sepsis detection, sepsis bundle compliance, fall-risk prediction, ICU readmission), bias-by-protected-class drives most regulatory action — both FDA's "AI/ML-Enabled Medical Devices" guidance and state laws (NYC LL 144 for hiring AI, Colorado SB 21-169 for insurance AI). The spec carries `evidence.bias_audit_uri` as a top-level field rather than a footnote.

## 4. Document structure

### 4.1 `clinical_ai_card_version` (required)

A semver string. **MUST** be `"0.1"` for documents conforming to this draft.

### 4.2 `system` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string (kebab-case) | yes | Stable, unique within the vendor. |
| `name` | string | yes | Display name. |
| `version` | string (semver) | yes | |
| `provider` | string | yes | Vendor name. |
| `homepage` | URI | no | |
| `description` | string | yes | One-paragraph description. |

### 4.3 `clinical_context` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `indication` | string | yes | What conditions / diagnoses / use cases the system addresses, in plain clinical language. |
| `care_settings` | array of enum | yes | One or more of `inpatient`, `outpatient`, `icu`, `ed`, `home`, `telehealth`, `research`, `pharmacy`, `radiology`, `pathology`. |
| `patient_population` | object | yes | `{ age_range_min, age_range_max, exclusions[] }`. `exclusions[]` is free-text — common values: `"pregnancy"`, `"pediatrics-under-2"`, `"immunocompromised"`. |
| `intended_use` | string | yes | The FDA-standard "intended use" statement. Verbatim from the 510(k) when applicable. |
| `off_label_uses_prohibited` | boolean | yes | Whether the vendor explicitly prohibits off-label use. SHOULD be `true` for SaMD class II and above. |

### 4.4 `regulatory` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `fda_status` | enum | yes | `510k_cleared` / `de_novo` / `pma` / `enforcement_discretion` / `research_use_only` / `not_applicable`. `not_applicable` means the vendor's position is "this isn't a device" — vendors using this value should explain in `notes`. |
| `fda_clearance_number` | string | conditional | Required when `fda_status` is `510k_cleared`, `de_novo`, or `pma`. E.g. `"K233456"`. |
| `fda_clearance_uri` | URI | conditional | Required alongside `fda_clearance_number`. Points at the FDA's public 510(k) summary or De Novo classification. |
| `iso_certifications` | array of string | no | E.g. `["ISO 13485", "IEC 62304", "ISO 14971"]`. |
| `is_medical_device` | boolean | yes | The vendor's position on whether this is a device. |
| `is_clinical_decision_support` | boolean | yes | Whether the system meets the FDA / 21st Century Cures Act definition of CDS. |
| `is_software_as_medical_device` | boolean | yes | SaMD. |
| `samd_class` | enum | conditional | `I` / `II` / `III` / `IV` per IMDRF. Required when `is_software_as_medical_device` is `true`. |
| `samd_classification_rationale` | string | conditional | Required alongside `samd_class`. The healthcare-decision × healthcare-situation matrix cell that supports the class. |
| `regional_authorizations` | array of object | no | `{ region: "EU"|"UK"|"CA"|"AU"|..., status: string, identifier: string?, uri: URI? }`. |
| `notes` | string | no | |

### 4.5 `clinical_role` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `decision_support_level` | enum | yes | `informational` / `advisory` / `autonomous`. |
| `clinician_override_required` | boolean | yes | Whether a clinician MUST be able to override every AI output before it affects patient care. SHOULD be `true` for `advisory` and `autonomous` levels. |
| `patient_facing_only` | boolean | yes | Whether the system is consumer / patient-facing without a clinician in the loop. |
| `transparency_to_patient_required` | boolean | yes | Whether patients must be informed an AI was involved in their care decision. Mirrors EU AI Act Art. 50 and several state laws. |
| `pre_authorization_use` | boolean | no | Whether this system makes prior-authorization or coverage-determination decisions. Triggers extra requirements under recent CMS rules and several state laws (e.g. California SB 1120). |

### 4.6 `evidence` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `validation_studies` | array of object | yes | At least one entry. Each: `{ title, uri, population_size, primary_outcome, results_summary, peer_reviewed: bool, published_at?: ISO 8601 }`. |
| `training_data_sources` | array of string | yes | Categories of training data, e.g. `["MIMIC-IV", "internal-dataset-N=240k-encounters", "synthetic-eICU"]`. |
| `bias_audit_uri` | URI | conditional | **Required** when any of: `samd_class` is `II`, `III`, or `IV`; `decision_support_level` is `autonomous`; `pre_authorization_use` is `true`. |
| `performance_metrics` | object | yes | At least `measurement_population` plus one of `sensitivity`, `specificity`, `auc`, `accuracy`, `false_positive_rate`, `false_negative_rate`, `precision`, `recall`, `f1`. Each metric is a float 0–1. |
| `performance_metrics.measurement_population` | string | yes | Describes the population the metrics were measured on — geographic, demographic, encounter type. |

### 4.7 `patient_data` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `phi_processed` | boolean | yes | Whether the system processes Protected Health Information. |
| `hipaa_compliant` | boolean | conditional | Required when `phi_processed` is `true`. |
| `baa_required` | boolean | conditional | Required when `phi_processed` is `true`. Whether the vendor will sign a Business Associate Agreement. |
| `de_identification_method` | enum | no | `safe-harbor` / `expert-determination` / `none` / `not-applicable`. |
| `retention_days` | integer | yes | How long patient data is retained. `0` means not retained beyond the immediate transaction. |
| `patient_consent_required` | boolean | yes | Whether explicit patient consent gates use. |
| `consent_flow_uri` | URI | no | Documentation of the consent flow. |
| `third_party_data_sharing` | boolean | yes | Whether patient data is shared with any party other than the vendor and covered entity. |
| `model_training_consent_required` | boolean | no | Whether the vendor uses patient data for model training, and whether explicit consent is required if so. |

### 4.8 `safety` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `human_in_loop_required_for` | array of string | yes | Contexts that escalate to a human regardless of system confidence (e.g. `pediatric-under-2`, `pregnancy`, `mental-health-crisis`, `oncologic-staging`). May be empty `[]`. |
| `escalation_protocols` | array of string | no | Free-text protocols. |
| `mandatory_reporting_categories` | array of string | yes | Reportable conditions the system surfaces or escalates: `adverse-drug-event`, `child-abuse`, `elder-abuse`, `suspected-suicide-risk`, `infectious-disease-notifiable`, etc. May be empty `[]`. |
| `blocks_diagnostic_claims` | boolean | no | For non-FDA-cleared systems, whether the system blocks itself from making definitive diagnostic statements. SHOULD be `true` when `fda_status` is `research_use_only`, `enforcement_discretion`, or `not_applicable` and `decision_support_level` is `informational`. |
| `treatment_recommendation_disclaimer_required` | boolean | no | Whether outputs carry a "not a substitute for medical advice" disclaimer. SHOULD be `true` for `patient_facing_only` systems. |

### 4.9 `ehr_integration` (optional)

| Field | Type | Required | Description |
|---|---|---|---|
| `fhir_version` | enum | no | `R4` / `R5` / `STU3` / `DSTU2`. |
| `supports_smart_on_fhir` | boolean | no | |
| `supports_cds_hooks` | boolean | no | |
| `ehr_vendors_supported` | array of string | no | `Epic` / `Cerner` / `Athenahealth` / `MEDITECH` / `Allscripts` / `eClinicalWorks` / etc. |

### 4.10 `agent_card_uri` (optional but recommended)

URI of the underlying [Agent Card](https://github.com/mizcausevic-dev/agent-cards-spec). When present, a reviewer can chain through to inspect general capability surface, refusal taxonomy, and deployment posture.

### 4.11 `evaluations` (optional)

Independent evaluations beyond the vendor's own `validation_studies`. Each entry: `{ suite, result_uri, metrics: { … }, ran_at, accreditation_body? }`. Use this for third-party benchmarks (e.g. STAT-AI, NIST AI Test, academic comparative studies).

### 4.12 `audit` (optional)

| Field | Type | Required | Description |
|---|---|---|---|
| `audit_log_uri` | URI | no | Where customers' audit logs are accessible. |
| `incident_response_uri` | URI | no | Vendor's published clinical-AI incident response plan. |
| `incident_card_index_uri` | URI | no | Index of published [AI Incident Cards](https://github.com/mizcausevic-dev/ai-incident-card-spec) affecting this system. |
| `disclosure_uri` | URI | no | Customer-disclosure / changelog location. |

## 5. Conditional rules

### 5.1 Autonomy ⇔ medical device

If `clinical_role.decision_support_level` is `autonomous`, then `regulatory.is_medical_device` **MUST** be `true`. Autonomous AI in a clinical context that is not classified as a device represents a misclassification under current FDA guidance.

### 5.2 SaMD class completeness

If `regulatory.is_software_as_medical_device` is `true`, then both `regulatory.samd_class` and `regulatory.samd_classification_rationale` **MUST** be present.

### 5.3 FDA clearance documentation

If `regulatory.fda_status` is one of `510k_cleared`, `de_novo`, or `pma`, then both `regulatory.fda_clearance_number` and `regulatory.fda_clearance_uri` **MUST** be present.

### 5.4 PHI gating

If `patient_data.phi_processed` is `true`, then `patient_data.hipaa_compliant` and `patient_data.baa_required` **MUST** be present (the vendor's posture on each must be explicit).

### 5.5 Bias audit required

If any of the following hold, then `evidence.bias_audit_uri` **MUST** be present:

- `regulatory.samd_class` is `II`, `III`, or `IV`
- `clinical_role.decision_support_level` is `autonomous`
- `clinical_role.pre_authorization_use` is `true`

### 5.6 Patient-facing safeguards

If `clinical_role.patient_facing_only` is `true` and `regulatory.fda_status` is not `510k_cleared` / `de_novo` / `pma`, then `safety.blocks_diagnostic_claims` **SHOULD** be `true` and `safety.treatment_recommendation_disclaimer_required` **SHOULD** be `true`. Consumers MAY downgrade trust when these are `false`.

### 5.7 Off-label gating for SaMD class II+

If `regulatory.samd_class` is `II`, `III`, or `IV`, then `clinical_context.off_label_uses_prohibited` **SHOULD** be `true`. Off-label clinical AI use is a fast-growing FDA enforcement area.

## 6. Cross-spec composition

A Clinical AI Card composes cleanly with the rest of the Kinetic Gain Protocol Suite:

```
Clinical AI Card
   ├── agent_card_uri ──────────────→ Agent Card        (capability + refusal disclosure)
   ├── evidence.validation_studies → AI Evidence        (per-study citation provenance)
   ├── evaluations[].result_uri ──→ Prompt Provenance  (when LLM-based)
   ├── audit.incident_card_index_uri → AI Incident Card (mandatory-reporting categories)
   └── safety.mandatory_reporting_categories ↔ AI Incident Card categories
```

A hospital CMIO inspecting a clinical AI vendor can pull a single Clinical AI Card and chain through to every related disclosure document the vendor publishes — no PDF reading, no procurement-team turnaround time.

## 7. Well-known location

A vendor publishing Clinical AI Cards **SHOULD** serve each card at:

```
https://<vendor-domain>/.well-known/clinical-ai/<system_id>.json
```

A vendor with multiple systems **MAY** also publish an index at:

```
https://<vendor-domain>/.well-known/clinical-ai.json
```

The index is a JSON array of `{ id, name, version, fda_status, system_uri }` entries.

## 8. Distribution and discoverability

- Hospital EHR app galleries (Epic Showroom, Cerner Code) **SHOULD** ingest Clinical AI Cards at integration time and surface the disclosed fields to the CMIO before installation.
- Third-party AI marketplaces (HealtheBay-style) **MAY** require a published Clinical AI Card as a precondition for listing.
- Patients **MAY** be directed to the Card via consent-flow links.

## 9. Security and policy considerations

- **PHI is never disclosed in the Card itself.** The Card describes *that* PHI is processed, not what specific PHI; vendors **MUST NOT** include patient records or de-identified samples in any Card field.
- **Audit log integrity.** The `audit.audit_log_uri` is the vendor's commitment — implementations are expected to preserve immutable, time-stamped logs.
- **Signing.** v0.1 does not require cryptographic signatures. v0.2 may add an optional detached signature similar to the AEO Protocol audit block, for procurement teams needing tamper-evidence.
- **Withdrawal.** A vendor who pulls a product **SHOULD** mark the Card with `regulatory.notes` describing the withdrawal rather than deleting the URL.

## 10. Open questions for v0.2

- **Continuous learning systems.** v0.1 treats the system as effectively static between releases. v0.2 will likely add a `continuous_learning` block describing retraining cadence, retraining-trigger conditions, and a guarantee of model-version-pinning for studies-in-progress.
- **Multi-tenant clinical use.** v0.2 may add multi-tenant disclosure fields when a single AI instance serves many covered entities.
- **State-law mapping.** v0.2 may add a `state_law_compliance[]` block enumerating compliance posture per state (TX HB 4, CA SB 1120, NY S3008, IL HB 4869, etc.).
- **EU AI Act mapping.** v0.2 may add a structured `eu_ai_act` block once high-risk-AI obligations under Article 9-15 fully apply (August 2026).

## 11. Versioning

The `clinical_ai_card_version` field identifies the spec revision. v0.1 is a draft; consumers **SHOULD** treat unknown future versions as an error rather than attempting forward-compatible parsing.
