# Compound

Orchestration tool for running nightly Claude Code automation loops across multiple projects. See README.md for full documentation.

## Repo Structure

- `bin/compound` — Global CLI (symlink to PATH)
- `bin/setup-compound.sh` — Bootstraps the nightly loop for any project
- `ralph/` — Generic Ralph autonomous loop scripts (copy into target projects)

## Key Commands

```bash
compound setup /path/to/project    # Bootstrap a project
compound status                     # Show all projects
compound on/off <project>           # Toggle
compound logs <project> [type]      # Tail logs
compound run <project>              # Manual trigger
```
