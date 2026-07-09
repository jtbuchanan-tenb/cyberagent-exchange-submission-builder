# CyberAgents Exchange Submit

A Claude Code skill that walks you through submitting your cybersecurity AI agent, skill, MCP server, or playbook to the [Tenable CyberAgents Exchange](https://exchange.tenable.com).

## What This Does

Instead of manually following the Exchange contribution guide — cloning repos, filling out templates, validating fields — this skill handles it all interactively. Run one command and it will:

1. **Validate your repo** — checks that your agent code is pushed to a public GitHub repo, has a README, has an open source license, and scans for accidentally committed secrets
2. **Generate your listing** — interviews you about your agent, auto-detects what it can from your code, and assembles the listing metadata file
3. **Submit your pull request** — clones the Exchange content repo, creates a branch, places your listing, and opens a PR for review

Your listing goes live on the Exchange once a maintainer merges the PR.

## What Is the CyberAgents Exchange?

The [Tenable CyberAgents Exchange](https://exchange.tenable.com) is a community directory for cybersecurity AI agents, skills, tools, and playbooks. It's where security practitioners discover and share AI-powered automation for vulnerability management, incident response, threat detection, and more.

The Exchange is an index — your agent's source code stays in your own GitHub repository. The Exchange listing is metadata that points to your repo.

## What Is a Claude Code Skill?

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) is a markdown file that teaches Claude Code how to perform a specific workflow. When you install this skill, Claude Code gains the ability to guide you through the Exchange submission process — it knows the required fields, validates against the Exchange's live schema, and handles the GitHub operations.

Skills work in Claude Code and other AI coding assistants that support the Agent Skills standard.

## Installation

### Option 1: Clone to your global skills directory

```bash
git clone https://github.com/jtbuchanan-tenb/cyberagent-exchange-submission-builder.git ~/.claude/skills/cyberagents-exchange-submit
```

The skill will be available in all projects.

### Option 2: Clone to a specific project

```bash
git clone https://github.com/jtbuchanan-tenb/cyberagent-exchange-submission-builder.git .claude/skills/cyberagents-exchange-submit
```

The skill will be available whenever you're working in that project.

## Prerequisites

Before running the skill, make sure you have:

- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** installed
- **[GitHub CLI (`gh`)](https://cli.github.com/)** installed and authenticated (`gh auth login`)
- **Your project code** pushed to a **public** GitHub repository under your personal account

> **Note:** If you're a Tenable employee, you'll need to use a personal GitHub account (not your corporate EMU account) to submit listings. The skill will detect this and help you switch if needed.

## Usage

1. Open your terminal in your project's repository directory
2. Start Claude Code
3. Ask Claude to submit your project:

```
Submit my agent to the Tenable CyberAgents Exchange
```

Or use the skill directly:

```
/cyberagents-exchange-submit
```

Claude will guide you through each step interactively, asking for confirmation before any git operations.

## What You'll Need to Provide

The skill auto-detects as much as possible from your repo, but will ask you about:

- **Type** — Is this an agent, skill, tool, MCP server, or playbook?
- **Description** — A one-line summary of what your project does
- **Tags** — Keywords for discoverability (e.g., `vuln-management`, `incident-response`)
- **Framework** — What your agent is built with (e.g., Claude Code SKILL, LangChain, MCP SDK)
- **Integrations** — Which platforms it works with (e.g., Tenable, CrowdStrike, Splunk)

For **skills**, the skill auto-detects compatible platforms (Claude Code, Cursor, Windsurf, etc.) and invocation commands from your repo structure.

For **MCP servers**, it detects transport type, runtime, auth method, compatible clients, and exposed tools/resources/prompts from your code.

For **playbooks**, you'll select a subtype and provide additional details:
- **Standard** — Describe the agent chain (which agents/steps are involved and how they connect)
- **Sponsored** — Vendor-partnered playbook with co-branding (requires a logo URL, may include vendor-type agent references)
- **n8n** — An n8n workflow; the skill reads your `workflow.json` and generates a Mermaid diagram automatically

If your listing needs a value that isn't in the Exchange's current vocabulary (e.g., a new integration vendor), the skill will add it to the validator for you and include the update in your submission PR.

## Supported Listing Types

| Type | What it is |
|------|-----------|
| **Agent** | A standalone autonomous AI system with its own runtime |
| **Skill** | A SKILL.md file for Claude Code, Cursor, or similar assistants |
| **Tool** | A CLI tool, library, script, or standalone utility |
| **MCP Server** | A Model Context Protocol server exposing tools/resources |
| **Playbook** | A workflow that chains multiple agents together |

### Playbook Subtypes

| Subtype | What it is |
|---------|-----------|
| **Standard** | A multi-agent orchestration you've designed — requires listing the agents in the chain |
| **Sponsored** | A vendor-partnered playbook with co-branding — requires a logo URL and may include vendor-type agent references |
| **n8n** | An n8n workflow with a `workflow.json` file — generates a visual Mermaid workflow diagram |

## Roadmap

- **KPI capture for contributions** — Future versions of this skill will help contributors capture key performance indicators (KPIs) demonstrating how their submission has improved security operations. This may include self-reported metrics (e.g., time saved, incidents handled, coverage gained) or optional monitoring integrations that automatically collect usage and impact data points over time.

## Questions or Issues?

- For Exchange questions: visit [exchange.tenable.com](https://exchange.tenable.com) or [open an issue](https://github.com/tenable/cyberagents-exchange/issues) on the Exchange repository
- For issues with this skill: [open an issue](https://github.com/jtbuchanan-tenb/cyberagent-exchange-submission-builder/issues) in this repository
