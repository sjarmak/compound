# Compound

A nightly automation loop for Claude Code that reviews your work, extracts learnings, researches improvements, and ships code while you sleep.

## What It Does

Every night, two jobs run per project:

**1. Compound Review** — Exports your Claude Code session transcripts, reads them, and extracts patterns, gotchas, and decisions into the project's `CLAUDE.md`. This builds a living knowledge base that makes every future session smarter.

**2. Auto-Compound** — Runs 5 phases:

| Phase | What happens | Output |
|-------|-------------|--------|
| Research | Reviews codebase, past reports, CLAUDE.md | `reports/nightly/YYYY-MM-DD-review.md` |
| PRD | Creates a PRD from the top recommendation | `tasks/prd-compound-YYYY-MM-DD.md` |
| Ralph JSON | Converts PRD to Ralph's task format | `prd.json` |
| Ralph loop | Runs up to 20 iterations on a feature branch | Committed code + `progress.txt` |
| Validate | Runs tests, pushes branch, creates draft PR | Draft PR on GitHub |

Each morning you wake up to:
- Updated `CLAUDE.md` with patterns your agent learned
- A research report with specific recommendations
- A draft PR implementing the top recommendation
- Logs showing exactly what happened

## Install

### Prerequisites

- **macOS** (uses launchd for scheduling)
- **Claude Code** CLI

```bash
# Verify Claude Code is installed
claude --version
```

### 1. Clone this repo

```bash
git clone https://github.com/sjarmak/compound.git ~/compound
```

### 2. Install dependencies

```bash
# Claude Code transcript exporter (for reviewing past sessions)
uv tool install claude-code-transcripts

# GitHub CLI (for creating draft PRs)
brew install gh
gh auth login

# jq (for parsing prd.json)
brew install jq
```

### 3. Add the `compound` CLI to your PATH

```bash
ln -sf ~/compound/bin/compound ~/.local/bin/compound
```

Or add `~/compound/bin` to your PATH in your shell config:

```bash
export PATH="$HOME/compound/bin:$PATH"
```

### 4. Set up the caffeinate agent

launchd won't wake a sleeping Mac. This keeps it awake from 5 PM to 2 AM:

```bash
cat > ~/Library/LaunchAgents/com.compound.caffeinate.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.compound.caffeinate</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/caffeinate</string>
    <string>-i</string>
    <string>-t</string>
    <string>32400</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key>
    <integer>17</integer>
    <key>Minute</key>
    <integer>0</integer>
  </dict>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.compound.caffeinate.plist
```

Extend the `-t` value (seconds) if you run multiple projects that go past 2 AM.

### 5. Set up Ralph in your project (for the implementation loop)

The auto-compound job uses Ralph to implement features autonomously. Copy the Ralph scripts into your project:

```bash
mkdir -p /path/to/your/project/scripts/ralph
cp ~/compound/ralph/ralph.sh /path/to/your/project/scripts/ralph/
cp ~/compound/ralph/CLAUDE.md /path/to/your/project/scripts/ralph/
cp ~/compound/ralph/prompt.md /path/to/your/project/scripts/ralph/
chmod +x /path/to/your/project/scripts/ralph/ralph.sh
```

If your project doesn't have Ralph, the auto-compound job will skip the implementation phase and still produce the research report and PRD.

## Usage

### Set up a project

```bash
compound setup /path/to/your/project
```

This creates the scripts, launchd agents, directories, and `CLAUDE.md` in your project. The nightly loop starts immediately.

With custom schedule (to stagger multiple projects):

```bash
compound setup /path/to/project \
  --review-hour 1 --review-min 0 \
  --compound-hour 1 --compound-min 30
```

### Manage projects

```bash
# See all projects and their status
compound status

# Disable a project's nightly loop
compound off my-project

# Re-enable it
compound on my-project

# View logs
compound logs my-project compound    # auto-compound output
compound logs my-project review      # review job output

# Manually trigger the review job
compound run my-project
```

### Customize what the nightly review focuses on

Edit `reports/priorities.md` in your project. This file tells the research phase what to look for:

```markdown
## Code Quality
- Review for stale code, outdated patterns, or technical debt

## Architecture
- Identify improvements to the data pipeline

## Features
- We need better error handling in the API layer
```

## How It Works

### The Review Job

1. Runs `claude-code-transcripts all` to export all Claude Code sessions as HTML
2. Sessions are stored at `~/claude-archive/{username}-{project}/`
3. Claude reads recent session transcripts and extracts learnings
4. Updates the project's `CLAUDE.md` with new patterns, gotchas, and decisions
5. Commits and pushes to main

### The Auto-Compound Job

1. Pulls main (with fresh `CLAUDE.md` from the review job)
2. Claude reviews the codebase and produces a research report
3. Creates a PRD from the top recommendation
4. Converts the PRD to Ralph's `prd.json` format
5. Runs `ralph.sh --tool claude 20` on a feature branch
6. Ralph iterates through user stories one at a time, committing each
7. Runs tests to validate
8. Pushes the branch and creates a draft PR

### Ralph

Ralph is an autonomous coding loop. Each iteration:
1. Reads `prd.json` for the current feature and stories
2. Picks the highest priority story where `passes: false`
3. Implements it
4. Runs quality checks (typecheck, lint, tests)
5. Commits with `feat: [US-001] - Story title`
6. Marks the story as `passes: true`
7. Appends learnings to `progress.txt`

When all stories pass, Ralph exits. Otherwise it continues to the next iteration.

## Project Structure

### This repo (orchestration)

```
compound/
├── bin/
│   ├── compound              # Global CLI
│   └── setup-compound.sh     # Project bootstrapper
├── ralph/
│   ├── ralph.sh              # Autonomous coding loop
│   ├── CLAUDE.md             # Ralph agent instructions
│   └── prompt.md             # Ralph prompt (for Amp CLI)
├── CLAUDE.md                 # This repo's documentation
└── README.md
```

### What gets created in each project

```
project/
├── CLAUDE.md                          # Living knowledge base (auto-updated)
├── prd.json                           # Current Ralph task list (auto-generated)
├── progress.txt                       # Ralph iteration log
├── logs/
│   ├── compound-review.log            # Review job output
│   └── auto-compound.log              # Auto-compound output
├── reports/
│   ├── priorities.md                  # Review focus areas (you edit this)
│   └── nightly/
│       └── YYYY-MM-DD-review.md       # Nightly research reports
├── tasks/
│   └── prd-compound-YYYY-MM-DD.md     # Generated PRDs
└── scripts/
    ├── daily-compound-review.sh       # Review job
    ├── ralph/                         # Ralph loop (copy from this repo)
    │   ├── ralph.sh
    │   └── CLAUDE.md
    └── compound/
        ├── auto-compound.sh           # 5-phase compound job
        └── toggle.sh                  # Per-project on/off
```

### LaunchAgent plists

Created at `~/Library/LaunchAgents/`:
```
com.compound.{project-name}.review.plist
com.compound.{project-name}.compound.plist
com.compound.caffeinate.plist
```

## Multiple Projects

Stagger schedules so projects don't overlap — Ralph can run for hours:

```bash
# Project A: 10:30 PM / 11:00 PM
compound setup ~/projects/project-a

# Project B: 1:00 AM / 1:30 AM
compound setup ~/projects/project-b \
  --review-hour 1 --review-min 0 \
  --compound-hour 1 --compound-min 30

# Project C: 4:00 AM / 4:30 AM
compound setup ~/projects/project-c \
  --review-hour 4 --review-min 0 \
  --compound-hour 4 --compound-min 30
```

Check all at once:

```bash
compound status
```

## Debugging

```bash
# Check if jobs are scheduled
launchctl list | grep compound

# Tail logs in real time
compound logs my-project compound
compound logs my-project review

# Manually trigger
compound run my-project

# Or trigger a specific job directly
launchctl start com.compound.my-project.review
launchctl start com.compound.my-project.compound
```

## What Pushes Where

- **CLAUDE.md updates** → pushed directly to `main` (learnings only, no code)
- **Research reports** → pushed directly to `main` (markdown only)
- **Code changes** → pushed to a `ralph/compound-YYYY-MM-DD` feature branch → draft PR

Code never lands on main without your review.

## Credits

Inspired by [Ryan Carson's compound engineering post](https://x.com/ryancarson/status/2016520542723924279) and built on:
- [Compound Engineering Plugin](https://github.com/kieranklaassen/claude-code-compound) by Kieran Klaassen
- Claude Code by Anthropic
