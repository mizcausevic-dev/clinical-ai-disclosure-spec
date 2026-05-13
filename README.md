# Clinical AI Disclosure

> **Open JSON spec for vendor disclosure of AI systems used in healthcare.**
> Machine-readable HIPAA / FDA / SaMD posture. Bias audits as a first-class field. v0.1 draft.

A draft specification for **Clinical AI Cards** — declarative documents that disclose what a healthcare AI system is for, what regulatory class it falls under, what evidence supports it, what patient-data posture it carries, and how it integrates with EHRs.

This spec is the **healthcare-vertical sibling** of [AI Tutor Cards](https://github.com/mizcausevic-dev/ai-tutor-card-spec). Both inherit the disclosure philosophy of [Agent Cards](https://github.com/mizcausevic-dev/agent-cards-spec); both add the regulatory fields the general-purpose spec deliberately omits — Tutor Cards add FERPA/COPPA/audience/pedagogy; Clinical AI Cards add HIPAA/FDA/SaMD/clinical-role.

## The eight required sections

| Pillar | What it captures |
|---|---|
| **System** | Vendor identity (id, name, version, provider, description) |
| **Clinical context** | Indication, care setting, patient population, intended use, off-label prohibitions |
| **Regulatory** | FDA status (510k / De Novo / PMA / enforcement discretion / research-use-only), clearance number + URI, ISO certifications, SaMD class + classification rationale, regional authorizations |
| **Clinical role** | Decision-support level (informational / advisory / autonomous), clinician override, patient-facing, transparency-to-patient, pre-authorization use |
| **Evidence** | Validation studies (population size, primary outcome, peer-reviewed), training data sources, bias audit URI, performance metrics with measurement population |
| **Patient data** | PHI processed, HIPAA, BAA, de-identification method, retention days, patient consent, third-party sharing, model-training consent |
| **Safety** | Human-in-loop categories, escalation protocols, mandatory reporting categories, diagnostic-claim blocking, treatment disclaimer |
| **EHR integration** *(optional)* | FHIR version, SMART on FHIR, CDS Hooks, supported EHR vendors |

## The conditional rules (the procurement spine)

These are the rules schema-enforced via `allOf/if/then` — most carry direct regulatory weight:

1. **Autonomy ⇔ medical device** — `decision_support_level=autonomous` ⇒ `is_medical_device=true`. The FDA's position: autonomous clinical diagnosis IS a device, end of story.
2. **SaMD completeness** — `is_software_as_medical_device=true` ⇒ `samd_class` + `samd_classification_rationale` required.
3. **FDA clearance documentation** — `fda_status ∈ {510k_cleared, de_novo, pma}` ⇒ clearance number + URI required.
4. **PHI gating** — `phi_processed=true` ⇒ HIPAA posture + BAA posture must be explicit.
5. **Bias audit required (3 triggers)** — `samd_class ∈ {II, III, IV}` OR `decision_support_level=autonomous` OR `pre_authorization_use=true` ⇒ `bias_audit_uri` required.

Beyond schema rules, the spec specifies SHOULDs for: patient-facing safeguards (block diagnostic claims, require treatment disclaimer), off-label prohibitions for SaMD II+, and transparency-to-patient when an AI is in the care pathway.

## Quickstart

1. Pick a `system.id` (kebab-case, unique within your vendor).
2. Author a Clinical AI Card conforming to [`clinical-ai-card.schema.json`](clinical-ai-card.schema.json). Start from one of the [examples](examples/).
3. Validate with any JSON Schema 2020-12 validator:
   ```bash
   npx -p ajv-cli -p ajv-formats ajv validate \
     -s clinical-ai-card.schema.json \
     -d examples/sepsis-early-warning.json \
     -c ajv-formats --spec=draft2020 --strict=false
   ```
4. Serve at `https://<your-vendor>/.well-known/clinical-ai/<system-id>.json` with `Content-Type: application/json`.
5. Optionally publish an index at `https://<your-vendor>/.well-known/clinical-ai.json` listing all your systems.

## Cross-spec composition

```
Clinical AI Card
   ├── agent_card_uri ────────────→ Agent Card        (capability + refusal disclosure)
   ├── evidence.validation_studies → AI Evidence       (per-study citation provenance)
   ├── evaluations[].result_uri ──→ Prompt Provenance  (when LLM-based)
   ├── audit.incident_card_index_uri → AI Incident Card (mandatory-reporting categories)
   └── safety.mandatory_reporting_categories ↔ AI Incident Card categories
```

A hospital CMIO can pull a single Clinical AI Card and chain through to every related disclosure document in one document-graph walk. No procurement-team PDF marathon.

## Files in this repo

- [`SPEC.md`](SPEC.md) — full v0.1 specification (11 sections)
- [`clinical-ai-card.schema.json`](clinical-ai-card.schema.json) — JSON Schema (draft 2020-12) with 7 conditional rules
- [`examples/`](examples/) — reference Clinical AI Cards:
  - [`sepsis-early-warning.json`](examples/sepsis-early-warning.json) — 510(k)-cleared SaMD class II advisory CDS, EHR-integrated, clinician-override-required, multi-site validated, bias-audited
  - [`symptom-triage-chatbot.json`](examples/symptom-triage-chatbot.json) — patient-facing, informational only, 988-handoff for suicide risk, blocks diagnostic claims, enforcement-discretion FDA status
  - [`research-molecule-design.json`](examples/research-molecule-design.json) — research-use-only generative chemistry, no PHI, IRB-gated for human-subjects research

## Status

**v0.1 draft.** Issues and pull requests welcome.

## License

MIT-licensed. The specification text, JSON Schema, and example documents in this repository may be freely implemented, extended, redistributed, or incorporated into commercial or non-commercial products with attribution. Reference implementations of this spec (such as [mcp-kinetic-gain](https://github.com/mizcausevic-dev/mcp-kinetic-gain)) are licensed separately under AGPL-3.0.

## Kinetic Gain Protocol Suite

A family of ten open specifications for the answer-engine and agent era, now with three regulated-vertical extensions:

| Spec | What it does |
|---|---|
| [AEO Protocol](https://github.com/mizcausevic-dev/aeo-protocol-spec) | Entity declaration at `/.well-known/aeo.json` |
| [Prompt Provenance](https://github.com/mizcausevic-dev/prompt-provenance-spec) | Versioned, lineaged, reviewable LLM prompt records |
| [Agent Cards](https://github.com/mizcausevic-dev/agent-cards-spec) | Declarative agent capability + refusal disclosure |
| [AI Evidence Format](https://github.com/mizcausevic-dev/ai-evidence-format-spec) | Structured citations for LLM-generated claims |
| [MCP Tool Cards](https://github.com/mizcausevic-dev/mcp-tool-card-spec) | Per-tool disclosure for Model Context Protocol servers |
| [AI Tutor Cards](https://github.com/mizcausevic-dev/ai-tutor-card-spec) | **EdTech vertical** — vendor-side disclosure |
| [Student AI Disclosure](https://github.com/mizcausevic-dev/student-ai-disclosure-spec) | **EdTech vertical** — student-side disclosure |
| [Classroom AI AUP](https://github.com/mizcausevic-dev/classroom-ai-aup-spec) | **EdTech vertical** — district/school/course AI policy |
| [AI Incident Card](https://github.com/mizcausevic-dev/ai-incident-card-spec) | Post-incident disclosure ("CVE for AI agents") |
| **Clinical AI Disclosure** (this) | **HealthTech vertical** — vendor-side healthcare disclosure |

Single landing: [`kinetic-gain-protocol-suite`](https://github.com/mizcausevic-dev/kinetic-gain-protocol-suite).

---

**Connect:** [LinkedIn](https://www.linkedin.com/in/mirzacausevic/) · [Kinetic Gain](https://kineticgain.com) · [Medium](https://medium.com/@mizcausevic/) · [Skills](https://mizcausevic.com/skills/)
