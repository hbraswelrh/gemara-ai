---
name: gemara-docs-export
description: Use when the user wants to export Gemara security artifacts as markdown documentation pages for a website or docs site. Also use when the user mentions publishing, sharing, or documenting their threat catalogs, control catalogs, risk catalogs, policies, or mapping documents.
---

# Gemara Docs Export

Export valid Gemara security artifacts as generic markdown documentation pages. The user integrates the output into their docs framework (Docusaurus, Hugo, MkDocs, etc.).

## Tools and Resources

| Tool / Resource | Purpose |
|----------------|---------|
| `validate_gemara_artifact` | Validate artifact YAML before export |
| `gemara://lexicon` | Term definitions for the Gemara security model |
| `gemara://schema/definitions` | CUE schema definitions for all artifact types |

## Supported Artifact Types

| Layer | Type | CUE Definition |
|-------|------|----------------|
| 2 | ThreatCatalog | `#ThreatCatalog` |
| 2 | ControlCatalog | `#ControlCatalog` |
| 2 | CapabilityCatalog | `#CapabilityCatalog` |
| 3 | RiskCatalog | `#RiskCatalog` |
| 3 | Policy | `#Policy` |
| 2-3 | MappingDocument | `#MappingDocument` |

## Output Rules

Generate **generic markdown only**. No framework-specific output.

- No Docusaurus frontmatter (`sidebar_label`, `sidebar_position`)
- No Hugo frontmatter (`weight`, `menu`)
- No MkDocs nav configuration
- No MDX syntax
- Standard markdown headings, tables, and lists only

If the user mentions a specific framework, acknowledge it but still produce generic markdown. The user adapts the output to their framework after generation.

## Workflow

Follow these steps in order. Do not skip steps.

### Step 1: Scan & Inventory

Search for `.yaml` and `.yml` files containing a `metadata.type` field with a supported Gemara artifact type. Build an inventory table:

| Layer | Artifact Type | Found | File |
|-------|--------------|-------|------|
| 2 | ThreatCatalog | | |
| 2 | ControlCatalog | | |
| 2 | CapabilityCatalog | | |
| 3 | RiskCatalog | | |
| 3 | Policy | | |
| 2-3 | MappingDocument | | |

If the user provides specific file paths, still validate those files in Step 2 before exporting.

### Step 2: Validate Artifacts

Call `validate_gemara_artifact` on each found artifact using the correct CUE definition from the table above. Only valid artifacts can be exported.

If an artifact fails validation:
- Report the error to the user
- Exclude it from the export list
- Suggest using the appropriate authoring skill to fix it (`gemara-artifact-authoring`, `gemara-policy`, `gemara-risk-catalog`, or `gemara-mapping-document`)

### Step 3: Select Artifacts

Present the inventory of valid artifacts. Ask the user which to export. Default to all valid ones.

### Step 4: Page Structure

For each selected artifact, ask: **single page** or **section** (index page + one sub-page per entry).

- Recommend single page for artifacts with fewer than ~10 entries
- Recommend section mode for larger artifacts
- Policy is always single page

### Step 5: Output Directory

Ask the user where to write the generated files. Suggest `docs/security/` as a default.

### Step 6: Generate & Write

Transform each artifact to markdown following the format templates below. Write files to the chosen directory.

**File naming:**
- Single-page: `{artifact-type-slug}.md` (e.g., `threat-catalog.md`, `risk-catalog.md`)
- Section mode: `{artifact-type-slug}/index.md` + `{artifact-type-slug}/{ENTRY-ID}.md`

### Step 7: Summary

Report what was generated. Show a file tree of all created pages.

## Format Templates

### Common Page Header

Every generated page starts with:

```markdown
# {artifact title}

| Field | Value |
|-------|-------|
| **ID** | {metadata.id} |
| **Version** | {metadata.version} |
| **Gemara Version** | {metadata.gemara-version} |
| **Author** | {metadata.author.name} |
| **Description** | {metadata.description} |
```

### ThreatCatalog

**Groups** as an H2 summary table, then **each threat** as H3 (single-page) or its own page (section mode):

```markdown
## Threat Groups

| ID | Title | Description |
|----|-------|-------------|
| {group.id} | {group.title} | {group.description} |

## {threat.id}: {threat.title}

**Group:** {group.title}

{threat.description}

**Actors:**

| Name | Type |
|------|------|
| {actor.name} | {actor.type} |

**Capabilities:**
- {reference-id} / {entry reference-id}

**Vectors:**
- {reference-id} / {entry reference-id}
```

### ControlCatalog

**Groups** as H2 summary table, then **each control** as H3 or its own page:

```markdown
## Control Groups

| ID | Title | Description |
|----|-------|-------------|
| {group.id} | {group.title} | {group.description} |

## {control.id}: {control.title}

**Objective:** {control.objective}

**State:** {control.state}

**Assessment Requirements:**

| ID | Requirement | Applicability | Recommendation |
|----|-------------|---------------|----------------|
| {ar.id} | {ar.text} | {ar.applicability} | {ar.recommendation} |

**Linked Threats:**
- {reference-id} / {entry reference-id}

**Guidelines:**
- {reference-id} / {entry reference-id}
```

### CapabilityCatalog

**Groups** as H2 summary table, then **each capability** as H3 or its own page:

```markdown
## Capability Groups

| ID | Title | Description |
|----|-------|-------------|
| {group.id} | {group.title} | {group.description} |

## {capability.id}: {capability.title}

**Group:** {group.title}

{capability.description}
```

### RiskCatalog

**Groups** as H2 summary table (with appetite and max-severity), then **each risk** as H3 or its own page:

```markdown
## Risk Groups

| ID | Title | Description | Appetite | Max Severity |
|----|-------|-------------|----------|--------------|
| {group.id} | {group.title} | {group.description} | {group.appetite} | {group.max-severity} |

## {risk.id}: {risk.title}

**Severity:** {risk.severity}

**Rank:** {risk.rank}

{risk.description}

**Impact:** {risk.impact}

**Ownership:**

| Role | Name | Affiliation | Email |
|------|------|-------------|-------|
| Responsible | {name} | {affiliation} | {email} |
| Accountable | {name} | {affiliation} | {email} |

**Related Threats:**
- {reference-id} / {entry reference-id}
```

Omit fields that are not present in the artifact (e.g., if a risk has no `rank`, omit the Rank line). Do not show empty fields, placeholder values, or dashes (`--`) for missing data. If a table column would be empty for all rows, omit the column entirely.

### Policy

Always single page:

```markdown
## Contacts

| Role | Name | Affiliation | Email |
|------|------|-------------|-------|
| Responsible | {name} | {affiliation} | {email} |
| Accountable | {name} | {affiliation} | {email} |
| Consulted | {name} | {affiliation} | {email} |
| Informed | {name} | {affiliation} | {email} |

## Scope

### In Scope
- **Technologies:** {list}
- **Geopolitical:** {list}
- **Sensitivity:** {list}

### Out of Scope
- **Technologies:** {list}

## Imported Catalogs

| Reference ID | Exclusions |
|-------------|------------|
| {reference-id} | {exclusion list} |

## Implementation Plan

**Notification Process:** {text}

**Evaluation Timeline:** {start} to {end}

**Notes:** {text}

## Risk Dispositions

### Mitigated Risks

| ID | Risk Reference | Entry |
|----|---------------|-------|
| {id} | {reference-id} | {entry-id} |

### Accepted Risks

| ID | Target | Risk Reference | Entry | Justification |
|----|--------|---------------|-------|---------------|
| {id} | {target-id} | {reference-id} | {entry-id} | {justification} |

## Adherence

### Evaluation Methods

| Type | Mode | Description | Executor |
|------|------|-------------|----------|
| {type} | {mode} | {description} | {executor.name} |

### Assessment Plans

| ID | Requirement | Frequency | Evidence |
|----|-------------|-----------|----------|
| {id} | {requirement-id} | {frequency} | {evidence-requirements} |
```

Omit sections entirely if the corresponding field is not present in the artifact.

### MappingDocument

```markdown
**Source:** {source-reference.reference-id} ({source-reference.entry-type})

**Target:** {target-reference.reference-id} ({target-reference.entry-type})

{remarks}

## Mappings

| ID | Source | Relationship | Target(s) | Strength | Confidence | Rationale |
|----|--------|-------------|-----------|----------|------------|-----------|
| {id} | {source} | {relationship} | {targets} | {strength} | {confidence} | {rationale} |
```

In section mode, each mapping entry gets its own page with the full detail laid out vertically instead of in a table row.

## Cross-Reference Format

Render all cross-references as plain text. Use the format:

```
{reference-id} / {entry reference-id}
```

If the referenced artifact's `mapping-references` entry includes a URL, append it:

```
{reference-id} / {entry reference-id} (URL)
```

Do not generate markdown links between exported pages.
