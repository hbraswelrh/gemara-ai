# Gemara Artifact Authoring Skill — Design Spec

## Overview

A single Claude Code plugin skill that triages, recommends, and guides users through creating schema-valid Gemara security artifacts. The skill replaces the current OpenPackage distribution with a unified Claude Code plugin that bundles both the gemara-mcp server configuration and the authoring skill.

## Goals

1. Single entry point for all Gemara artifact authoring (threat assessment, control catalog, mapping document, policy, risk catalog)
2. Smart triage that identifies what artifact to build based on existing repo artifacts, user-provided content, and layer dependency gaps
3. Self-contained wizard flows — the skill embeds the full authoring logic, not just pointers to MCP prompts
4. Every artifact produced must be schema-valid (enforced via `validate_gemara_artifact`)
5. Installable via `claude plugin add` — one install gives users the skill and the MCP server

## Non-Goals

- Migration wizard (v0→v1) — remains an MCP-only prompt; not included in the skill
- Evaluation, Enforcement, and Audit logs (Layers 5-7) — not supported in this version
- VectorCatalog, CapabilityCatalog, GuidanceCatalog, PrincipleCatalog, Lexicon authoring as standalone wizards — these are created as byproducts of other wizards (e.g., CapabilityCatalog during threat assessment) but don't have dedicated flows

## Plugin Structure

```
gemara-ai/
├── .claude-plugin/
│   └── plugin.json
├── .mcp.json
├── skills/
│   └── gemara-artifact-authoring/
│       ├── SKILL.md
│       ├── threat-assessment-wizard.md
│       ├── control-catalog-wizard.md
│       ├── mapping-document-wizard.md
│       ├── policy-wizard.md
│       └── risk-catalog-wizard.md
├── LICENSE
└── README.md
```

### Removed

- `gemara-mcp/` directory (OpenPackage layout)
- `openpackage.yml` (top-level and nested)

### plugin.json

```json
{
  "name": "gemara",
  "description": "Gemara security artifact authoring and validation",
  "version": "0.1.0"
}
```

### .mcp.json

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

## SKILL.md — Triage Skill

### Frontmatter

```yaml
---
name: gemara-artifact-authoring
description: Use when creating, iterating on, or validating Gemara security artifacts (threat catalogs, control catalogs, risk catalogs, policies, mapping documents). Also use when the user has security documentation, compliance requirements, threat models, or risk registers that could be structured as Gemara artifacts.
---
```

### Triage Logic

The skill performs three analysis steps in order:

#### Step 1: Scan the repo

Look for existing `.yaml` and `.yml` files containing a `metadata.type` field with a recognized Gemara artifact type. Build an inventory:

| Layer | Artifact Type | Found | File |
|-------|--------------|-------|------|
| 1 | GuidanceCatalog | - | - |
| 2 | ThreatCatalog | yes | threats.yaml |
| 2 | ControlCatalog | - | - |
| 2 | CapabilityCatalog | yes | capabilities.yaml |
| 3 | RiskCatalog | - | - |
| 3 | Policy | - | - |
| 3 | MappingDocument | - | - |

#### Step 2: Analyze user-provided content

If the user provides raw documentation, pasted YAML, or file references:

- **Structured YAML with `metadata.type`** — validate with `validate_gemara_artifact` to confirm type
- **Unstructured content** — analyze to identify the best artifact fit:
  - Lists of threats, attack scenarios, adversary techniques → ThreatCatalog
  - Security controls, safeguards, countermeasures → ControlCatalog
  - Compliance requirements, standards, regulations → GuidanceCatalog
  - Risk registers, risk assessments, impact analyses → RiskCatalog
  - Organizational policies, scope definitions, enforcement rules → Policy
  - Cross-references between two artifact sets → MappingDocument

#### Step 3: Recommend based on layer dependencies

Use the dependency graph to suggest the next artifact:

```
Layer 1: GuidanceCatalog
    ↓ (referenced by ControlCatalog guidelines)
Layer 2: ThreatCatalog → ControlCatalog ← CapabilityCatalog
    ↓ (feeds into)           ↓ (imported by)
Layer 3: RiskCatalog → Policy ← MappingDocument
```

Recommendation priority:
1. If no Layer 2 artifacts exist, recommend ThreatCatalog first
2. If ThreatCatalog exists but no ControlCatalog, recommend ControlCatalog
3. If both Layer 2 artifacts exist but no Layer 3, recommend RiskCatalog or Policy based on user context
4. MappingDocument is recommended when two or more artifacts exist that could be cross-referenced

#### Triage output

Present findings and recommendation:

```
## Existing Gemara Artifacts

| Layer | Type | File | Status |
|-------|------|------|--------|
| 2 | ThreatCatalog | threats.yaml | valid |
| 2 | CapabilityCatalog | capabilities.yaml | valid |

## Recommendation

You have a ThreatCatalog but no ControlCatalog. I recommend creating a
**ControlCatalog** to define security controls that mitigate your identified threats.

Alternatively, you could create:
- **RiskCatalog** — to document organizational risks linked to your threats
- **MappingDocument** — to map your threats to an external framework

Which artifact would you like to create?
```

### Soft-block prerequisite checking

When the user selects an artifact type, check prerequisites:

| Artifact Type | Prerequisites |
|--------------|---------------|
| ThreatCatalog | None |
| ControlCatalog | ThreatCatalog (for threat mappings) |
| RiskCatalog | ThreatCatalog (for threat linkages) |
| Policy | ControlCatalog (for imports), optionally RiskCatalog (for risk dispositions) |
| MappingDocument | Two existing artifacts to map between |

If prerequisites are missing, warn:

> "A Policy typically imports a ControlCatalog and may reference a RiskCatalog. Neither was found in this repo. You can proceed if you have external references (e.g., NIST 800-53), or we can build the prerequisites first. What would you like to do?"

Options:
1. Build the prerequisite first (redirect to that wizard)
2. Proceed with external references
3. Choose a different artifact type

### Routing to wizard files

After triage confirms the artifact type, the skill loads the corresponding wizard reference file. The skill instructs the assistant:

> Load and follow the step-by-step wizard in `{wizard-file}.md`. Use `validate_gemara_artifact` to validate sections as you build them. Do not present the artifact as complete until it passes final validation.

## Wizard Reference Files

Each wizard file is adapted from the gemara-mcp server's system prompts. The key adaptations:

1. **Remove MCP prompt framing** — no references to "embedded resources" or prompt message structure
2. **Add explicit tool instructions** — reference `validate_gemara_artifact` and `gemara://schema/definitions` by name
3. **Add validate-as-you-go checkpoints** — call `validate_gemara_artifact` after each major section, not just at the end
4. **Preserve the interaction model** — lettered-row tables with accept/select/modify/reject responses
5. **Preserve all constraints** — ID patterns, version pinning, YAML formatting rules

### Wizard sources

| Wizard File | Source | Artifact Type | CUE Definition |
|------------|--------|---------------|----------------|
| `threat-assessment-wizard.md` | Merged MCP prompt (main branch) | ThreatCatalog (+ CapabilityCatalog) | `#ThreatCatalog`, `#CapabilityCatalog` |
| `control-catalog-wizard.md` | Merged MCP prompt (main branch) | ControlCatalog | `#ControlCatalog` |
| `mapping-document-wizard.md` | PR #66 system prompt | MappingDocument | `#MappingDocument` |
| `policy-wizard.md` | PR #38 system prompt | Policy | `#Policy` |
| `risk-catalog-wizard.md` | PR #38 system prompt | RiskCatalog | `#RiskCatalog` |

### Common wizard structure

Every wizard follows this flow:

1. **Artifact Import** — identify and validate input artifacts
2. **Scope and Metadata** — collect component name, ID prefix, author info, generate metadata YAML
3. **Type-specific authoring steps** — groups, entries, mappings, etc. (varies per wizard)
4. **Assemble and Validate** — combine all sections, call `validate_gemara_artifact`, loop on errors until valid
5. **Next Steps** — recommend what to build next in the layer stack

### Validate-as-you-go checkpoints

Each wizard calls `validate_gemara_artifact` at these points:

- After metadata block is generated (validate partial artifact)
- After each major section is appended (groups, entries, mappings)
- Final assembly — full artifact validation is mandatory before presenting "Next Steps"

If validation fails at any checkpoint:
1. Diagnose the specific error from the tool response
2. Propose corrected YAML
3. Re-validate
4. Loop until valid or user intervenes

### Version pinning

Each wizard uses the correct `gemara-version` value:

| Wizard | gemara-version |
|--------|---------------|
| threat-assessment | Server's DefaultGemaraVersion (currently `"v1.0.0"` via go-gemara v0.5.0) |
| control-catalog | Server's DefaultGemaraVersion |
| mapping-document | Server's DefaultGemaraVersion |
| policy | `"1.0.0-rc.0"` (PolicyRiskWizardGemaraVersion) |
| risk-catalog | `"1.0.0-rc.0"` (PolicyRiskWizardGemaraVersion) |

### Local verification

After final validation passes, each wizard provides a `cue vet` command:

```bash
go install cuelang.org/go/cmd/cue@latest
cue vet -c -d '#<Definition>' github.com/gemaraproj/gemara@v1 <filename>.yaml
```

## Testing Strategy

Per the writing-skills TDD methodology, the skill will be tested with subagent pressure scenarios:

### Triage tests
- User provides a repo with existing ThreatCatalog — does the skill recommend ControlCatalog?
- User provides unstructured risk documentation — does the skill identify RiskCatalog?
- User requests a Policy with no ControlCatalog in repo — does the soft-block fire?

### Wizard flow tests
- Each wizard produces a schema-valid artifact when given cooperative inputs
- Validation failures trigger the auto-correct loop
- ID prefix constraints are enforced (regex `^[A-Z0-9.-]+$`)

### Schema validity tests
- Generated artifacts pass `validate_gemara_artifact` against the correct CUE definition
- Version pinning uses the correct `gemara-version` per wizard

## Migration Plan

### What changes in the repo

1. Remove `gemara-mcp/` directory and both `openpackage.yml` files
2. Add `.claude-plugin/plugin.json`
3. Add `.mcp.json`
4. Add `skills/gemara-artifact-authoring/` with SKILL.md and five wizard files
5. Update README.md with plugin installation instructions

### What stays the same

- The gemara-mcp Docker image and digest are unchanged
- The MCP tools (`validate_gemara_artifact`, `migrate_gemara_artifact`) and resources (`gemara://lexicon`, `gemara://schema/definitions`) remain available via the server
- The MCP server prompts continue to exist in the gemara-mcp repo — the skill is an alternative entry point, not a replacement for the server-side prompts

## Open Questions

None — all design decisions have been resolved through clarifying questions.
