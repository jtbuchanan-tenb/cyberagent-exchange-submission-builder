# Submission Checklist

Run through each section before submitting. Every item must pass.

## Contribution Agreement
- [ ] User has reviewed the [CyberAgents Exchange Contribution Agreement](https://github.com/tenable/cyberagents-exchange/blob/main/docs/CyberAgents_Contribution_Agreement)
- [ ] User has explicitly accepted the agreement
- [ ] `contribution_agreement_date` captured as ISO 8601 timestamp at the moment of acceptance

## Repo Structure
- [ ] CWD (or user-specified path) is a git repository
- [ ] Git remote points to a GitHub URL
- [ ] Remote is reachable (gh repo view succeeds)
- [ ] Repository visibility is PUBLIC
- [ ] No secrets or credentials detected in source files or git history
- [ ] GitHub account is personal (not EMU — username does NOT match `<org>_<name>` underscore pattern; hyphens are fine)
- [ ] README file exists at repo root (README.md, README.rst, or README)
- [ ] LICENSE file exists at repo root (LICENSE, LICENSE.md, LICENSE.txt, or COPYING)

## Listing Frontmatter (Agent)
Required fields — all must be present and valid:
- [ ] `name` — non-empty string
- [ ] `author` — matches the GitHub username from the remote
- [ ] `github_url` — valid URL, matches the actual repo remote
- [ ] `description` — non-empty, one-line summary
- [ ] `license` — valid SPDX identifier (MIT, Apache-2.0, GPL-3.0-only, BSD-2-Clause, etc.)
- [ ] `tier` — must be `"contributed"` for new submissions
- [ ] `tags` — non-empty array of lowercase strings
- [ ] `integrations` — array, each value validated against `Entry.integrations` Literal values in `validator.py`
- [ ] `date_added` — today's date in YYYY-MM-DD format
- [ ] `contribution_agreement_date` — ISO 8601 datetime from Step 1.0
- [ ] `works_with_tenable_hexa_mcp` — boolean, set based on Step 2.4b (only relevant if Tenable is an integration)

## Listing Frontmatter (Skill)
Required fields — all must be present and valid:
- [ ] All common fields (name, author, github_url, description, license, tier, tags, integrations, date_added, contribution_agreement_date)
- [ ] `works_with_tenable_hexa_mcp` — boolean, set based on Step 2.4b
- [ ] `compatible_platforms` — array, each validated against `Skill.compatible_platforms` Literal values in `validator.py`
- [ ] `invocation` — string, the command/trigger name

## Listing Frontmatter (Playbook — Standard)
Required fields — all must be present and valid:
- [ ] `playbook_type` — must be `"standard"`
- [ ] `name` — non-empty string
- [ ] `author` — matches the GitHub username from the remote
- [ ] `github_url` — valid URL, matches the actual repo remote
- [ ] `description` — non-empty, one-line summary
- [ ] `license` — valid SPDX identifier
- [ ] `tier` — must be `"contributed"`
- [ ] `tags` — non-empty array of lowercase strings
- [ ] `integrations` — array, each value validated against `Entry.integrations` Literal values in `validator.py`
- [ ] `agents_used` — non-empty array, each entry has:
  - `name` — non-empty string
  - `role` — non-empty string describing the agent's role in the chain
  - `type` — one of: `exchange`, `github`, `info` (NO `vendor`)
  - `ref` — required for `exchange` (slug) and `github` (URL); optional for `info`
- [ ] `date_added` — today's date in YYYY-MM-DD format
- [ ] `contribution_agreement_date` — ISO 8601 datetime from Step 1.0
- [ ] `works_with_tenable_hexa_mcp` — boolean, set based on Step 2.4b

## Listing Frontmatter (Playbook — Sponsored)
Required fields — all must be present and valid:
- [ ] `playbook_type` — must be `"sponsored"`
- [ ] `name` — non-empty string
- [ ] `author` — matches the GitHub username from the remote
- [ ] `github_url` — valid URL, matches the actual repo remote
- [ ] `description` — non-empty, one-line summary
- [ ] `license` — valid SPDX identifier
- [ ] `tier` — must be `"contributed"`
- [ ] `tags` — non-empty array of lowercase strings
- [ ] `integrations` — array, each value validated against `Entry.integrations` Literal values in `validator.py`
- [ ] `agents_used` — non-empty array, each entry has:
  - `name` — non-empty string
  - `role` — non-empty string describing the agent's role in the chain
  - `type` — one of: `exchange`, `github`, `info`, `vendor` (`vendor` allowed for sponsored only)
  - `ref` — required for `exchange` (slug), `github` (URL), and `vendor` (URL); optional for `info`
- [ ] `logo` — publicly accessible image URL (PNG or SVG recommended)
- [ ] `date_added` — today's date in YYYY-MM-DD format
- [ ] `contribution_agreement_date` — ISO 8601 datetime from Step 1.0
- [ ] `works_with_tenable_hexa_mcp` — boolean, set based on Step 2.4b

## Listing Frontmatter (Playbook — n8n)
Required fields — all must be present and valid:
- [ ] `playbook_type` — must be `"n8n"`
- [ ] `name` — non-empty string
- [ ] `author` — matches the GitHub username from the remote
- [ ] `github_url` — valid URL, matches the actual repo remote
- [ ] `description` — non-empty, one-line summary
- [ ] `license` — valid SPDX identifier
- [ ] `tier` — must be `"contributed"`
- [ ] `tags` — non-empty array of lowercase strings
- [ ] `integrations` — array, each value validated against `Entry.integrations` Literal values in `validator.py`
- [ ] `workflow_diagram` — Mermaid flowchart string (required)
- [ ] `agents_used` — optional array; if present, each entry has:
  - `name` — non-empty string
  - `role` — non-empty string describing the agent's role in the chain
  - `type` — one of: `exchange`, `github`, `info` (NO `vendor`)
  - `ref` — required for `exchange` (slug) and `github` (URL); optional for `info`
- [ ] `date_added` — today's date in YYYY-MM-DD format
- [ ] `contribution_agreement_date` — ISO 8601 datetime from Step 1.0
- [ ] `works_with_tenable_hexa_mcp` — boolean, set based on Step 2.4b

## Listing Frontmatter (MCP Server)
Required fields — all must be present and valid:
- [ ] All common fields (name, author, github_url, description, license, tier, tags, integrations, date_added, contribution_agreement_date)
- [ ] `works_with_tenable_hexa_mcp` — boolean, set based on Step 2.4b
- [ ] `transport` — validated against `MCPServer.transport` Literal values in `validator.py`
- [ ] `runtime` — validated against `MCPServer.runtime` Literal values in `validator.py`
- [ ] `auth_method` — validated against `MCPServer.auth_method` Literal values in `validator.py`
- [ ] `compatible_clients` — array, each validated against `MCPServer.compatible_clients` Literal values in `validator.py`
- [ ] `tools_exposed` — array of {name, description} objects (can be empty)
- [ ] `resources_exposed` — array of {name, description} objects (can be empty)
- [ ] `prompts_exposed` — array of {name, description} objects (can be empty)

## Exchange Submission
- [ ] `gh` CLI is authenticated (`gh auth status` succeeds)
- [ ] Authenticated user can access `tenable/cyberagents-exchange`
- [ ] Listing filename is a valid slug (lowercase, hyphens, no spaces, no special chars)
- [ ] No filename conflict with existing files in target directory (`agents/`, `mcp-servers/`, `playbooks/`, or `skills/`)
- [ ] PR title follows format: `"Add listing: <Agent Name>"`
