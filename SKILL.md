---
name: cyberagents-exchange-submit
description: "Submit an agent, skill, MCP server, or playbook to the Tenable CyberAgents Exchange. Use when a user wants to list their cybersecurity AI agent, skill, or workflow on the Tenable Exchange (exchange.tenable.com), verify their repo meets requirements, generate listing metadata, and open a pull request to the exchange content repository. Triggers on: submit to exchange, Tenable CyberAgents Exchange, Exchange Tenable, list my agent, list my skill, publish to exchange."
---

# CyberAgents Exchange Submission

Guide the user through submitting their agent, skill, MCP server, or playbook to the Tenable CyberAgents Exchange. This is a multi-phase process: validate their repo, generate a listing file, and submit a PR to the exchange content repository.

The exchange content repo is: `tenable-cyberagents-exchange/exchange-founders-prelaunch-agents`

Always confirm with the user before any git commit, push, or PR creation.

---

## Phase 1: Locate & Validate the Agent Repo

### Step 1.1 — Detect the repo

Check if the current working directory is a git repository with a GitHub remote:

```bash
git remote -v
```

If this succeeds and shows a GitHub URL (github.com), use CWD as the agent repo.

If CWD is not a git repo or has no GitHub remote, ask the user:
> "I don't detect a GitHub-connected repo in the current directory. For best results, run this skill from within your agent/playbook repository. Can you provide the path to your repo?"

Once you have the repo path, cd into it and continue.

### Step 1.2 — Verify repo structure

Check for a README:
```bash
ls README* readme*
```

If no README exists, warn:
> "Your repo doesn't have a README file. The Exchange strongly recommends a README explaining what your agent does and how to use it. Would you like to continue anyway, or create a README first?"

Do not block — let the user proceed if they choose.

Extract the GitHub remote URL:
```bash
git remote get-url origin
```

Parse the owner and repo name from the URL. Handle both SSH (`git@github.com:owner/repo.git`) and HTTPS (`https://github.com/owner/repo`) formats.

### Step 1.3 — Verify pushed to GitHub and public

Confirm the remote repo exists and is accessible:
```bash
gh repo view <owner>/<repo> --json name,url,visibility
```

If this fails, the repo either doesn't exist on GitHub yet or isn't pushed. Guide the user:
> "I can't reach your repo on GitHub. Have you pushed it yet? You can push with:"
> ```
> git push -u origin main
> ```

If the repo is **private** (visibility is not "PUBLIC"), warn the user:
> "Your repo is currently private. The CyberAgents Exchange requires all listed projects to be in public repositories so that users can access and evaluate your work.
>
> You can make it public in your repo settings, or I can do it for you:"
> ```
> gh repo edit <owner>/<repo> --visibility public
> ```
> "Would you like me to make it public now, or would you prefer to do it yourself?"

**Do not proceed past Phase 1 until the repo is public.** Re-check after the user confirms.

### Step 1.4 — Scan for secrets

Before proceeding, scan the repo for accidentally committed secrets:

```bash
# Check for common secret file patterns
ls .env .env.local .env.production .env.* credentials.json service-account*.json *secret* *token* 2>/dev/null

# Search for hardcoded secrets in source files
grep -rn --include="*.py" --include="*.ts" --include="*.js" --include="*.go" --include="*.rs" --include="*.yaml" --include="*.yml" --include="*.toml" --include="*.json" -E '(api[_-]?key|api[_-]?secret|auth[_-]?token|access[_-]?token|secret[_-]?key|private[_-]?key|password)\s*[:=]\s*["\x27][A-Za-z0-9+/=_-]{16,}' . 2>/dev/null | grep -v node_modules | grep -v .venv | grep -v __pycache__

# Check for AWS/cloud credential patterns
grep -rn --include="*.py" --include="*.ts" --include="*.js" --include="*.go" --include="*.env*" -E '(AKIA[0-9A-Z]{16}|sk-[a-zA-Z0-9]{20,}|ghp_[a-zA-Z0-9]{36}|gho_[a-zA-Z0-9]{36}|xox[bpas]-[a-zA-Z0-9-]+)' . 2>/dev/null | grep -v node_modules | grep -v .venv
```

**If any matches are found**, report them to the user:
> "I found what appear to be secrets or credentials in your repository:"
> - `<file>:<line>` — <description of what was found>
>
> "Since this repo needs to be public for the Exchange, these should be removed before proceeding. Common fixes:"
> - Move secrets to environment variables and add the files to `.gitignore`
> - Use `git filter-branch` or [BFG Repo-Cleaner](https://reclaimtheweb.com/bfg-repo-cleaner/) to purge them from git history
> - Rotate any exposed credentials immediately
>
> "Would you like help resolving these?"

**Do not proceed past Phase 1 until the user has addressed the findings or confirmed they are false positives** (e.g., example/placeholder values, test fixtures).

If no matches are found, continue silently.

### Step 1.5 — Check account type (EMU detection)

Inspect the owner from the remote URL. EMU (Enterprise Managed User) accounts follow the pattern `<enterprise>_<username>` with an **underscore** (e.g., `tenable_jbuchanan`). Hyphens in usernames (e.g., `jtbuchanan-tenb`) are normal personal accounts — do NOT flag these.

Only if the owner contains an underscore separating what looks like an org prefix from a username, warn:
> "It looks like this repo is under an Enterprise Managed User account (`<owner>`). EMU accounts cannot access the Exchange content repository. You'll need to push this repo to a personal GitHub account instead."

Ask: "Do you have a personal GitHub account? If so, what's the username? I can help you add it as a remote and push."

If they don't have one, guide them to github.com/signup to create a free account.

After any account switch, re-run the validation from Step 1.2.

---

## Phase 2: Interview & Generate Listing

### First: Fetch live data from the exchange repo

Before starting the interview, fetch the validator to extract controlled vocabularies:

```bash
# Fetch the validator which contains all controlled vocabularies as Literal types
gh api repos/tenable-cyberagents-exchange/exchange-founders-prelaunch-agents/contents/validator.py --jq '.content' | base64 -d
```

Parse the Pydantic models in `validator.py` to extract valid values from `Literal[...]` type annotations:
- **Integrations** — from `Entry.integrations` field's Literal values
- **Types** — from `Agent.type` field's Literal values (e.g., "agent", "tool", "mcp-server")
- **Tiers** — from `Entry.tier` field's Literal values
- **Platforms** — from `Skill.compatible_platforms` field's Literal values
- **Transports** — from `MCPServer.transport` field's Literal values
- **Runtimes** — from `MCPServer.runtime` field's Literal values
- **Auth methods** — from `MCPServer.auth_method` field's Literal values
- **Clients** — from `MCPServer.compatible_clients` field's Literal values

Store these results for validation throughout Phase 2. You'll also need the appropriate template later — fetch it after the user selects their type:

```bash
# For agents/tools:
gh api repos/tenable-cyberagents-exchange/exchange-founders-prelaunch-agents/contents/templates/agent-template.md --jq '.content' | base64 -d

# For skills:
gh api repos/tenable-cyberagents-exchange/exchange-founders-prelaunch-agents/contents/templates/skill-template.md --jq '.content' | base64 -d

# For MCP servers:
gh api repos/tenable-cyberagents-exchange/exchange-founders-prelaunch-agents/contents/templates/mcp-server-template.md --jq '.content' | base64 -d

# For playbooks:
gh api repos/tenable-cyberagents-exchange/exchange-founders-prelaunch-agents/contents/templates/playbook-template.md --jq '.content' | base64 -d
```

### Step 2.1 — License validation

Scan the repo root for a license file:
```bash
ls LICENSE* COPYING* license* copying* 2>/dev/null
```

Also check for a `license` field in common manifests:
```bash
cat package.json 2>/dev/null | grep -i '"license"'
cat pyproject.toml 2>/dev/null | grep -i 'license'
cat Cargo.toml 2>/dev/null | grep -i 'license'
```

**If a license file is found:** Identify the license type by reading the first few lines. Map it to the SPDX identifier:
- "MIT License" → `MIT`
- "Apache License, Version 2.0" → `Apache-2.0`
- "GNU General Public License" → `GPL-3.0-only`
- "BSD 2-Clause" → `BSD-2-Clause`

Confirm with the user: "I detected a `<license-type>` license. I'll use `<SPDX-id>` in your listing. Correct?"

**If no license file is found:**

Tell the user:
> "The CyberAgents Exchange requires all listed projects to have an open source license. Your repo doesn't appear to have a LICENSE file.
>
> I recommend the **MIT License** — it's the most permissive and widely used license for open source tools. It lets anyone use, modify, and distribute your work with minimal restrictions.
>
> Would you like me to create a LICENSE file with the MIT license? Or would you prefer a different license? (Options: MIT, Apache-2.0, GPL-3.0, BSD-2-Clause)"

If the user agrees to MIT (or picks another), create the LICENSE file:
- Use the user's name (from their GitHub profile or ask them) and the current year
- Write the standard license text to `LICENSE` in the repo root
- **Confirm before writing**

If the repo has a `package.json` without a `license` field, suggest:
> "I notice your package.json doesn't have a `license` field. For ecosystem consistency, I'd recommend adding `\"license\": \"MIT\"` to it. Want me to do that?"

### Step 2.2 — Type selection

Ask the user:
> "What type of listing are you submitting?"

Present the types fetched from the founders repo with their descriptions, plus `mcp-server` (which is now its own collection). If the fetched data is the new object format (with `id` and `description` fields), show each option with its description. If it's the old flat array, show just the values:

> - **agent** — A standalone autonomous AI system with its own runtime and control loop
> - **skill** — An agent skill file (SKILL.md) that extends AI coding assistants (goes to `skills/` directory)
> - **tool** — A CLI tool, library, script, or standalone utility
> - **mcp-server** — A Model Context Protocol server exposing data sources or actions

Validate their selection: `agent`, `skill`, and `tool` must match the `Agent.type` Literal values from the validator. `mcp-server` is always valid (it's a separate collection now). `playbook` is also always valid.

After type selection, fetch the appropriate template (agent-template.md for agent/tool, skill-template.md for skill, mcp-server-template.md for mcp-server, playbook-template.md for playbooks).

### Step 2.3 — Auto-detected fields

Determine these fields automatically and present them for confirmation:

| Field | How to detect |
|-------|---------------|
| `name` | Parse the first `# heading` from README.md. Fallback: repo name with hyphens replaced by spaces and title-cased. |
| `github_url` | The HTTPS URL from git remote (e.g., `https://github.com/owner/repo`) |
| `author` | The owner from the GitHub remote URL |
| `license` | The SPDX identifier from Step 2.1 |
| `date_added` | Today's date in YYYY-MM-DD format |
| `tier` | Always `"unreviewed"` — hardcoded, do not ask |

Present these to the user:
> "Here's what I've detected from your repo. Please confirm or correct:"
> - **Name:** <detected>
> - **GitHub URL:** <detected>
> - **Author:** <detected>
> - **License:** <detected>
> - **Date added:** <today>
> - **Tier:** unreviewed (all new submissions start here)

### Step 2.4 — Generative-assisted fields

For these fields, read the repo context (README, code structure, imports, dependencies) and propose values, but let the user guide the final answer.

**`description`** — Read the README and propose a one-line summary (under 120 characters). Ask:
> "Here's my suggested one-line description: '<proposed>'. Does this capture what your agent does, or would you like to revise it?"

**`tags`** — Suggest 3-7 tags based on README content, code keywords, and the domain. Tags should be lowercase, hyphenated. Ask:
> "Suggested tags: `[<tag1>, <tag2>, ...]`. Want to add, remove, or change any?"

**`framework`** — Infer from repo structure:
- `SKILL.md` present → "Claude Code SKILL"
- `mcp` or `@modelcontextprotocol` in package.json/deps → "MCP SDK"
- `langchain` in deps → "LangChain"
- `crewai` in deps → "CrewAI"
- `autogen` in deps → "AutoGen"
- Python with agent patterns → ask the user
- Otherwise → ask the user

Confirm: "I'm guessing the framework is `<detected>`. Is that right?"

**`integrations`** — Search the README and code for references to known platforms. Cross-reference with the integrations list parsed from the validator. Suggest matches and show the full list of valid options:
> "Based on your code, I think these integrations apply: `[<suggested>]`. Here's the full list of valid integrations: `[<all valid>]`. Want to add or remove any?"

If the user types an integration that's not in the list, fuzzy-match against valid values (e.g., "crowdstrike" → "CrowdStrike", "sentinelone" → "SentinelOne") and suggest the correction.

If no fuzzy match is found and the user confirms the value is correct (e.g., a new vendor not yet in the vocabulary), inform them:
> "That value isn't in the current controlled vocabulary. I can include an update to `validator.py` in your submission PR to add it — the maintainers will review the vocabulary addition alongside your listing. Want me to do that?"

If the user agrees, track the new value and which field it belongs to. In Phase 3 (Step 3.4), the skill will apply these vocabulary updates to `validator.py` in the exchange repo clone before committing.

This same handling applies to any controlled vocabulary field (platforms, clients, runtimes, transports, auth methods) where the user needs a value that doesn't yet exist.

### Step 2.4-SKILL — Skill-specific fields (only for `skill` type)

When the user selects `skill` as their type, run auto-detection for skill-specific fields before asking questions.

#### Detection scan

Run these checks against the user's repo:

```bash
# Detect compatible platforms
ls SKILL.md .cursor/rules .cursorrules .github/copilot-instructions.md .windsurfrules GEMINI.md AGENTS.md codex.json 2>/dev/null
ls -d .gemini .cline .clinerules 2>/dev/null
```

#### Platform detection rules

| Signal | Platform |
|--------|----------|
| `SKILL.md` at repo root | Claude Code |
| `.cursor/rules` or `.cursorrules` file | Cursor |
| `.github/copilot-instructions.md` | GitHub Copilot |
| `.windsurfrules` file | Windsurf |
| `GEMINI.md` or `.gemini/` directory | Gemini CLI |
| `AGENTS.md` or `codex.json` | Codex |
| `.clinerules` or `.cline/` directory | Cline |

#### Invocation detection

Check for the invocation/trigger name:

```bash
# Check SKILL.md frontmatter for name field
head -20 SKILL.md 2>/dev/null | grep -i "^name:"
```

| Signal | Inference |
|--------|-----------|
| SKILL.md frontmatter has `name:` field | Use that as invocation (prepend `/` if not present) |
| README mentions a slash command (e.g., `/something`) | Extract it |
| No signal | Ask user directly |

#### Validate platforms

Validate detected platforms against the `Skill.compatible_platforms` Literal values parsed from the validator in Phase 2's initial fetch.

#### Present findings

After running detection, present findings for user confirmation:

> "Based on your repo, I've detected:"
> - **Compatible platforms:** <values> (<reasons>)
> - **Invocation:** <value> (<reason>)
>
> "Does this look right? Anything to add or correct?"

For any field that couldn't be detected, ask the user directly. For platforms, show the full list of valid options from the fetched vocabulary.

Note: Skills do NOT have a `framework` field — skip the framework detection from Step 2.4 for skill types.

### Step 2.4-MCP — MCP-specific fields (only for `mcp-server` type)

When the user selects `mcp-server` as their type, run auto-detection for all MCP-specific fields before asking questions.

#### Detection scan

Run these checks against the user's repo:

```bash
# Detect runtime
ls package.json tsconfig.json bun.lockb bunfig.toml pyproject.toml setup.py requirements.txt Pipfile Cargo.toml go.mod 2>/dev/null

# Detect transport patterns
grep -r "StdioServerTransport\|stdio_server\|SSEServerTransport\|StreamableHTTPServerTransport" --include="*.py" --include="*.ts" --include="*.js" -l 2>/dev/null

# Detect MCP tool/resource/prompt definitions
grep -rn "server\.tool\|@server\.tool\|server\.resource\|@server\.resource\|server\.prompt\|@server\.prompt" --include="*.py" --include="*.ts" --include="*.js" 2>/dev/null

# Check README for client mentions
grep -i "claude desktop\|claude code\|cursor\|windsurf\|copilot\|cline\|continue" README* readme* 2>/dev/null
```

#### Detection rules

**Runtime** (high confidence):
- `package.json` or `tsconfig.json` → node
- `bun.lockb` or `bunfig.toml` → bun
- `pyproject.toml`, `setup.py`, `requirements.txt`, `Pipfile` → python
- `Cargo.toml` → rust
- `go.mod` → go
- None of the above → binary

**Transport** (medium-high confidence):
- `StdioServerTransport` or `stdio_server` found → stdio
- `SSEServerTransport` or `StreamableHTTPServerTransport` found → http
- Both patterns found, or README mentions both modes → both
- If nothing detected but README has Claude Desktop JSON config example → stdio

**Auth method** (medium confidence):
- No auth middleware or env vars for keys/tokens on the MCP server itself → none
- OAuth patterns, `/authorize` endpoint, token refresh logic → oauth2
- Bearer token validation on MCP endpoint → token
- README mentions "set your API key" for the MCP connection → api-key
- If only upstream-service API keys found → ask user to clarify

**Compatible clients** (from README + transport inference):
- README contains `mcpServers` JSON block → include "Claude Desktop"
- README mentions specific clients by name → include those
- Transport is stdio → suggest Claude Desktop, Claude Code, Cursor
- Transport is http → suggest Claude Desktop, Claude Code

**Tools/resources/prompts** (high confidence):
- Parse `server.tool("name", "description", ...)` patterns (TypeScript)
- Parse `@server.tool()` decorated functions (Python) — name from function name, description from docstring
- Parse `server.resource(...)` / `@server.resource(...)` similarly
- Parse `server.prompt(...)` / `@server.prompt(...)` similarly

#### Present findings

After running detection, present all findings in a single summary for user confirmation:

> "Based on your repo, I've detected:"
> - **Runtime:** <value> (<reason>)
> - **Transport:** <value> (<reason>)
> - **Auth method:** <value> (<reason>)
> - **Compatible clients:** <values>
> - **Tools (<count>):** <name1>, <name2>, ...
> - **Resources (<count>):** <names or "none detected">
> - **Prompts (<count>):** <names or "none detected">
>
> "Does this look right? Anything to add or correct?"

For any field that couldn't be detected, ask the user directly. Validate all values against the controlled vocabularies parsed from the validator in Phase 2's initial fetch (transports from `MCPServer.transport`, runtimes from `MCPServer.runtime`, auth methods from `MCPServer.auth_method`, clients from `MCPServer.compatible_clients`).

### Step 2.5 — Playbook-specific: `agents_used` chain

Only if the type is a playbook listing. Ask:
> "Walk me through the agents or steps in this playbook workflow. For each one, I'll need:
> 1. **Name** — what to call this agent/step
> 2. **Role** — what it does in the chain (one sentence)
> 3. **Type** — is it listed on the CyberAgents Exchange (`exchange`), on GitHub (`github`), or just an informational/manual step (`info`)?
> 4. **Ref** — the exchange slug, GitHub URL, or leave blank for info steps"

Collect each agent one at a time. After each, ask "Any more agents in the chain, or is that the complete workflow?"

Present the full chain for review:
> "Here's your agent chain:"
> 1. **<name>** (<type>) — <role>
> 2. ...

### Step 2.6 — Body content

Generate a markdown body for the listing. Read the README and produce 2-3 short sections:
- A brief intro paragraph
- "## What it does" — capabilities overview
- "## How it works" — brief technical approach

Present this to the user:
> "Here's the body content I've drafted for your listing page. You can edit this, approve it as-is, or replace it entirely:"

### Step 2.7 — Assemble and review

Assemble the complete listing markdown file using the fetched template as the structural guide. The output should look like:

For agents:
```yaml
---
name: "<name>"
author: "<author>"
github_url: "<url>"
description: "<description>"
license: "<spdx-id>"
type: "<type>"
tier: "unreviewed"
tags: [<tags>]
framework: "<framework>"
integrations: [<integrations>]
date_added: <YYYY-MM-DD>
---

<body content>
```

For skills:
```yaml
---
name: "<name>"
author: "<author>"
github_url: "<url>"
description: "<description>"
license: "<spdx-id>"
tier: "unreviewed"
tags: [<tags>]
integrations: [<integrations>]
date_added: <YYYY-MM-DD>
compatible_platforms: [<platforms>]
invocation: "<invocation>"
---

<body content>
```

For MCP servers:
```yaml
---
name: "<name>"
author: "<author>"
github_url: "<url>"
description: "<description>"
license: "<spdx-id>"
tier: "unreviewed"
tags: [<tags>]
integrations: [<integrations>]
date_added: <YYYY-MM-DD>
transport: "<transport>"
runtime: "<runtime>"
auth_method: "<auth-method>"
compatible_clients: [<clients>]
tools_exposed:
  - name: "<name>"
    description: "<description>"
resources_exposed: []
prompts_exposed: []
---

<body content>
```

For playbooks:
```yaml
---
name: "<name>"
author: "<author>"
github_url: "<url>"
description: "<description>"
license: "<spdx-id>"
tier: "unreviewed"
tags: [<tags>]
integrations: [<integrations>]
agents_used:
  - name: "<name>"
    role: "<role>"
    type: "<type>"
    ref: "<ref>"
date_added: <YYYY-MM-DD>
---

<body content>
```

Show the complete file to the user:
> "Here's your complete listing file. Review it carefully — this is what will appear on the Exchange:"

Ask for any final changes. Iterate until the user approves.

### Step 2.8 — Commit listing to agent repo

Read `references/submission-checklist.md` and verify the listing passes all relevant checks before proceeding.

Determine the filename: generate a slug from the `name` field (lowercase, replace spaces and special characters with hyphens, strip leading/trailing hyphens).

Write the listing file to the repo root (e.g., `<slug>.md`).

**Confirm before git action:**
> "Ready to commit `<slug>.md` to your repo and push to GitHub. Proceed? (This will commit and push to your `<branch>` branch)"

```bash
git add <slug>.md
git commit -m "Add CyberAgents Exchange listing"
git push
```

---

## Phase 3: Submit to the Exchange

### Step 3.1 — Verify GitHub CLI authentication

```bash
gh auth status
```

If not authenticated, tell the user:
> "The GitHub CLI isn't authenticated. I need it to clone the exchange repo and create a PR. Let's log in:"
> ```
> gh auth login
> ```
> "This will open a browser for authentication. Follow the prompts to log in with your **personal** GitHub account (not an EMU/corporate account)."

### Step 3.2 — Verify exchange repo access

Test access to the content repository:
```bash
gh api repos/tenable-cyberagents-exchange/exchange-founders-prelaunch-agents --jq '.full_name'
```

**If this fails (404 or 403):**

First, check if this is an EMU account issue:
```bash
gh api user --jq '.login'
```

Inspect the returned username. If it contains an **underscore** separating an org prefix from a username (e.g., `tenable_jbuchanan`), it's an EMU account. Hyphens are normal — do NOT flag those. Tell the user:

> "You're currently authenticated as `<username>`, which appears to be an Enterprise Managed User (EMU) account. EMU accounts cannot access repositories outside their enterprise. You need to switch to a personal GitHub account.
>
> Do you have a personal GitHub account? If so, what's the username? I can help you switch."

If they provide a personal username, run:
```bash
gh auth login
```
Walk them through the browser auth flow. After login, re-run the access check.

If they don't have a personal account, direct them to github.com/signup to create one.

**If not an EMU issue** (username looks normal but access denied):

> "You don't currently have access to the exchange content repository (`tenable-cyberagents-exchange/exchange-founders-prelaunch-agents`). This is a private repo — you need to be added as a collaborator.
>
> Contact one of these people to request access:
> - **Justin Buchanan** — [@jtbuchanan-tenb](https://github.com/jtbuchanan-tenb)
> - **Patrick Ramseier** — [@pramseier-tenb](https://github.com/pramseier-tenb)
> - **DJ Zito**
>
> Once you've been added and accepted the invitation, let me know and I'll continue."

Pause and wait for the user to confirm before continuing.

### Step 3.3 — Clone the exchange repo and prepare branch

Clone the exchange repo directly to a temporary directory (no forking required — contributors have push access):
```bash
CLONE_DIR=$(mktemp -d)
gh repo clone tenable-cyberagents-exchange/exchange-founders-prelaunch-agents "$CLONE_DIR"
cd "$CLONE_DIR"
```

Create a branch for the submission:
```bash
git checkout -b add-<slug>
```

Where `<slug>` is the same slug used for the listing filename.

### Step 3.4 — Place listing file

Determine the target directory:
- If type is `agent` or `tool` → place in `agents/`
- If type is `skill` → place in `skills/`
- If type is `mcp-server` → place in `mcp-servers/`
- If type is playbook → place in `playbooks/`

Check for filename conflicts:
```bash
ls agents/<slug>.md mcp-servers/<slug>.md playbooks/<slug>.md 2>/dev/null
```

If a conflict exists, inform the user and ask them to choose a different name.

Copy the listing file:
```bash
cp <path-to-listing-in-agent-repo>/<slug>.md <target-directory>/<slug>.md
```

### Step 3.5 — Update validator vocabulary (if needed)

If the user requested new vocabulary values during Phase 2, apply them now to `validator.py` in the exchange repo clone:

1. Open `validator.py` and locate the appropriate `Literal[...]` type annotation for each new value:
   - Integrations → `Entry.integrations` field
   - Platforms → `Skill.compatible_platforms` field
   - Clients → `MCPServer.compatible_clients` field
   - Transports → `MCPServer.transport` field
   - Runtimes → `MCPServer.runtime` field
   - Auth methods → `MCPServer.auth_method` field

2. Insert the new value into the Literal list in alphabetical order, maintaining the existing formatting (one value per line, with trailing comma).

3. Inform the user:
   > "I've added `<value>` to the `<field>` vocabulary in `validator.py`. The maintainers will review this alongside your listing."

If no vocabulary updates are needed, skip this step.

### Step 3.6 — Commit, push, and create PR

**Confirm before git action:**
> "Ready to submit your listing to the CyberAgents Exchange. This will:"
> 1. Commit `<target-directory>/<slug>.md` to your branch
> 2. Push the branch to GitHub
> 3. Open a pull request to the exchange repo
>
> "Proceed?"

If vocabulary updates were made, include both files in the commit:
```bash
git add <target-directory>/<slug>.md validator.py
git commit -m "Add listing: <Agent Name>"
git push -u origin add-<slug>
```

Otherwise:
```bash
git add <target-directory>/<slug>.md
git commit -m "Add listing: <Agent Name>"
git push -u origin add-<slug>
```

Create the pull request. If vocabulary updates were included, mention them in the PR body:
```bash
gh pr create \
  --repo tenable-cyberagents-exchange/exchange-founders-prelaunch-agents \
  --title "Add listing: <Agent Name>" \
  --body "## New Listing: <Agent Name>

**Repository:** <github_url>
**Type:** <type>
**Description:** <description>

### Checklist
- [x] Agent/playbook code is in a personal GitHub repo
- [x] Repo has a README
- [x] Repo has an open source license (<license>)
- [x] Listing file passes schema validation
- [x] Listing placed in correct directory (<target-directory>/)
- [x] Filename is a valid slug (<slug>.md)

### Vocabulary Updates (if applicable)
- Added `<value>` to `<field>` in `validator.py`"
```

Omit the "Vocabulary Updates" section from the PR body if no vocabulary changes were made.

### Step 3.7 — Report success

After the PR is created, tell the user:
> "Your listing has been submitted! Here's your pull request:"
> **<PR URL>**
>
> "A maintainer will review and merge it. Once merged, your listing will appear on the CyberAgents Exchange on the next deploy.
>
> If you need to make changes, you can push additional commits to your `add-<slug>` branch and the PR will update automatically."

Clean up the temporary clone:
```bash
rm -rf "$CLONE_DIR"
```
