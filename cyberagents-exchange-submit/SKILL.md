---
name: cyberagents-exchange-submit
description: "Submit an agent, playbook, or MCP server to the Tenable CyberAgents Exchange. Use when a user wants to list their cybersecurity AI agent or workflow on the Tenable Exchange (exchange.tenable.com), verify their repo meets requirements, generate listing metadata, and open a pull request to the exchange content repository. Triggers on: submit to exchange, Tenable CyberAgents Exchange, Exchange Tenable, list my agent, publish to exchange."
---

# CyberAgents Exchange Submission

Guide the user through submitting their agent or playbook to the Tenable CyberAgents Exchange. This is a multi-phase process: validate their repo, generate a listing file, and submit a PR to the exchange content repository.

The exchange content repo is: `tenable-cyberagents-exchange/exchange-founders-prelaunch-agents`

Always confirm with the user before any git commit, push, fork, or PR creation.

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

### Step 1.3 — Verify pushed to GitHub

Confirm the remote repo exists and is accessible:
```bash
gh repo view <owner>/<repo> --json name,url
```

If this fails, the repo either doesn't exist on GitHub yet or isn't pushed. Guide the user:
> "I can't reach your repo on GitHub. Have you pushed it yet? You can push with:"
> ```
> git push -u origin main
> ```

### Step 1.4 — Check account type (EMU detection)

Inspect the owner from the remote URL. EMU (Enterprise Managed User) accounts typically follow the pattern `<enterprise>_<username>` (e.g., `tenable_jbuchanan`).

If the owner contains an underscore and looks like an EMU pattern, warn:
> "It looks like this repo is under an Enterprise Managed User account (`<owner>`). EMU accounts cannot fork external repositories, which is required to submit to the Exchange. You'll need to push this repo to a personal GitHub account instead."

Ask: "Do you have a personal GitHub account? If so, what's the username? I can help you add it as a remote and push."

If they don't have one, guide them to github.com/signup to create a free account.

After any account switch, re-run the validation from Step 1.2.
