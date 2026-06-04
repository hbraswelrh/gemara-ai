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
