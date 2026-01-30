# Compound Engineering

Orchestration tool for running nightly Claude Code loops across multiple projects.

## Project Overview

This repo contains the `compound` CLI and setup tooling. It is NOT a project that gets the nightly loop itself — it's the tool that sets up and manages nightly loops for other projects.

## How It Works

Each night, per project:

1. **Review** (10:30 PM default) — Exports Claude Code transcripts, reads recent sessions, extracts learnings into the project's CLAUDE.md
2. **Compound** (11:00 PM default) — Reviews codebase, writes research report, creates PRD, converts to Ralph format, runs Ralph loop (up to 20 iterations), runs tests, creates draft PR

## Repo Structure

```
compound/
├── bin/
│   ├── compound              # Global CLI (symlink this to PATH)
│   └── setup-compound.sh     # Project bootstrapper
└── CLAUDE.md                 # This file
```

## What gets created in each project

When you run `compound setup /path/to/project`:

```
project/
├── CLAUDE.md                          # Auto-updated learnings
├── logs/                              # Job output
├── reports/
│   ├── priorities.md                  # Review focus areas (you edit)
│   └── nightly/                       # Dated research reports
├── scripts/
│   ├── daily-compound-review.sh       # Review job
│   └── compound/
│       ├── auto-compound.sh           # 5-phase compound job
│       └── toggle.sh                  # Per-project on/off
```

Plus LaunchAgent plists at `~/Library/LaunchAgents/com.compound.{name}.*.plist`

## Usage

```bash
# Set up a project
compound setup /path/to/project

# See all projects
compound status

# Toggle
compound off my-project
compound on my-project

# View logs
compound logs my-project review
compound logs my-project compound

# Manual trigger
compound run my-project
```

## Prerequisites

- macOS (launchd)
- Claude Code CLI (`claude`)
- `claude-code-transcripts` CLI
- `gh` CLI (for PRs)
- `jq` (for prd.json)
- Ralph (`scripts/ralph/ralph.sh`) in target project for implementation loop
