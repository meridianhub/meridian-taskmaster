# Meridian Task Master Repository

This is the **project management repository** for the Meridian Hub project, containing all Task Master AI tracking and planning files.

## Purpose

This repository sits **outside** the main `meridian` codebase to prevent merge conflicts and file clashes during development:

- **Separate from code PRs**: Task changes don't interfere with code pull requests
- **No JSON conflicts**: Multiple developers can work on different tasks without `tasks.json` merge conflicts
- **Clean code history**: Code repository stays focused on implementation, not task management
- **Shared across worktrees**: Single source of truth for all development branches

## Structure

```
.taskmaster/
├── tasks/
│   ├── tasks.json          # Main task database
│   └── task-*.md          # Individual task files (auto-generated)
├── docs/
│   └── prd-*.md           # Product Requirements Documents
├── reports/
│   └── *.json             # Complexity analysis reports
├── templates/
│   └── example_prd*.md    # PRD templates
├── config.json            # AI model configuration
├── state.json             # Current tag and session state
└── CLAUDE.md              # Task Master workflow guide
```

## How It Works

The main `meridian` repository contains a **symlink** pointing to this directory:

```bash
meridian/
├── .taskmaster -> ../.taskmaster/  # Symlink to this repo
├── meridian-main/                  # Main codebase (pristine develop)
└── worktree/                       # Feature branches
```

This allows:
- Task Master commands to work from any location within the project
- Tasks to be tracked centrally regardless of which branch you're on
- PRs in the main repo to never touch task management files

## Usage

From anywhere in the Meridian project:

```bash
# View tasks
task-master list

# Get next task
task-master next

# Update task status
task-master set-status --id=1.2 --status=in-progress

# Create tasks from PRD
task-master parse-prd .taskmaster/docs/prd.md
```

## Git Workflow

This repository is managed independently:

```bash
cd /path/to/meridianhub/.taskmaster

# Make task changes (via task-master commands)
task-master update-task --id=X --prompt="..."

# Commit and push
git add .
git commit -m "update: task progress"
git push origin main
```

**No PRs required** - push directly to `main` since this is project coordination, not production code.

## Related

- **Main Repository**: [meridianhub/meridian](https://github.com/meridianhub/meridian)
- **Task Master Documentation**: See `CLAUDE.md` in this repo
- **Git Remote**: https://github.com/meridianhub/meridian-taskmaster.git
