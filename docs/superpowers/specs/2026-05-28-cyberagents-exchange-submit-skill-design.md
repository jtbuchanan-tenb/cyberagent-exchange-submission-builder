# CyberAgents Exchange Submit Skill — Design Spec

## Overview

A Claude Code skill (`/cyberagents-exchange-submit`) that automates the process of submitting an agent, playbook, or MCP server listing to the Tenable CyberAgents Exchange. The skill validates the user's repo, generates listing metadata through an interview + generative approach, handles licensing, and submits a pull request to the exchange content repository.

## Skill Metadata

- **Name:** `cyberagents-exchange-submit`
- **Trigger description:** "Submit an agent, playbook, or MCP server to the Tenable CyberAgents Exchange. Use when a user wants to list their cybersecurity AI agent or workflow on the Tenable Exchange (exchange.tenable.com), verify their repo meets requirements, generate listing metadata, and open a pull request to the exchange content repository. Triggers on: submit to exchange, Tenable CyberAgents Exchange, Exchange Tenable, list my agent, publish to exchange."

## File Structure

```
cyberagents-exchange-submit/
├── SKILL.md                          # Main flow orchestrator
└── references/
    ├── agent-template.md             # Agent listing template + field guide
    ├── playbook-template.md          # Playbook listing template + field guide
    └── submission-checklist.md       # Pre-submission validation rules
```

Integrations and types/categories are fetched live from the exchange founders repo at runtime (not stored as static reference files).

## Target Repositories

- **Content repo (PR target):** `tenable-cyberagents-exchange/exchange-founders-prelaunch-agents`
- **Website repo (consumes content):** `tenb-Marketing/cyberagents-exchange-website`

## Taxonomy

The `type` field (formerly `category`) describes how the listing is packaged/delivered:

| Value | Description |
|-------|-------------|
| `agent` | Standalone autonomous AI system with its own runtime/control loop |
| `skill` | Agent skill file (SKILL.md — works in Claude Code, Cursor, etc.) |
| `tool` | CLI, library, script, or standalone utility |
| `mcp-server` | MCP protocol server exposing tools/resources |

This field answers "what form does this take?" — topical classification lives in `tags`.

---

## Phase 1: Locate & Validate Agent Repo

### Step 1.1 — Detect repo

- Check if CWD is a git repo with a GitHub remote.
- If not, ask the user for the path to their agent/playbook repo.
- Advise users that running the skill from within their agent repo is recommended.

### Step 1.2 — Verify repo structure

- Confirm a README exists at repo root. If missing, warn and suggest creating one (don't block).
- Parse `git remote -v` to extract GitHub URL.

### Step 1.3 — Verify pushed to GitHub

- Confirm the remote is reachable (`gh repo view`).
- If the remote doesn't exist or isn't pushed, guide the user to push.

### Step 1.4 — Check account type

- Parse the remote URL to determine if it's under a personal account or an org/EMU.
- EMU accounts typically follow `<enterprise>_<username>` pattern.
- If EMU: warn clearly — "EMU accounts can't fork external repos. You need to push this to a personal GitHub account." Guide them through creating/switching to a personal account.

---

## Phase 2: Interview & Generate Listing

### Step 2.1 — License validation

1. Scan repo root for license file: `LICENSE`, `LICENSE.md`, `LICENSE.txt`, `COPYING`, or a `license` field in `package.json`/`pyproject.toml`/`Cargo.toml`.
2. If found: identify the license type, auto-fill `license` field with SPDX identifier (e.g., `"MIT"`).
3. If not found:
   - Explain that the Exchange requires an open source license.
   - Recommend MIT: "MIT is the most permissive and widely used license for open source tools. It lets anyone use, modify, and distribute your work with minimal restrictions."
   - Offer to create the `LICENSE` file with proper MIT text (user's name/GitHub username + current year).
   - Support Apache-2.0, GPL-3.0, BSD-2-Clause as alternatives if user prefers.
   - Confirm before writing.
4. Use SPDX short identifier in the listing (e.g., `"MIT"`, not `"MIT License"`).
5. If repo has `package.json` or equivalent without a `license` field, suggest adding it for ecosystem consistency.

### Step 2.2 — Type selection

- Ask: "Are you submitting an agent, skill, tool, or MCP server?"
- Fetch `data/types.json` live from founders repo to show descriptions for each type. (Fallback: try `data/categories.json` if `types.json` not found yet during transition.)
- Validate selection against live list.

### Step 2.3 — Auto-detected fields (present for confirmation)

| Field | Source |
|-------|--------|
| `name` | Repo name or README title |
| `github_url` | Git remote URL |
| `author` | GitHub username from remote |
| `license` | LICENSE file (or just created) |
| `date_added` | Today's date |
| `tier` | Always `"unreviewed"` |

### Step 2.4 — Generative-assisted fields (interview + suggest)

- **`description`** — Read README, propose a one-liner, ask if it captures the essence.
- **`tags`** — Suggest based on README content and code keywords. Let user add/remove.
- **`framework`** — Infer from code structure (SKILL.md present → "Claude Code SKILL", `mcp` in deps → "MCP SDK", Python agent patterns → framework name). Confirm with user.
- **`integrations`** — Suggest based on imports, API references, README mentions. Validate against live `data/integrations.json` from founders repo. Show available options. Fuzzy-match on typos (e.g., "Crowdstrike" → "CrowdStrike").

### Step 2.5 — Playbook-specific: `agents_used` chain

- Ask: "Walk me through the agents/steps in this workflow. For each one, I need the name, what role it plays, and whether it's listed on the Exchange, on GitHub, or is a manual/info step."
- For each agent: determine `type` (exchange/github/info) and `ref`.
- Present the full chain for review.

### Step 2.6 — Body content

- Generate markdown body from README context (description, "What it does", "How it works").
- Present for editing — user can trim, rewrite, or approve as-is.

### Step 2.7 — Final review & commit

- Show the complete assembled listing markdown in full.
- Ask for any final changes.
- **Confirm before git action:** commit the listing `.md` to the agent repo and push.

---

## Phase 3: Exchange Submission

### Step 3.1 — Verify GitHub CLI auth

- Run `gh auth status`.
- If not authenticated, guide the user through `gh auth login`.

### Step 3.2 — Verify exchange repo access

- `gh api repos/tenable-cyberagents-exchange/exchange-founders-prelaunch-agents` to test access.
- On 404/403: explain this is a private repo, provide contacts:
  - Justin Buchanan — @jtbuchanan-tenb
  - Patrick Ramseier — @pramseier-tenb
  - DJ Zito
- Pause and wait for user to confirm they have access before continuing.

### Step 3.3 — Fork the exchange repo

- **Confirm before git action.**
- Check if fork already exists (`gh repo list --fork` or `gh api`).
- If not, fork via `gh repo fork tenable-cyberagents-exchange/exchange-founders-prelaunch-agents`.

### Step 3.4 — Clone and prepare branch

- Clone the fork to a temp directory (or use existing clone if found).
- Create branch: `add-<listing-slug>` (e.g., `add-my-vuln-scanner`).

### Step 3.5 — Place listing file

- Generate slug from `name` field: lowercase, spaces/special chars → hyphens, strip leading/trailing hyphens.
  - Example: "My Cool Vuln Scanner" → `my-cool-vuln-scanner`
- Check for filename conflicts with existing files in target directory.
- Copy listing to `agents/<slug>.md` or `playbooks/<slug>.md`.

### Step 3.6 — Commit, push, and PR

- **Confirm before git action.**
- Commit with message: `"Add listing: <Agent Name>"`
- Push branch to fork.
- Create PR via `gh pr create`:
  - **Title:** `"Add listing: <Agent Name>"`
  - **Body:** Link to agent repo, one-line description, confirms checklist items met.
- Report the PR URL to the user.

---

## Validation Rules

### Pre-flight (Phase 1)
- Git repo with GitHub remote exists
- Remote is reachable
- Account is personal (not EMU)
- README exists (warn if missing)

### Listing (Phase 2)
- All required frontmatter fields present
- `integrations` values validated against live `data/integrations.json`
- `type` validated against live categories/types list
- `license` is a valid SPDX identifier
- `tier` is always `"unreviewed"`
- `date_added` is today
- `github_url` matches actual repo remote
- LICENSE file exists in repo

### Exchange (Phase 3)
- `gh` CLI authenticated
- User has access to content repo
- No filename conflicts with existing listings
- Slug is valid (lowercase, hyphens, no spaces)

---

## User Confirmation Points

The skill confirms with the user before each of these actions:
1. Creating a LICENSE file (if missing)
2. Committing the listing `.md` to the agent repo
3. Pushing the agent repo to GitHub
4. Forking the exchange repo
5. Committing, pushing, and creating the PR

---

## Future Extensions

- **MCP server type:** When added to the website, add `references/mcp-server-template.md` and any MCP-specific interview questions.
- **Listing updates:** Support updating an existing listing (detect existing PR or file, update in place).
- **Validation CI:** The founders repo has `scripts/` for CI validation — the skill could run equivalent checks locally before submitting.
