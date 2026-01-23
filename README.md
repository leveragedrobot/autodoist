# 🤖 Autodoist

Autonomous Todoist task execution for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Scans your tasks, categorizes them by what AI can handle, routes to dedicated skills, and executes what it can — with your approval.

## Features

- **Smart categorization**: Separates tasks into "Can Complete", "Need Approval", and "Human Required"
- **Skill routing**: Delegates tasks to specialized `/commands` when available
- **Priority ordering**: P1 tasks first, with URGENT flags for overdue items
- **Work logging**: Adds comments to completed tasks documenting what was done
- **Error handling**: Proper handling for blocked or partially complete tasks
- **Batch execution**: Groups similar tasks for efficiency

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- [td CLI](https://github.com/sachaos/todoist) (Todoist CLI) installed and authenticated
  ```bash
  # Install (macOS)
  brew install sachaos/todoist/todoist

  # Authenticate
  todoist sync
  ```

## Installation

### Option 1: Clone and Symlink (Recommended)

```bash
# Clone the repo
git clone https://github.com/leveragedrobot/autodoist.git ~/autodoist

# Create the plugins directory if it doesn't exist
mkdir -p ~/.claude/plugins/marketplaces/claude-plugins-official/plugins

# Symlink to Claude's plugin directory
ln -s ~/autodoist ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/autodoist
```

### Option 2: Direct Clone

```bash
git clone https://github.com/leveragedrobot/autodoist.git \
  ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/autodoist
```

## Usage

In any Claude Code session:

```
/autodoist
```

Claude will analyze your tasks and present a categorized summary:

```
🤖 AUTODOIST

🔀 SKILL ROUTED (will delegate to dedicated skills):
| # | Task | Skill | Due |
|---|------|-------|-----|
| 1 | Complete managers schedule | /schedule-maker | Today |

✅ CAN COMPLETE (ready to execute):
| # | Task | Project | Pri | Due |
|---|------|---------|-----|-----|
| 2 | Research competitor pricing | Work | P1 | — |
| 3 | Write README for new project | Side Projects | P2 | — |

⚠️ NEED APPROVAL:
| # | Task | Pri | Due |
|---|------|-----|-----|
| 4 | Post update to Twitter | P2 | Today ⚠️ |

🚫 HUMAN REQUIRED:
| Task | Pri | Due | Reason |
|------|-----|-----|--------|
| Call dentist | P1 | Tomorrow ⚠️ URGENT | Phone call |

Reply with task numbers (e.g., "1, 2, 3") or "all" to execute.
```

## How It Works

### Task Categories

| Category | Source | Description |
|----------|--------|-------------|
| **Skill Routed** | All tasks | Delegated to dedicated `/commands` |
| **Can Complete** | All tasks | Research, writing, code, planning |
| **Need Approval** | Today/Inbox/Upcoming | Emails, purchases, deletions |
| **Human Required** | Today/Inbox/Upcoming | Physical tasks, calls, sensitive decisions |

### Execution Flow

1. Fetches tasks via `td` CLI
2. Checks for skill routing matches
3. Categorizes remaining tasks
4. Presents summary sorted by priority
5. Executes approved tasks with logging
6. Marks complete in Todoist

## Customization

Edit `commands/autodoist.md` to:
- Add skill routing patterns for your custom commands
- Adjust categorization rules
- Modify the output format

## License

MIT
