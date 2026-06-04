# Gemara Artifact Authoring Plugin — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the OpenPackage distribution with a Claude Code plugin that bundles the gemara-mcp server and a single artifact authoring skill with five wizard reference files.

**Architecture:** A Claude Code plugin (`.claude-plugin/plugin.json` + `.mcp.json`) provides the MCP server. A single skill (`skills/gemara-artifact-authoring/SKILL.md`) handles triage — scanning the repo for existing artifacts, analyzing user content, and recommending artifact types via a layer dependency graph. Once an artifact type is selected, the skill loads one of five wizard reference files that contain the full step-by-step authoring flow. Every wizard enforces schema validity via `validate_gemara_artifact` at multiple checkpoints.

**Tech Stack:** Claude Code plugin system, gemara-mcp Docker image (v0.4.0), MCP tools (`validate_gemara_artifact`, `migrate_gemara_artifact`), MCP resources (`gemara://lexicon`, `gemara://schema/definitions`), Gemara CUE schemas.

**Spec:** `docs/superpowers/specs/2026-06-05-gemara-artifact-authoring-skill-design.md`

---

## File Map

| Action | Path | Responsibility |
|--------|------|---------------|
| Create | `.claude-plugin/plugin.json` | Plugin identity and version |
| Create | `.mcp.json` | MCP server config (artifact + advisory modes) |
| Create | `skills/gemara-artifact-authoring/SKILL.md` | Triage logic, prerequisite checking, wizard routing |
| Create | `skills/gemara-artifact-authoring/threat-assessment-wizard.md` | ThreatCatalog + CapabilityCatalog authoring flow |
| Create | `skills/gemara-artifact-authoring/control-catalog-wizard.md` | ControlCatalog authoring flow |
| Create | `skills/gemara-artifact-authoring/mapping-document-wizard.md` | MappingDocument authoring flow |
| Create | `skills/gemara-artifact-authoring/policy-wizard.md` | Policy authoring flow |
| Create | `skills/gemara-artifact-authoring/risk-catalog-wizard.md` | RiskCatalog authoring flow |
| Rewrite | `README.md` | Plugin installation instructions |
| Delete | `openpackage.yml` | Replaced by plugin.json |
| Delete | `gemara-mcp/` (entire directory) | Replaced by .mcp.json |

---

### Task 1: Remove OpenPackage files and create plugin scaffold

**Files:**
- Delete: `openpackage.yml`
- Delete: `gemara-mcp/openpackage.yml`
- Delete: `gemara-mcp/root/openpackage.yml`
- Delete: `gemara-mcp/root/mcp.jsonc`
- Delete: `gemara-mcp/root/README.md`
- Create: `.claude-plugin/plugin.json`
- Create: `.mcp.json`

- [ ] **Step 1: Delete the OpenPackage directory and top-level manifest**

```bash
rm -rf gemara-mcp/
rm openpackage.yml
```

- [ ] **Step 2: Create `.claude-plugin/plugin.json`**

```bash
mkdir -p .claude-plugin
```

Write `.claude-plugin/plugin.json`:

```json
{
  "name": "gemara",
  "description": "Gemara security artifact authoring and validation",
  "version": "0.1.0"
}
```

- [ ] **Step 3: Create `.mcp.json`**

Write `.mcp.json`:

```json
{
  "mcpServers": {
    "gemara-mcp": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "ghcr.io/gemaraproj/gemara-mcp@sha256:0d05c93d237c08483a2b046cff16b1765c42f3cfcba152b02b0904da7d8a05f0",
        "serve"
      ]
    },
    "gemara-mcp-advisory": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "ghcr.io/gemaraproj/gemara-mcp@sha256:0d05c93d237c08483a2b046cff16b1765c42f3cfcba152b02b0904da7d8a05f0",
        "serve", "--mode", "advisory"
      ]
    }
  }
}
```

- [ ] **Step 4: Verify plugin structure**

```bash
find . -not -path './.git/*' -not -path './.claude/*' -not -path './docs/*' -type f | sort
```

Expected output should include `.claude-plugin/plugin.json` and `.mcp.json`, and NOT include any `openpackage.yml` or `gemara-mcp/` files.

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin/plugin.json .mcp.json
git rm openpackage.yml gemara-mcp/openpackage.yml gemara-mcp/root/openpackage.yml gemara-mcp/root/mcp.jsonc gemara-mcp/root/README.md
git commit -m "refactor: replace OpenPackage with Claude Code plugin scaffold

Removes gemara-mcp OpenPackage directory and top-level openpackage.yml.
Adds .claude-plugin/plugin.json for plugin identity and .mcp.json for
MCP server configuration (artifact and advisory modes).

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 2: Create SKILL.md — triage skill

**Files:**
- Create: `skills/gemara-artifact-authoring/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/gemara-artifact-authoring
```

- [ ] **Step 2: Write SKILL.md**

Write `skills/gemara-artifact-authoring/SKILL.md`:

````markdown
---
name: gemara-artifact-authoring
description: Use when creating, iterating on, or validating Gemara security artifacts (threat catalogs, control catalogs, risk catalogs, policies, mapping documents). Also use when the user has security documentation, compliance requirements, threat models, or risk registers that could be structured as Gemara artifacts.
---

# Gemara Artifact Authoring

Guide users through creating schema-valid Gemara security artifacts. Triage what to build, check prerequisites, then walk through the appropriate wizard.

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
  - Risk registers, risk assessments, impact analyses → RiskCatalog
  - Organizational policies, scope definitions, enforcement rules → Policy
  - Cross-references between two artifact sets → MappingDocument

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
3. Both Layer 2 exist, no Layer 3 → recommend RiskCatalog or Policy based on user context
4. Two or more artifacts exist → MappingDocument is an option

Present findings and recommendation, then ask which artifact to create.

## Prerequisite Check

Before starting a wizard, check prerequisites. If missing, warn the user and offer three options:

| Artifact Type | Prerequisites |
|--------------|---------------|
| ThreatCatalog | None |
| ControlCatalog | ThreatCatalog (for threat mappings) |
| RiskCatalog | ThreatCatalog (for threat linkages) |
| Policy | ControlCatalog (for imports), optionally RiskCatalog (for risk dispositions) |
| MappingDocument | Two existing artifacts to map between |

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
| MappingDocument | mapping-document-wizard.md | `#MappingDocument` |
| Policy | policy-wizard.md | `#Policy` |
| RiskCatalog | risk-catalog-wizard.md | `#RiskCatalog` |

Follow the wizard step-by-step. Do not skip steps or present the artifact as complete until it passes final validation with `validate_gemara_artifact`.

## Validation Rules

- **Validate as you go** — call `validate_gemara_artifact` after each major section (metadata, groups, entries, mappings)
- **Final validation is mandatory** — do not present "Next Steps" until the full artifact passes validation
- **Auto-correct loop** — if validation fails, diagnose the error, propose corrected YAML, re-validate, loop until valid
- **Version pinning** — use the correct `gemara-version` per wizard (see each wizard file)
- **Local verification** — after validation passes, provide the `cue vet` command for independent verification
````

- [ ] **Step 3: Verify the file exists and frontmatter is correct**

```bash
head -4 skills/gemara-artifact-authoring/SKILL.md
```

Expected:

```
---
name: gemara-artifact-authoring
description: Use when creating, iterating on, or validating Gemara security artifacts (threat catalogs, control catalogs, risk catalogs, policies, mapping documents). Also use when the user has security documentation, compliance requirements, threat models, or risk registers that could be structured as Gemara artifacts.
---
```

- [ ] **Step 4: Commit**

```bash
git add skills/gemara-artifact-authoring/SKILL.md
git commit -m "feat: add gemara-artifact-authoring triage skill

Single entry point for Gemara artifact authoring. Scans the repo for
existing artifacts, analyzes user content, recommends artifact types
via layer dependency graph, checks prerequisites, and routes to the
appropriate wizard reference file.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 3: Create threat-assessment-wizard.md

**Files:**
- Create: `skills/gemara-artifact-authoring/threat-assessment-wizard.md`

This wizard is adapted from the merged MCP prompt at `internal/server/prompts/threat_assessment_system.md` in the gemara-mcp repo. The key changes from the MCP prompt:

1. Remove "Embedded Resources" section (skill doesn't use MCP prompt message framing)
2. Add instruction to read `gemara://lexicon` and `gemara://schema/definitions` resources via MCP at the start
3. Add validate-as-you-go checkpoints after steps 2, 3, and 4 (not just step 5)
4. Replace `${COMPONENT}` and `${ID_PREFIX}` template variables with instructions to ask the user for these values
5. Replace `${GEMARA_VERSION}` with the literal value `"v1.0.0"`
6. Preserve the full wizard flow: catalog import → metadata → capabilities → threats → assemble/validate → next steps
7. Preserve the interaction model (lettered-row tables)
8. Preserve all constraints (ID patterns, artifact type identification, version checking)

- [ ] **Step 1: Write threat-assessment-wizard.md**

Write `skills/gemara-artifact-authoring/threat-assessment-wizard.md`. Adapt the MCP system prompt applying the changes listed above. The file must contain:

- Title: `# Threat Assessment Wizard`
- A preamble stating: the assistant is a threat assessment wizard guiding users through creating a ThreatCatalog (and optionally a CapabilityCatalog) for a user-specified component. Every mapping, reference, and threat entry requires explicit user approval.
- Instruction to read `gemara://lexicon` and `gemara://schema/definitions` resources at the start for terminology and schema awareness.
- Available Tool table referencing `validate_gemara_artifact` with definition `#ThreatCatalog` and `#CapabilityCatalog`.
- The full outline from the MCP prompt with these steps:
  1. Catalog Import (with FINOS CCC Core as default suggestion)
  2. Scope and Metadata (with `gemara-version: "v1.0.0"`)
  3. Identify Capabilities (groups, capability matching, CapabilityCatalog generation)
  4. Identify Threats (groups, MITRE ATT&CK opt-in, threat entries with capability/vector linkages)
  5. Assemble and Validate (with `cue vet` command)
  6. Next Steps (recommend ControlCatalog)
- Validate-as-you-go checkpoint instructions after steps 2, 3, and 4: "Call `validate_gemara_artifact` with the artifact YAML assembled so far to catch errors early."
- The Artifact Type Identification procedure (unchanged from MCP prompt)
- The Threat Catalog Constraints section (unchanged from MCP prompt)
- All interaction model instructions (lettered-row tables with accept/select/modify/reject)

Source reference: fetch the MCP prompt from `gh api repos/gemaraproj/gemara-mcp/contents/internal/server/prompts/threat_assessment_system.md --jq '.content' | base64 -d`

- [ ] **Step 2: Verify the file is well-formed**

```bash
wc -l skills/gemara-artifact-authoring/threat-assessment-wizard.md
head -5 skills/gemara-artifact-authoring/threat-assessment-wizard.md
```

The file should be approximately 200-280 lines and start with `# Threat Assessment Wizard`.

- [ ] **Step 3: Commit**

```bash
git add skills/gemara-artifact-authoring/threat-assessment-wizard.md
git commit -m "feat: add threat assessment wizard reference

Adapted from gemara-mcp threat_assessment_system.md prompt. Adds
validate-as-you-go checkpoints, explicit MCP resource instructions,
and replaces template variables with concrete values.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 4: Create control-catalog-wizard.md

**Files:**
- Create: `skills/gemara-artifact-authoring/control-catalog-wizard.md`

Same adaptation pattern as Task 3, applied to the control catalog MCP prompt. Key changes from the MCP prompt:

1. Remove "Embedded Resources" section
2. Add instruction to read `gemara://lexicon` and `gemara://schema/definitions` resources via MCP at the start
3. Add validate-as-you-go checkpoints after steps 2, 3, and 4
4. Replace `${COMPONENT}` and `${ID_PREFIX}` with instructions to ask the user
5. Replace `${GEMARA_VERSION}` with `"v1.0.0"`
6. Preserve the full wizard flow: catalog import → metadata (with guideline framework selection and applicability groups) → control groups → controls (with threat/guideline mappings and assessment requirements) → assemble/validate → next steps
7. Preserve the assessment requirement quality rules (MUST/SHOULD, observable verbs, boundary thresholds)

- [ ] **Step 1: Write control-catalog-wizard.md**

Write `skills/gemara-artifact-authoring/control-catalog-wizard.md`. Adapt the MCP system prompt applying the changes listed above. The file must contain:

- Title: `# Control Catalog Wizard`
- Preamble stating: the assistant guides users through creating a ControlCatalog for a user-specified component. Every mapping, reference, and control objective requires explicit user approval.
- Instruction to read MCP resources at the start.
- Available Tool table referencing `validate_gemara_artifact` with definition `#ControlCatalog`.
- The full outline with steps:
  1. Catalog Import (FINOS CCC Core default)
  2. Scope and Metadata (with guideline framework selection table — NIST CSF, CSA CCM, ISO 27001, NIST 800-53 — and applicability groups, `gemara-version: "v1.0.0"`)
  3. Define Control Groups
  4. Define Controls (ID pattern `C##`, objective, threat mappings, guideline mappings, assessment requirements with ID pattern `C##.TR##`)
  5. Assemble and Validate (with `cue vet` command)
  6. Next Steps (recommend Policy)
- Validate-as-you-go checkpoints after steps 2, 3, and 4.
- Artifact Type Identification procedure.
- Control Catalog Constraints section.
- Assessment requirement format rules: "When [trigger/condition], [subject] MUST [observable, measurable action]." with the good/bad examples from the MCP prompt.

Source reference: `gh api repos/gemaraproj/gemara-mcp/contents/internal/server/prompts/control_catalog_system.md --jq '.content' | base64 -d`

- [ ] **Step 2: Verify the file is well-formed**

```bash
wc -l skills/gemara-artifact-authoring/control-catalog-wizard.md
head -5 skills/gemara-artifact-authoring/control-catalog-wizard.md
```

The file should be approximately 200-260 lines and start with `# Control Catalog Wizard`.

- [ ] **Step 3: Commit**

```bash
git add skills/gemara-artifact-authoring/control-catalog-wizard.md
git commit -m "feat: add control catalog wizard reference

Adapted from gemara-mcp control_catalog_system.md prompt. Adds
validate-as-you-go checkpoints, explicit MCP resource instructions,
and replaces template variables with concrete values.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 5: Create mapping-document-wizard.md

**Files:**
- Create: `skills/gemara-artifact-authoring/mapping-document-wizard.md`

Adapted from PR #66 system prompt (`internal/server/prompts/mapping_document_system.md`). Key changes:

1. Remove "Embedded Resources" section
2. Add instruction to read MCP resources at the start
3. Add validate-as-you-go checkpoints after steps 2 and 3
4. Replace `${COMPONENT}` and `${ID_PREFIX}` with instructions to ask the user
5. Replace `${GEMARA_VERSION}` with `"v1.0.0"`
6. Preserve the experimental schema warning
7. Preserve the full wizard flow: artifact import (with artifact type identification and version check) → metadata → mapping direction → define mappings (with relationship types table) → assemble/validate → next steps

- [ ] **Step 1: Write mapping-document-wizard.md**

Write `skills/gemara-artifact-authoring/mapping-document-wizard.md`. Adapt the PR #66 system prompt applying the changes listed above. The file must contain:

- Title: `# Mapping Document Wizard`
- Preamble with the experimental schema warning note.
- Instruction to read MCP resources at the start.
- Available Tool table with `validate_gemara_artifact` and definition `#MappingDocument`.
- Full outline with steps:
  1. Artifact Import (with artifact type identification procedure, version check procedure, valid artifact types table, non-Gemara artifact support)
  2. Scope and Metadata (`gemara-version: "v1.0.0"`, mapping-references)
  3. Configure Mapping Direction (source-reference, target-reference with entry-type confirmation)
  4. Define Mappings (relationship types table: implements, implemented-by, supports, supported-by, equivalent, subsumes, no-match, relates-to; strength 1-10, confidence levels, rationale; ID pattern `MAP##`)
  5. Assemble and Validate (with `cue vet` command)
  6. Next Steps (recommend Policy referencing this mapping)
- Validate-as-you-go checkpoints after steps 2, 3, and each batch of mappings in step 4.
- Artifact Type Identification procedure.
- Version Check procedure (with migration prompt referral for ThreatCatalog/ControlCatalog).
- Mapping Document Constraints section (ID regex, unique mapping IDs, targets required when not no-match, strength 1-10, confidence levels).

Source reference: the mapping document system prompt content already read from PR #66 diff earlier in this conversation.

- [ ] **Step 2: Verify the file is well-formed**

```bash
wc -l skills/gemara-artifact-authoring/mapping-document-wizard.md
head -5 skills/gemara-artifact-authoring/mapping-document-wizard.md
```

The file should be approximately 220-280 lines and start with `# Mapping Document Wizard`.

- [ ] **Step 3: Commit**

```bash
git add skills/gemara-artifact-authoring/mapping-document-wizard.md
git commit -m "feat: add mapping document wizard reference

Adapted from gemara-mcp PR #66 mapping_document_system.md prompt.
Includes artifact type identification, version checking, relationship
type definitions, and validate-as-you-go checkpoints.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 6: Create policy-wizard.md

**Files:**
- Create: `skills/gemara-artifact-authoring/policy-wizard.md`

Adapted from PR #38 system prompt (`internal/server/prompts/policy_system.md`). Key changes:

1. Remove "Embedded Resources" section
2. Add instruction to read MCP resources at the start
3. Add validate-as-you-go checkpoints after steps 2, 3, 4, 5, 7, and 8
4. Replace `${COMPONENT}` and `${ID_PREFIX}` with instructions to ask the user
5. Replace `${GEMARA_VERSION}` with `"1.0.0-rc.0"` (PolicyRiskWizardGemaraVersion)
6. Preserve the full wizard flow: all 10 steps from the MCP prompt

- [ ] **Step 1: Write policy-wizard.md**

Write `skills/gemara-artifact-authoring/policy-wizard.md`. Adapt the PR #38 policy system prompt applying the changes listed above. The file must contain:

- Title: `# Policy Wizard`
- Preamble stating: the assistant guides users through creating a Policy (Layer 3). Every contact, scope decision, import, risk disposition, and adherence requirement requires explicit user approval.
- Instruction to read MCP resources at the start.
- Available Tool table with `validate_gemara_artifact` and definitions `#Policy`, `#ControlCatalog`, `#RiskCatalog`, `#GuidanceCatalog` (for artifact type identification).
- Full outline with steps:
  1. Catalog and Artifact Import (with artifact type identification, import target mapping table)
  2. Scope and Metadata (`gemara-version: "1.0.0-rc.0"`, mapping-references)
  3. Define Contacts (RACI structure with Actor types)
  4. Define Scope (in/out dimensions: technologies, geopolitical, sensitivity, users, groups)
  5. Define Imports (catalogs with exclusions/constraints/assessment-requirement-modifications, guidance, policies)
  6. Implementation Plan (optional — notification, evaluation/enforcement timelines with ISO 8601 dates)
  7. Risk Dispositions (optional — mitigated risks with ID pattern `MR##`, accepted risks with ID pattern `AR##`, justifications)
  8. Define Adherence (evaluation methods, assessment plans with ID pattern `AP##`, enforcement methods, non-compliance)
  9. Assemble and Validate (with `cue vet` command)
  10. Next Steps (recommend Evaluation Log, Privateer plugins)
- Validate-as-you-go checkpoints after steps 2, 3, 4, 5, 7, and 8.
- Artifact Type Identification procedure (with layer-to-import-field mapping table).
- Policy Constraints section.

Source reference: the policy system prompt content already read from PR #38 diff earlier in this conversation.

- [ ] **Step 2: Verify the file is well-formed**

```bash
wc -l skills/gemara-artifact-authoring/policy-wizard.md
head -5 skills/gemara-artifact-authoring/policy-wizard.md
```

The file should be approximately 350-420 lines and start with `# Policy Wizard`.

- [ ] **Step 3: Commit**

```bash
git add skills/gemara-artifact-authoring/policy-wizard.md
git commit -m "feat: add policy wizard reference

Adapted from gemara-mcp PR #38 policy_system.md prompt. Covers RACI
contacts, scope, catalog/guidance/policy imports, implementation plan,
risk dispositions, and adherence with validate-as-you-go checkpoints.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 7: Create risk-catalog-wizard.md

**Files:**
- Create: `skills/gemara-artifact-authoring/risk-catalog-wizard.md`

Adapted from PR #38 system prompt (`internal/server/prompts/risk_catalog_system.md`). Key changes:

1. Remove "Embedded Resources" section
2. Add instruction to read MCP resources at the start
3. Add validate-as-you-go checkpoints after steps 2, 3, and each batch of risks in step 4
4. Replace `${COMPONENT}` and `${ID_PREFIX}` with instructions to ask the user
5. Replace `${GEMARA_VERSION}` with `"1.0.0-rc.0"` (PolicyRiskWizardGemaraVersion)
6. Preserve the full wizard flow including custom severity scale support

- [ ] **Step 1: Write risk-catalog-wizard.md**

Write `skills/gemara-artifact-authoring/risk-catalog-wizard.md`. Adapt the PR #38 risk catalog system prompt applying the changes listed above. The file must contain:

- Title: `# Risk Catalog Wizard`
- Preamble stating: the assistant guides users through creating a RiskCatalog (Layer 3). Every group, risk entry, severity assessment, and threat linkage requires explicit user approval.
- Instruction to read MCP resources at the start.
- Available Tool table with `validate_gemara_artifact` and definitions `#RiskCatalog`, `#ThreatCatalog`, `#ControlCatalog`, `#GuidanceCatalog`.
- Full outline with steps:
  1. Threat Catalog Import (with artifact type identification)
  2. Scope and Metadata (`gemara-version: "1.0.0-rc.0"`, mapping-references)
  3. Define Risk Groups (RiskCategory with appetite levels table: Minimal/Low/Moderate/High; custom severity scale support with mapping to Gemara values Low/Medium/High/Critical; appetite-to-max-severity matrix)
  4. Define Risks (ID pattern `RSK##`, severity with custom scale translation, optional RACI owner, impact, threat linkages via MultiEntryMapping)
  5. Assemble and Validate (with `cue vet` command)
  6. Next Steps (recommend Policy, Threat Catalog, Control Catalog)
- Validate-as-you-go checkpoints after steps 2, 3, and each batch of risks in step 4.
- Artifact Type Identification procedure (with layer-to-field mapping table for RiskCatalog).
- Risk Catalog Constraints section.

Source reference: the risk catalog system prompt content already read from PR #38 diff earlier in this conversation.

- [ ] **Step 2: Verify the file is well-formed**

```bash
wc -l skills/gemara-artifact-authoring/risk-catalog-wizard.md
head -5 skills/gemara-artifact-authoring/risk-catalog-wizard.md
```

The file should be approximately 240-300 lines and start with `# Risk Catalog Wizard`.

- [ ] **Step 3: Commit**

```bash
git add skills/gemara-artifact-authoring/risk-catalog-wizard.md
git commit -m "feat: add risk catalog wizard reference

Adapted from gemara-mcp PR #38 risk_catalog_system.md prompt. Covers
risk groups with appetite/max-severity matrix, custom severity scales,
RACI ownership, threat linkages, and validate-as-you-go checkpoints.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 8: Update README.md

**Files:**
- Rewrite: `README.md`

- [ ] **Step 1: Write the updated README**

Write `README.md`:

````markdown
# gemara-ai

Gemara's Claude Code plugin for security artifact authoring and validation.

## Installation

Install via Claude Code:

```bash
claude plugin add /path/to/gemara-ai
```

This installs both the **gemara-mcp** MCP server and the **gemara-artifact-authoring** skill.

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (for the gemara-mcp container)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

### What you get

| Component | Description |
|-----------|-------------|
| `gemara-mcp` server | MCP server providing `validate_gemara_artifact`, `migrate_gemara_artifact` tools and Gemara lexicon/schema resources |
| `gemara-mcp-advisory` server | Read-only mode — validation only, no migration or wizard prompts |
| `gemara-artifact-authoring` skill | Interactive artifact authoring with triage, prerequisite checking, and step-by-step wizards |

## Supported Artifact Types

| Layer | Artifact Type | Wizard |
|-------|--------------|--------|
| 2 | ThreatCatalog | Threat Assessment |
| 2 | ControlCatalog | Control Catalog |
| 3 | RiskCatalog | Risk Catalog |
| 3 | Policy | Policy |
| 3 | MappingDocument | Mapping Document |

## Usage

The skill automatically triages what artifact to build by:

1. Scanning your repo for existing Gemara artifacts
2. Analyzing any content you provide (YAML, docs, risk registers)
3. Recommending the next artifact based on layer dependencies

Every artifact produced is validated against the Gemara CUE schema before completion.

## MCP Server

The plugin bundles the [gemara-mcp](https://github.com/gemaraproj/gemara-mcp) server (v0.4.0) as a Docker container. The server provides:

- **Tools:** `validate_gemara_artifact`, `migrate_gemara_artifact`
- **Resources:** `gemara://lexicon`, `gemara://schema/definitions`
- **Prompts:** `threat_assessment`, `control_catalog`, `mapping_document`, `policy`, `risk_catalog`, `migration`

### Verifying the container image

```bash
cosign verify \
  --certificate-identity-regexp="https://github.com/gemaraproj/gemara-mcp/.github/workflows/release.yml" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/gemaraproj/gemara-mcp@sha256:0d05c93d237c08483a2b046cff16b1765c42f3cfcba152b02b0904da7d8a05f0
```

## License

[Apache License 2.0](LICENSE)
````

- [ ] **Step 2: Verify README renders correctly**

```bash
head -20 README.md
```

Expected: starts with `# gemara-ai` and includes the installation section.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: update README for Claude Code plugin installation

Replaces OpenPackage documentation with plugin installation instructions.
Documents the skill, MCP server modes, supported artifact types, and
container image verification.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 9: Validate end-to-end plugin structure

**Files:**
- None (verification only)

- [ ] **Step 1: Verify complete file tree**

```bash
find . -not -path './.git/*' -not -path './.claude/*' -not -path './docs/*' -type f | sort
```

Expected:

```
./.claude-plugin/plugin.json
./.mcp.json
./LICENSE
./README.md
./skills/gemara-artifact-authoring/SKILL.md
./skills/gemara-artifact-authoring/control-catalog-wizard.md
./skills/gemara-artifact-authoring/mapping-document-wizard.md
./skills/gemara-artifact-authoring/policy-wizard.md
./skills/gemara-artifact-authoring/risk-catalog-wizard.md
./skills/gemara-artifact-authoring/threat-assessment-wizard.md
```

- [ ] **Step 2: Verify plugin.json is valid JSON**

```bash
python3 -c "import json; json.load(open('.claude-plugin/plugin.json')); print('valid')"
```

Expected: `valid`

- [ ] **Step 3: Verify .mcp.json is valid JSON**

```bash
python3 -c "import json; json.load(open('.mcp.json')); print('valid')"
```

Expected: `valid`

- [ ] **Step 4: Verify SKILL.md frontmatter has required fields**

```bash
head -4 skills/gemara-artifact-authoring/SKILL.md
```

Expected: YAML frontmatter with `name` and `description` fields.

- [ ] **Step 5: Verify no OpenPackage remnants**

```bash
find . -name 'openpackage.yml' -not -path './.git/*' | wc -l
find . -path '*/gemara-mcp/*' -not -path './.git/*' | wc -l
```

Expected: both output `0`.

- [ ] **Step 6: Verify all wizard files exist and have content**

```bash
for f in threat-assessment-wizard.md control-catalog-wizard.md mapping-document-wizard.md policy-wizard.md risk-catalog-wizard.md; do
  echo "$f: $(wc -l < skills/gemara-artifact-authoring/$f) lines"
done
```

Expected: each file has 150+ lines.

- [ ] **Step 7: Verify each wizard references validate_gemara_artifact**

```bash
for f in skills/gemara-artifact-authoring/*-wizard.md; do
  count=$(grep -c 'validate_gemara_artifact' "$f")
  echo "$(basename $f): $count references"
done
```

Expected: each wizard has 3+ references to `validate_gemara_artifact`.

- [ ] **Step 8: Verify version pinning in wizard files**

```bash
grep -l 'v1.0.0' skills/gemara-artifact-authoring/threat-assessment-wizard.md skills/gemara-artifact-authoring/control-catalog-wizard.md skills/gemara-artifact-authoring/mapping-document-wizard.md
grep -l '1.0.0-rc.0' skills/gemara-artifact-authoring/policy-wizard.md skills/gemara-artifact-authoring/risk-catalog-wizard.md
```

Expected: first command lists threat-assessment, control-catalog, and mapping-document wizards. Second command lists policy and risk-catalog wizards.
