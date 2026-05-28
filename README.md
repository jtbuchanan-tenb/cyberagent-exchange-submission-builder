# CyberAgents Exchange Submission Builder

A Claude Code skill that automates submitting cybersecurity AI agents and playbooks to the [Tenable CyberAgents Exchange](https://exchange.tenable.com).

## Quick Start

1. Copy the skill to your Claude Code skills directory:

```bash
cp -r cyberagents-exchange-submit ~/.claude/skills/
```

2. Open Claude Code in your agent's repository and say:

```
Submit my agent to the Tenable CyberAgents Exchange
```

That's it. The skill walks you through the rest.

## What It Does

- Validates your repo (pushed to GitHub, has README, has open source license)
- Interviews you to generate the Exchange listing metadata
- Auto-detects fields from your code where possible
- Validates against the Exchange's live schema
- Forks the Exchange content repo, places your listing, and opens a PR

## Full Documentation

See [`cyberagents-exchange-submit/README.md`](cyberagents-exchange-submit/README.md) for installation options, prerequisites, and detailed usage.

## License

MIT
