# Submission Checklist

Run through each section before submitting. Every item must pass.

## Repo Structure
- [ ] CWD (or user-specified path) is a git repository
- [ ] Git remote points to a GitHub URL
- [ ] Remote is reachable (gh repo view succeeds)
- [ ] GitHub account is personal (not EMU — username does NOT match `<org>_<name>` pattern)
- [ ] README file exists at repo root (README.md, README.rst, or README)
- [ ] LICENSE file exists at repo root (LICENSE, LICENSE.md, LICENSE.txt, or COPYING)

## Listing Frontmatter (Agent)
Required fields — all must be present and valid:
- [ ] `name` — non-empty string
- [ ] `author` — matches the GitHub username from the remote
- [ ] `github_url` — valid URL, matches the actual repo remote
- [ ] `description` — non-empty, one-line summary
- [ ] `license` — valid SPDX identifier (MIT, Apache-2.0, GPL-3.0-only, BSD-2-Clause, etc.)
- [ ] `type` — validated against live `data/types.json` (fallback: `data/categories.json`) from founders repo
- [ ] `tier` — must be `"unreviewed"` for new submissions
- [ ] `tags` — non-empty array of lowercase strings
- [ ] `framework` — non-empty string describing the implementation framework
- [ ] `integrations` — array, each value validated against live `data/integrations.json` from founders repo
- [ ] `date_added` — today's date in YYYY-MM-DD format

## Listing Frontmatter (Playbook)
Required fields — all must be present and valid:
- [ ] `name` — non-empty string
- [ ] `author` — matches the GitHub username from the remote
- [ ] `github_url` — valid URL, matches the actual repo remote
- [ ] `description` — non-empty, one-line summary
- [ ] `license` — valid SPDX identifier
- [ ] `tier` — must be `"unreviewed"`
- [ ] `tags` — non-empty array of lowercase strings
- [ ] `integrations` — array, each value validated against live `data/integrations.json`
- [ ] `agents_used` — non-empty array, each entry has:
  - `name` — non-empty string
  - `role` — non-empty string describing the agent's role in the chain
  - `type` — one of: `exchange`, `github`, `info`
  - `ref` — required for `exchange` (slug) and `github` (URL); optional for `info`
- [ ] `date_added` — today's date in YYYY-MM-DD format

## Exchange Submission
- [ ] `gh` CLI is authenticated (`gh auth status` succeeds)
- [ ] Authenticated user can access `tenable-cyberagents-exchange/exchange-founders-prelaunch-agents`
- [ ] Listing filename is a valid slug (lowercase, hyphens, no spaces, no special chars)
- [ ] No filename conflict with existing files in target directory (`agents/` or `playbooks/`)
- [ ] PR title follows format: `"Add listing: <Agent Name>"`
