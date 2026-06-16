---
name: gemara-artifact-authoring
description: Use when creating, iterating on, or validating Gemara security artifacts (threat catalogs, control catalogs). Also use when the user has security documentation, threat models, or attack scenarios that could be structured as Layer 2 Gemara artifacts. For policies, risk catalogs, or mapping documents, use the dedicated gemara-policy, gemara-risk-catalog, or gemara-mapping-document skills instead.
---

# Gemara Artifact Authoring

Guide users through creating schema-valid Gemara Layer 2 security artifacts. Triage what to build, check prerequisites, then walk through the appropriate wizard.

## Tools and Resources

| Tool / Resource | Purpose |
|----------------|---------|
| `validate_gemara_artifact` | Validate YAML against a Gemara CUE schema definition |
| `migrate_gemara_artifact` | Migrate v0 artifacts to v1 schema |
| `gemara://lexicon` | Term definitions for the Gemara security model |
| `gemara://schema/definitions` | CUE schema definitions for all artifact types |

## Triage

Perform these three steps before starting any wizard.

### Step 1: Scan the repo

Search for `.yaml` and `.yml` files containing a `metadata.type` field with a recognized Gemara artifact type. Build an inventory table:

| Layer | Artifact Type | Found | File |
|-------|--------------|-------|------|
| 1 | GuidanceCatalog | | |
| 2 | ThreatCatalog | | |
| 2 | ControlCatalog | | |
| 2 | CapabilityCatalog | | |
| 3 | RiskCatalog | | |
| 3 | Policy | | |
| 3 | MappingDocument | | |

### Step 2: Analyze user-provided content

If the user provides raw documentation, pasted YAML, or file references:

- **Structured YAML with `metadata.type`** — call `validate_gemara_artifact` to confirm type
- **Unstructured content** — identify the best artifact fit:
  - Threats, attack scenarios, adversary techniques → ThreatCatalog
  - Security controls, safeguards, countermeasures → ControlCatalog
  - Compliance requirements, standards, regulations → GuidanceCatalog
  - Risk registers, risk assessments, impact analyses → use the `gemara-risk-catalog` skill
  - Organizational policies, scope definitions, enforcement rules → use the `gemara-policy` skill
  - Cross-references between two artifact sets → use the `gemara-mapping-document` skill

### Step 3: Recommend based on layer dependencies

```
Layer 1: GuidanceCatalog
    ↓ (referenced by ControlCatalog guidelines)
Layer 2: ThreatCatalog → ControlCatalog ← CapabilityCatalog
    ↓ (feeds into)           ↓ (imported by)
Layer 3: RiskCatalog → Policy ← MappingDocument
```

Recommendation priority:
1. No Layer 2 artifacts → recommend ThreatCatalog
2. ThreatCatalog exists, no ControlCatalog → recommend ControlCatalog
3. Both Layer 2 exist, no Layer 3 → recommend the `gemara-risk-catalog` or `gemara-policy` skill based on user context
4. Two or more artifacts exist → recommend the `gemara-mapping-document` skill

Present findings and recommendation, then ask which artifact to create. If the user chooses a Layer 3 artifact or MappingDocument, direct them to the corresponding standalone skill.

## Prerequisite Check

Before starting a wizard, check prerequisites. If missing, warn the user and offer three options:

| Artifact Type | Prerequisites |
|--------------|---------------|
| ThreatCatalog | None |
| ControlCatalog | ThreatCatalog (for threat mappings) |

If prerequisites are missing:

> "A [artifact type] typically requires [prerequisites]. These were not found in this repo. You can proceed if you have external references (e.g., NIST 800-53), or we can build the prerequisites first. What would you like to do?"
>
> 1. Build the prerequisite first
> 2. Proceed with external references
> 3. Choose a different artifact type

## Wizard Routing

After the user confirms the artifact type, load and follow the corresponding wizard file:

| Artifact Type | Wizard File | CUE Definition |
|--------------|-------------|----------------|
| ThreatCatalog | threat-assessment-wizard.md | `#ThreatCatalog`, `#CapabilityCatalog` |
| ControlCatalog | control-catalog-wizard.md | `#ControlCatalog` |

For other artifact types, direct the user to the standalone skill:

| Artifact Type | Skill |
|--------------|-------|
| MappingDocument | `gemara-mapping-document` |
| Policy | `gemara-policy` |
| RiskCatalog | `gemara-risk-catalog` |

Follow the wizard step-by-step. Do not skip steps or present the artifact as complete until it passes final validation with `validate_gemara_artifact`.

## Validation Rules

- **Validate as you go** — call `validate_gemara_artifact` after each major section (metadata, groups, entries, mappings)
- **Final validation is mandatory** — do not present "Next Steps" until the full artifact passes validation
- **Auto-correct loop** — if validation fails, diagnose the error, propose corrected YAML, re-validate, loop until valid
- **Version pinning** — use the correct `gemara-version` per wizard (see each wizard file)
- **Local verification** — after validation passes, provide the `cue vet` command for independent verification
