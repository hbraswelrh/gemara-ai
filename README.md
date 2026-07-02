# gemara-ai

Gemara's Claude Code plugin for security artifact authoring and validation.

## Installation

Install via Claude Code:

```bash
claude plugin install gemara
```

This installs both the **gemara-mcp** MCP server and the **gemara-artifact-authoring** skill.

### Prerequisites

- [Podman](https://podman.io/docs/installation) or [Docker](https://docs.docker.com/get-docker/) (for the gemara-mcp container)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

> **Note:** The plugin's `.mcp.json` uses `podman` as the container runtime command.
> If you use Docker, either create a symlink (`ln -s $(which docker) /usr/local/bin/podman`)
> or replace `"podman"` with `"docker"` in `.mcp.json`.

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

## Local Development

### 1. Clone the repository

```bash
git clone https://github.com/gemaraproj/gemara-ai.git
cd gemara-ai
```

### 2. Pull the container image

With Podman:

```bash
podman pull ghcr.io/gemaraproj/gemara-mcp@sha256:0d05c93d237c08483a2b046cff16b1765c42f3cfcba152b02b0904da7d8a05f0
```

With Docker:

```bash
docker pull ghcr.io/gemaraproj/gemara-mcp@sha256:0d05c93d237c08483a2b046cff16b1765c42f3cfcba152b02b0904da7d8a05f0
```

Optionally, [verify the image signature](#verifying-the-container-image) with cosign before use.

### 3. Validate the plugin

```bash
claude plugin validate .
```

This checks that `plugin.json`, `.mcp.json`, and all skill definitions are well-formed.

### 4. Install the plugin locally

```bash
claude plugin install gemara-ai --scope local
```

### 5. Verify the MCP servers connect

Start a Claude Code session and check that both servers are healthy:

```bash
claude mcp list
claude mcp get gemara-mcp
claude mcp get gemara-mcp-advisory
```

Both servers should show a connected status. If prompted, approve the project-scoped MCP servers from `.mcp.json`.

### 6. Test a skill

From a project directory, start Claude Code and invoke one of the artifact authoring wizards:

```
/gemara:gemara-artifact-authoring
```

The skill will scan the project for existing Gemara artifacts and guide you through creating one.

## MCP Server

The plugin bundles the [gemara-mcp](https://github.com/gemaraproj/gemara-mcp) server (v0.4.0) as a container image. The server provides:

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
