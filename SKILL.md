---
name: cyberagents-exchange-submit
description: "Submit an agent, skill, MCP server, or playbook to the Tenable CyberAgents Exchange. Use when a user wants to list their cybersecurity AI agent, skill, or workflow on the Tenable Exchange (exchange.tenable.com), verify their repo meets requirements, generate listing metadata, and open a pull request to the exchange content repository. Triggers on: submit to exchange, Tenable CyberAgents Exchange, Exchange Tenable, list my agent, list my skill, publish to exchange."
---

# CyberAgents Exchange Submission

Guide the user through submitting their agent, skill, MCP server, or playbook to the Tenable CyberAgents Exchange. This is a multi-phase process: validate their repo, generate a listing file, and submit a PR to the exchange content repository.

The exchange content repo is: `tenable/cyberagents-exchange`

Always confirm with the user before any git commit, push, or PR creation.

---

## Phase 0: Version Check

**This phase runs first, before anything else.** Compare the user's installed version of this skill against the latest version in the public repository.

### Step 0.1 — Determine the installed skill location

Find where this skill file lives on the user's machine:

```bash
# Check common install locations
find ~/.claude -name "cyberagents-exchange-submit*" -o -name "SKILL.md" 2>/dev/null | head -20
```

Identify the directory containing the installed skill. This is the directory that will need updating if the version is outdated.

### Step 0.2 — Compare against upstream

Fetch the latest commit SHA for SKILL.md from the public repo:

```bash
gh api repos/jtbuchanan-tenb/cyberagent-exchange-submission-builder/commits?path=SKILL.md\&per_page=1 --jq '.[0].sha'
```

Get the local HEAD commit SHA for the installed skill's repo (if it's a git clone):

```bash
git -C <skill_directory> rev-parse HEAD 2>/dev/null
```

If the local directory is not a git repo (e.g., the skill was copied manually), fall back to content comparison:

```bash
# Fetch the upstream SKILL.md content
gh api repos/jtbuchanan-tenb/cyberagent-exchange-submission-builder/contents/SKILL.md --jq '.content' | base64 -d > /tmp/upstream-skill.md

# Compare with local
diff -q <local_skill_path> /tmp/upstream-skill.md
```

### Step 0.3 — Handle outdated version

**If the versions match** (same commit SHA, or diff shows no differences), continue silently to Phase 1.

**If the versions differ**, inform the user:

> "Your installed version of the CyberAgents Exchange submission skill is outdated. The skill changes frequently and you need the latest version to ensure your submission meets current requirements."
>
> "Would you like me to update it now?"

**If the user agrees to update:**

If the skill directory is a git clone:
```bash
git -C <skill_directory> pull origin main
```

If the skill was installed via a different mechanism (e.g., copied or symlinked), replace it with the latest:
```bash
gh api repos/jtbuchanan-tenb/cyberagent-exchange-submission-builder/contents/SKILL.md --jq '.content' | base64 -d > <local_skill_path>
```

After updating, inform the user:
> "Updated successfully. Please re-run the skill to use the latest version."

**Stop here — do not continue with the submission process.** The user needs to re-invoke the skill so the updated instructions are loaded.

**If the user declines the update:**

> "This skill changes frequently and an outdated version may produce submissions that don't meet current Exchange requirements. You'll need to update before continuing."

**Do not proceed with the submission.** The skill cannot continue with an outdated version.

---

## Phase 1: Locate & Validate the Agent Repo

### Step 1.0 — Contribution Agreement

Before anything else, ask the user to review and accept the contribution agreement:

> "Before we begin, you'll need to review and accept the CyberAgents Exchange Contribution Agreement:"
>
> https://github.com/tenable/cyberagents-exchange/blob/main/docs/CyberAgents_Contribution_Agreement
>
> "Have you reviewed and do you accept the CyberAgents Exchange Contribution Agreement?"

The user must explicitly confirm acceptance (e.g., "yes", "I accept", "agreed"). If they say no, decline, or express uncertainty:

> "The Contribution Agreement must be accepted before submitting to the Exchange. Feel free to come back when you're ready."

**Do not proceed with any further steps until the user accepts.**

**When the user accepts**, immediately record the current UTC date and time as the `contribution_agreement_date`. Store this as an ISO 8601 timestamp (e.g., `2026-07-09T14:30:00Z`). Use the current time at the moment of acceptance:

```bash
date -u +%Y-%m-%dT%H:%M:%SZ
```

This timestamp will be included in the listing frontmatter.

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

### Step 1.6 — Archive policy check

Scan the repo for compressed archive files:

```bash
find . -maxdepth 3 -type f \( -name "*.zip" -o -name "*.tar.gz" -o -name "*.tgz" -o -name "*.tar.bz2" -o -name "*.skill" -o -name "*.rar" -o -name "*.7z" \) -not -path "*/.git/*" -not -path "*/node_modules/*" -not -path "*/.venv/*" 2>/dev/null
```

**If archives are found**, determine whether they are the sole delivery path for the skill/agent content:

1. **Archive-only structure** (BLOCKING): If the primary content (e.g., SKILL.md, source code, configuration) exists ONLY inside an archive and is not also present unpacked in the repo tree, this is a submission-blocking issue:

> "I found archive file(s) in your repo: `<list>`
>
> The CyberAgents Exchange requires that all skill/agent content be available as unpacked files in the repository. Archives cannot be the only path to the content — users and reviewers must be able to read the files directly on GitHub without downloading and extracting anything.
>
> You need to extract the contents of your archive(s) to the repo root (or appropriate subdirectories) so the files are directly accessible. Would you like help restructuring?"

**Do not proceed past Phase 1 until the archive-only structure is resolved.**

2. **Archives as supplements** (non-blocking, advisory): If the primary content also exists unpacked (e.g., SKILL.md is at root AND a `.skill` zip exists as a convenience download), inform the user:

> "I found archive file(s) in your repo: `<list>`. Since the primary content is also available unpacked, this is fine — the archives can stay as supplementary downloads. Note that the reviewer will inspect archive contents during review."

Continue to Phase 2.

3. **No archives found**: Continue silently.

---

## Phase 2: Interview & Generate Listing

### First: Fetch live data from the exchange repo

Before starting the interview, fetch the validator and contributing checklist:

```bash
# Fetch the validator which contains all controlled vocabularies as Literal types
gh api repos/tenable/cyberagents-exchange/contents/validator.py --jq '.content' | base64 -d

# Fetch the contributing checklist which defines submission requirements
gh api repos/tenable/cyberagents-exchange/contents/docs/contributing_checklist.md --jq '.content' | base64 -d
```

Parse the Pydantic models in `validator.py` to extract valid values from `Literal[...]` type annotations:
- **Integrations** — from `Entry.integrations` field's Literal values
- **Tiers** — from `Entry.tier` field's Literal values
- **Platforms** — from `Skill.compatible_platforms` field's Literal values
- **Transports** — from `MCPServer.transport` field's Literal values
- **Runtimes** — from `MCPServer.runtime` field's Literal values
- **Auth methods** — from `MCPServer.auth_method` field's Literal values
- **Clients** — from `MCPServer.compatible_clients` field's Literal values

Store these results for validation throughout Phase 2.

Review the fetched contributing checklist and note the Tier 1 requirements. As you guide the user through the interview, build the submission so it satisfies every Tier 1 item: the listing must be a single file in the directory matching its type, with a valid slug filename containing no leftover template placeholders; frontmatter must pass validator.py with all controlled-vocabulary fields using valid values; the body must be substantive (not a stub); and the linked repository must have a README covering what it does, prerequisites, how to run, outputs, and known limitations, plus a detectable open-source LICENSE file. This checklist is the authoritative standard the reviewer skill applies, so aligning to it up front means fewer review round-trips.

You'll also need the appropriate template later — fetch it after the user selects their type:

```bash
# For agents/tools:
gh api repos/tenable/cyberagents-exchange/contents/templates/agent-template.md --jq '.content' | base64 -d

# For skills:
gh api repos/tenable/cyberagents-exchange/contents/templates/skill-template.md --jq '.content' | base64 -d

# For MCP servers:
gh api repos/tenable/cyberagents-exchange/contents/templates/mcp-server-template.md --jq '.content' | base64 -d

# For playbooks:
gh api repos/tenable/cyberagents-exchange/contents/templates/playbook-template.md --jq '.content' | base64 -d
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

#### Copyright holder check

After identifying the license, silently check for signals that the user may be a Tenable employee:

```bash
# Check git user email
git config user.email 2>/dev/null
# Check GitHub username from remote
git remote get-url origin 2>/dev/null
```

Look for any of these signals:
- Git email contains `@tenable.com`
- GitHub username contains `tenable` or `tenb` (e.g., `jtbuchanan-tenb`, `tenable_jdoe`)
- The repo owner from Step 1.2 contains `tenable` or `tenb`

**If any signal is detected**, ask:
> "Are you a Tenable employee?"

If the user says they are not a Tenable employee, continue normally without further action.

If they confirm yes:
> "Because you're a Tenable employee, the copyright in your LICENSE file needs to belong to **Tenable, Inc.** — not your individual name."

Then read the LICENSE file and check whether the copyright line already names `Tenable, Inc.` (case-insensitive match on "Tenable"). If it does, continue to the next step.

If the copyright line does **not** name Tenable, Inc., offer to fix it:
> "Your LICENSE currently has: `<current copyright line>`. I'll update it to:"
> ```
> Copyright (c) <year> Tenable, Inc.
> ```
> "Want me to make that change?"

If they accept, replace the copyright line with `Copyright (c) <year> Tenable, Inc.` and commit the change. Then re-read the LICENSE to confirm the fix before continuing.

**This is a hard gate.** Do NOT proceed to the next step until the LICENSE copyright line names `Tenable, Inc.`. If the user declines the update, explain that Tenable employees must assign copyright to Tenable, Inc. and ask them to fix it manually before continuing. Do not proceed past this check until the copyright is correct.

**If no signal is detected**, skip this check silently.

**If no license file is found:**

Tell the user:
> "The CyberAgents Exchange requires all listed projects to have an open source license. Your repo doesn't appear to have a LICENSE file.
>
> I recommend the **MIT License** — it's the most permissive and widely used license for open source tools. It lets anyone use, modify, and distribute your work with minimal restrictions.
>
> Would you like me to create a LICENSE file with the MIT license? Or would you prefer a different license? (Options: MIT, Apache-2.0)"

If the user agrees to MIT (or picks another), create the LICENSE file:
- Use the user's name (from their GitHub profile or ask them) and the current year
- Write the standard license text to `LICENSE` in the repo root
- **Confirm before writing**

If the repo has a `package.json` without a `license` field, suggest:
> "I notice your package.json doesn't have a `license` field. For ecosystem consistency, I'd recommend adding `\"license\": \"MIT\"` to it. Want me to do that?"

### Step 2.2 — Type selection

Ask the user:
> "What type of listing are you submitting?"

Present the options:

> - **agent** — A standalone autonomous AI system, CLI tool, library, or utility
> - **skill** — An agent skill file (SKILL.md) that extends AI coding assistants (goes to `skills/` directory)
> - **mcp-server** — A Model Context Protocol server exposing data sources or actions
> - **playbook** — A multi-agent workflow or orchestration

After type selection, fetch the appropriate template (agent-template.md for agents, skill-template.md for skills, mcp-server-template.md for MCP servers, playbook-template.md or n8n-playbook-template.md for playbooks).

### Step 2.2a — Subagent detection and type correction

**Before proceeding with the selected type**, scan the repository for Claude Code subagent definitions:

```bash
# Check for .claude/agents/ subagent definitions
find . -path "*/.claude/agents/*.md" -not -path "*/.git/*" 2>/dev/null

# Also check for .claude/agents/ directory existence
ls -la .claude/agents/ 2>/dev/null
```

**If `.claude/agents/*.md` files are found AND the user selected "agent" type:**

This is a Claude Code subagent definition, not a standalone agent. On the CyberAgents Exchange, subagent definitions are listed as **skills** because they extend an AI coding assistant (Claude Code) rather than running independently. They must be submitted with a `SKILL.md` at the repo root.

Inform the user:

> "I found a Claude Code subagent definition in your repo (`.claude/agents/<name>.md`). On the CyberAgents Exchange, subagent definitions like this are listed as **skills**, not agents — because they extend Claude Code rather than running as standalone programs.
>
> To submit this to the Exchange, your repo needs to be restructured as a Claude Code skill with a `SKILL.md` file at the root. This is what allows Claude Code users to install and invoke your subagent as a skill.
>
> I'll switch your listing type to **skill** and help you create the required `SKILL.md`. Sound good?"

**If the user agrees**, change the type to `skill` and proceed to help them create a `SKILL.md`:

1. Read the existing subagent definition to understand its purpose and instructions:
```bash
cat .claude/agents/<name>.md
```

2. Generate a `SKILL.md` that wraps the subagent's functionality. The SKILL.md should:
   - Have proper YAML frontmatter with `name:` and `description:` fields
   - Contain the skill instructions (which can reference or delegate to the subagent definition)
   - Be placed at the repo root

3. Present the proposed `SKILL.md` to the user for review and approval before writing it.

4. After creating SKILL.md, confirm before committing:
   > "I've created `SKILL.md` at the repo root. Ready to commit this? (The subagent definition in `.claude/agents/` stays as-is — SKILL.md provides the Exchange-compatible entry point.)"

**If the user disagrees or insists on "agent" type**, explain:

> "The Exchange requires that Claude Code subagent definitions be submitted as skills. A standalone agent listing is for programs that run independently (their own process, CLI, or service). Since yours requires Claude Code to run and is installed into a user's `.claude/agents/` directory, it falls under the skill category.
>
> Would you like me to help restructure it as a skill, or would you prefer to stop here and revisit later?"

**Do not proceed with an "agent" type listing if the repo's primary artifact is a `.claude/agents/*.md` subagent definition.**

**If no subagent definitions are found**, or the user selected a type other than "agent", continue normally.

### Step 2.2b — Playbook subtype selection (only for playbooks)

If the user selected "playbook" in Step 2.2, ask:
> "What type of playbook is this?"
> - **standard** — A multi-agent orchestration you've designed (requires listing the agents in the chain)
> - **sponsored** — A vendor-partnered playbook with co-branding (requires a logo URL and may include vendor-type agent references)
> - **n8n** — An n8n workflow with a `workflow.json` file in your repo (generates a visual workflow diagram)

Store the selection as `playbook_subtype`. This determines which template to fetch and which fields to collect.

Fetch the appropriate template:
```bash
# For standard/sponsored:
gh api repos/tenable/cyberagents-exchange/contents/templates/playbook-template.md --jq '.content' | base64 -d

# For n8n:
gh api repos/tenable/cyberagents-exchange/contents/templates/n8n-playbook-template.md --jq '.content' | base64 -d
```

### Step 2.3 — Auto-detected fields

Determine these fields automatically and present them for confirmation:

| Field | How to detect |
|-------|---------------|
| `name` | Parse the first `# heading` from README.md. Fallback: repo name with hyphens replaced by spaces and title-cased. |
| `github_url` | The HTTPS URL from git remote (e.g., `https://github.com/owner/repo`) |
| `author` | The owner from the GitHub remote URL |
| `license` | The SPDX identifier from Step 2.1 |
| `date_added` | Today's date in YYYY-MM-DD format |
| `tier` | Always `"contributed"` — hardcoded, do not ask |
| `contribution_agreement_date` | The ISO 8601 timestamp captured in Step 1.0 when the user accepted the Contribution Agreement |

Present these to the user:
> "Here's what I've detected from your repo. Please confirm or correct:"
> - **Name:** <detected>
> - **GitHub URL:** <detected>
> - **Author:** <detected>
> - **License:** <detected>
> - **Date added:** <today>
> - **Contribution Agreement accepted:** <timestamp from Step 1.0>
> - **Tier:** contributed (all new submissions start here)

### Step 2.4 — Generative-assisted fields

For these fields, read the repo context (README, code structure, imports, dependencies) and propose values, but let the user guide the final answer.

**`description`** — Read the README and propose a one-line summary (under 120 characters). Ask:
> "Here's my suggested one-line description: '<proposed>'. Does this capture what your agent does, or would you like to revise it?"

**`tags`** — Suggest 3-7 tags based on README content, code keywords, and the domain. Tags should be lowercase, hyphenated. Ask:
> "Suggested tags: `[<tag1>, <tag2>, ...]`. Want to add, remove, or change any?"

**`integrations`** — Search the README and code for references to known platforms. Cross-reference with the integrations list parsed from the validator. Suggest matches and show the full list of valid options:
> "Based on your code, I think these integrations apply: `[<suggested>]`. Here's the full list of valid integrations: `[<all valid>]`. Want to add or remove any?"

If the user types an integration that's not in the list, fuzzy-match against valid values (e.g., "crowdstrike" → "CrowdStrike", "sentinelone" → "SentinelOne") and suggest the correction.

If no fuzzy match is found and the user confirms the value is correct (e.g., a new vendor not yet in the vocabulary), inform them:
> "That value isn't in the current controlled vocabulary. I can include an update to `validator.py` in your submission PR to add it — the maintainers will review the vocabulary addition alongside your listing. Want me to do that?"

If the user agrees, track the new value and which field it belongs to. In Phase 3 (Step 3.4), the skill will apply these vocabulary updates to `validator.py` in the exchange repo clone before committing.

This same handling applies to any controlled vocabulary field (platforms, clients, runtimes, transports, auth methods) where the user needs a value that doesn't yet exist.

### Step 2.4b — Tenable Hexa MCP detection

**Only ask this if the user's `integrations` list includes `"Tenable"`.** If Tenable is not among the integrations, skip this step entirely and omit `works_with_tenable_hexa_mcp` from the listing (it is optional — only include it when `true`).

If `"Tenable"` is in the integrations, ask:

> "Your submission integrates with Tenable. Does it use the **Tenable Hexa MCP** server for its Tenable integration?
>
> - If it connects to Tenable via the [Tenable Hexa MCP](https://github.com/tenable/hexa-mcp), I'll mark it as `works_with_tenable_hexa_mcp: true`.
> - If it uses other Tenable APIs directly (Tenable VM API, Security Center API, Nessus API, etc.), I'll leave this as `false`.
>
> Which does yours use?"

Additionally, scan the repository for evidence:

```bash
# Look for Hexa MCP references
grep -ri "hexa.mcp\|hexa-mcp\|tenable.*hexa\|tenable/hexa" README* *.md .claude/* 2>/dev/null
grep -ri "hexa.mcp\|hexa-mcp" --include="*.py" --include="*.ts" --include="*.js" --include="*.json" --include="*.yaml" --include="*.yml" . 2>/dev/null | grep -v node_modules | grep -v .venv
```

If repo evidence supports Hexa MCP usage AND the user confirms, set `works_with_tenable_hexa_mcp: true` **and** add `"Tenable Hexa AI MCP"` to the `integrations` list (if not already present). The validator enforces that `works_with_tenable_hexa_mcp: true` requires the `"Tenable Hexa AI MCP"` integration — they must always appear together. If the user says they use other Tenable APIs (or the evidence doesn't support Hexa MCP), omit `works_with_tenable_hexa_mcp` from the listing entirely — do not include it as `false`.

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

### Step 2.4-SKILL-STRUCT — Validate skill structure (only for `skill` type)

After the user confirms their `compatible_platforms` selection, validate that the repo actually contains the required files for each declared platform. This prevents reviewer rejections for missing platform signals.

#### SKILL.md structural validation

If any declared platform requires `SKILL.md` (Claude Code, Claude Desktop, Cowork), validate its internal structure:

```bash
# Check frontmatter exists and has required fields
head -30 SKILL.md 2>/dev/null
```

Verify:
1. File begins with `---` (YAML frontmatter delimiter)
2. Contains a `name:` field (non-empty)
3. Contains a `description:` field (non-empty)
4. Closes with `---`
5. Body below frontmatter is substantive (not a placeholder or empty)

If SKILL.md is missing or malformed:
> "Your SKILL.md needs valid frontmatter for Claude Code/Desktop to load it. It must have this structure:"
> ```
> ---
> name: your-skill-name
> description: "One-line description of what it does"
> ---
>
> # Skill content here...
> ```
> "Would you like me to help fix this?"

#### Reference file completeness

If the SKILL.md body references any files (e.g., `references/checklist.md`, `templates/something.md`):

```bash
# Extract file references from SKILL.md
grep -oE '(references|templates)/[a-zA-Z0-9_./-]+' SKILL.md 2>/dev/null
```

Verify each referenced file actually exists in the repo. If any are missing:
> "Your SKILL.md references these files that don't exist in the repo:"
> - `<missing-file>`
>
> "The reviewer will flag these as broken references. Would you like to create them or remove the references?"

#### Per-platform congruence check

For each platform the user declared in `compatible_platforms`, verify the required structure exists:

| Declared Platform | Required Structure | Check |
|-------------------|--------------------|-------|
| Claude Code | `SKILL.md` at root with `name:`/`description:` frontmatter | Already validated above |
| Claude Desktop | `SKILL.md` at root with `name:`/`description:` frontmatter | Same as Claude Code |
| Cowork | `SKILL.md` at root with `name:`/`description:` frontmatter | Same as Claude Code |
| Cursor | `.cursor/rules/` dir OR `.cursorrules` file at root | `ls .cursor/rules .cursorrules 2>/dev/null` |
| Codex | `AGENTS.md` at repo root | `ls AGENTS.md 2>/dev/null` |
| Cline | `.cline/skills/<name>/SKILL.md` OR `.clinerules` at root | `find .cline -name "SKILL.md" 2>/dev/null; ls .clinerules 2>/dev/null` |
| Gemini CLI | `.gemini/skills/<name>/SKILL.md` OR `GEMINI.md` at root | `find .gemini -name "SKILL.md" 2>/dev/null; ls GEMINI.md 2>/dev/null` |
| GitHub Copilot | `.github/skills/<name>/SKILL.md` OR `.github/copilot-instructions.md` | `find .github/skills -name "SKILL.md" 2>/dev/null; ls .github/copilot-instructions.md 2>/dev/null` |
| Windsurf | `.windsurf/skills/<name>/SKILL.md` OR `.windsurfrules` at root | `find .windsurf -name "SKILL.md" 2>/dev/null; ls .windsurfrules 2>/dev/null` |

For each platform that has no matching structure:

> "You declared `<platform>` as compatible, but I don't see the required file(s) for that platform:"
> - **<platform>** needs: `<required structure>`
>
> "Options:"
> 1. I can help create the platform-specific file(s)
> 2. Remove `<platform>` from your compatible_platforms list
>
> "Which would you prefer?"

If the user chooses to create files, guide them through it. For platforms that use SKILL.md variants in subdirectories (Cline, Gemini CLI, GitHub Copilot, Windsurf), the content can often be symlinked or copied from the root SKILL.md — offer this as a convenience.

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

### Step 2.4-PLAYBOOK-SPONSORED — Sponsored playbook fields (only for `sponsored` subtype)

When the user selected `sponsored` as their playbook subtype:

#### Logo URL

Ask:
> "What's the URL to your company/product logo? This will appear in the co-brand banner on the Exchange.
>
> Best practice: use a raw.githubusercontent.com URL from your repo (e.g., `https://raw.githubusercontent.com/your-username/your-repo/main/logo.png`).
>
> Requirements: publicly accessible image URL, PNG or SVG recommended, will display at max 80px height."

Validate the URL is reachable:
```bash
curl -sI "<logo_url>" | head -5
```

If the URL returns a non-200 status, warn the user and ask them to fix it.

### Step 2.4-PLAYBOOK-N8N — n8n workflow fields (only for `n8n` subtype)

When the user selected `n8n` as their playbook subtype:

#### Workflow detection and diagram generation

Scan the repo for n8n workflow files:
```bash
find . -name "*.json" -not -path "*/node_modules/*" -not -path "*/.git/*" | head -20
```

For each JSON file found, check if it's an n8n workflow:
```bash
cat <file> | python3 -c "import json,sys; d=json.load(sys.stdin); assert 'nodes' in d and isinstance(d['nodes'], list) and len(d['nodes']) > 0 and 'type' in d['nodes'][0]" 2>/dev/null && echo "n8n workflow: <file>"
```

**If a workflow file is found:**

Read and parse it to generate a high-level Mermaid diagram:

1. Identify trigger node(s) — nodes with type containing "trigger" or "schedule" or "webhook"
2. Follow the `connections` object to trace the main execution path
3. Group adjacent nodes by logical purpose:
   - Multiple HTTP requests to the same domain → one step
   - Code nodes that transform data → group with their output
   - Sub-workflow calls → single "Process" step
4. Name each step descriptively based on node `name` field (clean up n8n's default names)
5. Cap at 4-7 nodes for readability
6. Choose direction: `flowchart LR` for linear flows (≤1 branch point), `flowchart TD` for workflows with parallel branches

Present the generated diagram:
> "I've analyzed your workflow and generated this visualization:"
> ```
> flowchart LR
>   A[...] --> B[...]
>   ...
> ```
> "Does this accurately represent your workflow? Feel free to edit the node labels or structure."

**If no workflow file is found:**
> "I didn't find an n8n workflow JSON file in your repo. You can either:
> 1. Point me to the file path if it's named differently
> 2. Write the Mermaid diagram manually (I'll help with the syntax)
>
> The diagram should show 4-7 high-level steps of your workflow using `flowchart LR` or `flowchart TD` syntax."

#### agents_used (optional for n8n)

After the diagram is confirmed, ask:
> "Does your workflow integrate with any agents or services listed on the CyberAgents Exchange? If so, I can add cross-references. If it's self-contained (just calling external APIs directly), we can skip this.
>
> Note: only sponsored playbooks can reference vendor-type agents."

If the user wants to add agents, collect them one at a time (same flow as Step 2.5 for standard playbooks, but limited to `exchange`, `github`, and `info` types — reject `vendor` with an explanation).

### Step 2.5 — Playbook-specific: `agents_used` chain

Only if the type is a playbook listing.

**For `standard` subtype:** (required)
Ask:
> "Walk me through the agents or steps in this playbook workflow. For each one, I'll need:
> 1. **Name** — what to call this agent/step
> 2. **Role** — what it does in the chain (one sentence)
> 3. **Type** — is it listed on the CyberAgents Exchange (`exchange`), on GitHub (`github`), or just an informational/manual step (`info`)?
> 4. **Ref** — the exchange slug, GitHub URL, or leave blank for info steps"
>
> Note: `vendor` type is only available for sponsored playbooks.

Collect each agent one at a time. After each, ask "Any more agents in the chain, or is that the complete workflow?"

**For `sponsored` subtype:** (required)
Same flow as standard, but additionally:
> "Does this playbook include any vendor-specific proprietary components? If so, I'll mark them as `vendor` type with a link to the vendor's documentation."

Allow `type: "vendor"` entries for sponsored playbooks.

**For `n8n` subtype:** Handled in Step 2.4-PLAYBOOK-N8N (optional, skip here).

Present the full chain for review:
> "Here's your agent chain:"
> 1. **<name>** (<type>) — <role>
> 2. ...

### Step 2.5b — Installation instructions audit (all types)

Before generating body content, verify that the README contains actionable installation instructions that are congruent with the repo structure and declared platforms.

```bash
# Check README for installation/setup sections
grep -inE "^#{1,3}.*(install|setup|getting started|usage|how to use|quick start)" README* readme* 2>/dev/null
```

#### For skill submissions:

Read the installation section and verify:

1. **Instructions reference actual files that exist in the repo.** If the README says "copy SKILL.md to your skills directory" — confirm `SKILL.md` exists at root. If it says "download the .skill file" — confirm the archive exists AND that unpacked content is also available (per Step 1.6 policy).

2. **Per-platform instructions exist for each declared platform.** For each platform in `compatible_platforms`, the README should explain how to install on that platform:
   - Claude Code: how to add the skill (e.g., `/install-skill` command, manual copy to `~/.claude/skills/`)
   - Cursor: where to place the rules file
   - Codex: reference to AGENTS.md
   - Cline: where to place in `.cline/skills/`
   - Gemini CLI: where to place in `.gemini/skills/`
   - GitHub Copilot: where to place in `.github/skills/`
   - Windsurf: where to place in `.windsurf/skills/`

3. **No references to non-existent files or commands.** If the README says "run `npm install`" but there's no `package.json`, flag it.

#### For all submission types:

Verify the README has:
- Clear prerequisites (dependencies, API keys needed, etc.)
- At least one concrete command or action the user can take to get started
- No references to files/paths that don't exist in the repo

**If issues are found:**

> "I found some issues with your installation instructions that the reviewer will flag:"
> - <issue 1>
> - <issue 2>
>
> "Would you like help fixing these? Clear installation instructions are required for the Exchange."

Guide the user through fixes. If the README has no installation section at all:

> "Your README doesn't have an installation/setup section. The Exchange requires clear instructions for how users can get started with your <type>. Would you like me to draft one based on your repo structure?"

**Do not proceed to body content until installation instructions are present and congruent.**

### Step 2.6 — Body content

Generate a markdown body for the listing. Read the README and produce 2-3 short sections:
- A brief intro paragraph
- "## What it does" — capabilities overview
- "## How it works" — brief technical approach

Present this to the user:
> "Here's the body content I've drafted for your listing page. You can edit this, approve it as-is, or replace it entirely:"

### Step 2.7 — Assemble and review

Assemble the complete listing markdown file using the fetched template as the structural guide. The output should look like:

**Note:** Only include `works_with_tenable_hexa_mcp: true` if Step 2.4b confirmed Hexa MCP usage. If not applicable or false, omit the field entirely.

For agents:
```yaml
---
name: "<name>"
author: "<author>"
github_url: "<url>"
description: "<description>"
license: "<spdx-id>"
tier: "contributed"
tags: [<tags>]
integrations: [<integrations>]
date_added: <YYYY-MM-DD>
contribution_agreement_date: <ISO-8601-TIMESTAMP>
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
tier: "contributed"
tags: [<tags>]
integrations: [<integrations>]
date_added: <YYYY-MM-DD>
contribution_agreement_date: <ISO-8601-TIMESTAMP>
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
tier: "contributed"
tags: [<tags>]
integrations: [<integrations>]
date_added: <YYYY-MM-DD>
contribution_agreement_date: <ISO-8601-TIMESTAMP>
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

For standard playbooks:
```yaml
---
playbook_type: "standard"
name: "<name>"
author: "<author>"
github_url: "<url>"
description: "<description>"
license: "<spdx-id>"
tier: "contributed"
tags: [<tags>]
integrations: [<integrations>]
agents_used:
  - name: "<name>"
    role: "<role>"
    type: "<type>"
    ref: "<ref>"
date_added: <YYYY-MM-DD>
contribution_agreement_date: <ISO-8601-TIMESTAMP>
---

<body content>
```

For sponsored playbooks:
```yaml
---
playbook_type: "sponsored"
name: "<name>"
author: "<author>"
github_url: "<url>"
description: "<description>"
license: "<spdx-id>"
tier: "contributed"
tags: [<tags>]
integrations: [<integrations>]
agents_used:
  - name: "<name>"
    role: "<role>"
    type: "<type>"
    ref: "<ref>"
logo: "<logo_url>"
date_added: <YYYY-MM-DD>
contribution_agreement_date: <ISO-8601-TIMESTAMP>
---

<body content>
```

For n8n playbooks:
```yaml
---
playbook_type: "n8n"
name: "<name>"
author: "<author>"
github_url: "<url>"
description: "<description>"
license: "<spdx-id>"
tier: "contributed"
tags: [<tags>]
integrations: [<integrations>]
workflow_diagram: |
  <mermaid source>
date_added: <YYYY-MM-DD>
contribution_agreement_date: <ISO-8601-TIMESTAMP>
---

<body content>
```

Show the complete file to the user:
> "Here's your complete listing file. Review it carefully — this is what will appear on the Exchange:"

Ask for any final changes. Iterate until the user approves.

### Step 2.8 — Commit listing to agent repo

Re-read the contributing checklist fetched at the start of Phase 2 (from `docs/contributing_checklist.md` in the exchange repo) and verify the listing passes all relevant Tier 1 checks before proceeding.

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

### Step 3.2 — Verify account and exchange repo access

First, check if the authenticated user is an EMU (Enterprise Managed User) account:
```bash
gh api user --jq '.login'
```

Inspect the returned username. If it contains an **underscore** separating an org prefix from a username (e.g., `tenable_jbuchanan`), it's an EMU account. Hyphens in usernames (e.g., `jtbuchanan-tenb`) are normal personal accounts — do NOT flag those.

**If EMU detected:**

> "You're currently authenticated as `<username>`, which appears to be an Enterprise Managed User (EMU) account. EMU accounts cannot fork public repositories or create pull requests outside their enterprise. You need to switch to a personal GitHub account.
>
> Do you have a personal GitHub account? If so, run:"
> ```
> gh auth login
> ```
> "Follow the prompts to log in with your personal account."

If they don't have a personal account, direct them to github.com/signup to create one. After login, re-run the EMU check.

**Do not proceed until the authenticated user is on a personal account.**

Then verify the exchange repository is reachable:
```bash
gh api repos/tenable/cyberagents-exchange --jq '.full_name'
```

This is a public repository, so it should always be reachable. If this fails, it's likely a network or `gh` auth issue — guide the user to check `gh auth status`.

### Step 3.3 — Fork the exchange repo and prepare branch

Fork the exchange repo and clone it to a temporary directory:
```bash
CLONE_DIR=$(mktemp -d)
gh repo fork tenable/cyberagents-exchange --clone --clone-dir "$CLONE_DIR"
cd "$CLONE_DIR"
```

Create a branch for the submission:
```bash
git checkout -b add-<slug>
```

Where `<slug>` is the same slug used for the listing filename.

### Step 3.4 — Place listing file

Determine the target directory:
- If type is `agent` → place in `agents/`
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

Create the pull request against the upstream repo. If vocabulary updates were included, mention them in the PR body:
```bash
gh pr create \
  --repo tenable/cyberagents-exchange \
  --title "Add listing: <Agent Name>" \
  --body "## New Listing: <Agent Name>

**Repository:** <github_url>
**Description:** <description>

### Checklist
- [x] I have reviewed and accept the [CyberAgents Exchange Contribution Agreement](https://github.com/tenable/cyberagents-exchange/blob/main/docs/CyberAgents_Contribution_Agreement)
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
