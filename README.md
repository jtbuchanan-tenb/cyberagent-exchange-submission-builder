# CyberAgents Exchange Submit

A Claude Code skill that walks you through submitting your cybersecurity AI agent or playbook to the [Tenable CyberAgents Exchange](https://exchange.tenable.com).

## What This Does

Instead of manually following the Exchange contribution guide — forking repos, filling out templates, validating fields — this skill handles it all interactively. Run one command and it will:

1. **Validate your repo** — checks that your agent code is pushed to GitHub, has a README, and has an open source license
2. **Generate your listing** — interviews you about your agent, auto-detects what it can from your code, and assembles the listing metadata file
3. **Submit your pull request** — forks the Exchange content repo, places your listing, and opens a PR for review

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
- **Your agent/playbook code** pushed to a GitHub repository under your personal account
- **Access to the Exchange content repo** — contact Justin Buchanan ([@jtbuchanan-tenb](https://github.com/jtbuchanan-tenb)), Patrick Ramseier ([@pramseier-tenb](https://github.com/pramseier-tenb)), or DJ Zito to be added as a collaborator

> **Note:** If you're a Tenable employee, you'll need to use a personal GitHub account (not your corporate EMU account) to submit listings. The skill will detect this and help you switch if needed.

## Usage

1. Open your terminal in your agent's repository directory
2. Start Claude Code
3. Ask Claude to submit your agent:

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

- **Type** — Is this an agent, skill, tool, or MCP server?
- **Description** — A one-line summary of what your agent does
- **Tags** — Keywords for discoverability (e.g., `vuln-management`, `incident-response`)
- **Framework** — What your agent is built with (e.g., Claude Code SKILL, LangChain, MCP SDK)
- **Integrations** — Which platforms it works with (e.g., Tenable, CrowdStrike, Splunk)

For playbooks, you'll also describe the agent chain — which agents/steps are involved and how they connect.

## Supported Listing Types

| Type | What it is |
|------|-----------|
| **Agent** | A standalone autonomous AI system with its own runtime |
| **Skill** | A SKILL.md file for Claude Code, Cursor, or similar assistants |
| **Tool** | A CLI tool, library, script, or standalone utility |
| **MCP Server** | A Model Context Protocol server exposing tools/resources |
| **Playbook** | A workflow that chains multiple agents together |

## Questions or Issues?

- For Exchange access or listing questions: contact the Exchange maintainers listed above
- For issues with this skill: open an issue in this repository
