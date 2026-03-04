# Autodoist

Autonomous Todoist task execution for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Scans your tasks, classifies them using a 5-signal priority system, routes to dedicated skills with chained execution, and completes what it can — with your approval.

## Features

- **5-signal classification**: Cascading priority system (labels → project context → skill mapping → task descriptions → content keywords) for accurate task categorization
- **Skill chaining**: Multi-step execution pipelines with approval gates, handoff data between steps, and per-step failure handling
- **Batch mode**: Queue up autonomous tasks for walk-away execution — no babysitting required
- **Pre-flight checks**: Verifies tool availability (SSH, browser, deploy CLI) before classifying tasks — blocks tasks whose dependencies are down
- **Async clarification**: Posts questions as Todoist comments, reschedules, and picks up answers on next run — no blocking on user input
- **Cross-session memory**: Tracks blocked tasks and partial progress across runs via batch file and Todoist comments
- **Smart categorization**: Separates tasks into Execute Now, Execute With Approval, Needs Input, Blocked, and Human Only
- **Approval gates**: Pauses before irreversible actions (posting, deploying, sending) with yes/edit/skip options
- **Failure handling**: Per-step-type rules — tests fail? stop chain, never deploy. Post fails? retry once, then save content for manual posting
- **Context strategy**: Automatically decides inline vs sub-agent execution based on chain type (browser-heavy = sub-agent)
- **Priority ordering**: P1 tasks first, with URGENT flags for overdue items
- **Work logging**: Adds comments to completed tasks documenting what was done
- **Backlog sweeping**: Scans project backlogs for undated tasks after handling today's items

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
## Todoist Daily Review - 2026-03-04

### Execute Now (Claude does these autonomously)
| # | Task | Project | How Claude Does It |
|---|------|---------|-------------------|
| 1 | Write unit tests | Backend | /code-task → /run-tests |
| 2 | Research pricing tools | Side Project | Web search + compile |

Say "go" or pick numbers for inline execution. Say "batch" for walk-away mode.

### Execute With Approval
| # | Task | What Claude Prepares | You Do |
|---|------|---------------------|--------|
| 3 | Post to Reddit | Draft 3 subreddit posts | Review & submit |

### Blocked
| Task | Reason |
|------|--------|
| Deploy backend | Remote server unreachable |

### Human Only
| Task | Why |
|------|-----|
| Call dentist | Phone call |
| Gym | Physical task |

### Quick Stats
Total: 8 | Claude: 5 (63%) | Overdue: 1 | Blocked: 1
```

Reply with numbers to execute, "go" for all, or "batch" for walk-away mode.

## How It Works

### 5-Signal Classification

Tasks are classified using a cascading priority system. The first match wins:

| Signal | Source | Example |
|--------|--------|---------|
| **1. Labels** | Todoist labels | `computer` → YES, `home` → NO, `errands` → NO |
| **2. Project** | Project membership | Software project → code/deploy, Personal → research only |
| **3. Skills** | Task content matches | "deploy" → `/deploy`, coding task → `/code-task` |
| **4. Description** | Step-by-step instructions | Detailed description → Claude can follow it |
| **5. Keywords** | Verb analysis (fallback) | "build" → Claude, "call" → Human |

### Pre-Flight Checks

Before classifying tasks, autodoist verifies that required tools are available:

```
| Capability       | Check                        | Fallback if down              |
|-----------------|------------------------------|-------------------------------|
| SSH to server    | ssh -o ConnectTimeout=3 ...  | Block remote tasks            |
| Browser          | MCP tabs_context_mcp         | Save content as drafts        |
| Deploy CLI       | vercel --version             | Skip deploy tasks             |
```

Tasks that depend on an unavailable capability go straight to "Blocked" — no wasted classification effort.

### Cross-Session Memory

Autodoist remembers what happened on previous runs:
- **Blocked tasks** with the same reason are skipped (don't re-attempt until blocker is resolved)
- **Clarification questions** posted as Todoist comments are checked for replies
- **Partial progress** is continued from where the last run left off

### Async Clarification

When a task needs user input, autodoist doesn't block:
1. Posts the question as a Todoist comment
2. Reschedules the task to tomorrow
3. Moves on to other tasks
4. Next run checks for the user's reply and unblocks the task

### Execution Modes

**Inline mode** (`go` or pick numbers): Execute tasks in the current session with real-time feedback.

**Batch mode** (`batch`): Queue autonomous tasks to a batch file and execute them sequentially. Tasks that succeed get marked complete; tasks that fail get logged with the blocker reason. Walk away and come back to results.

### Skill Chains

When a task requires multiple skills, autodoist executes them as a chain with approval gates:

```
Example: Content → Post chain

1. /content-post   → generates tweet content
   HANDOFF: tweet text
2. GATE            → shows user the tweet, waits for approval
3. /post-to-x     → posts via browser automation
   HANDOFF: post URL
4. Log + complete  → comments on task, marks done
```

**Approval gates** pause before irreversible actions. The user gets three options:
- **yes** → continue the chain
- **edit** → provide corrections, re-run the current step
- **skip** → skip this step (stop chain if remaining steps depend on it)

**Failure handling** varies by step type:

| Step Type | On Failure |
|-----------|-----------|
| Generate (content, code) | Retry once, then log blocker |
| Test/Validate | Stop chain — never proceed to deploy |
| Post/Send | Retry once, then save content for manual action |
| Deploy | Stop chain, log error |
| Browser automation | Retry once, then stop (browser state may be stale) |

### Task Categories

| Category | Description |
|----------|-------------|
| **Execute Now** | Autonomous — Claude handles end-to-end |
| **Execute With Approval** | Claude does the work, you click send/post/submit |
| **Needs Your Input** | Claude can do it but needs clarification first |
| **Blocked** | Dependency unavailable or same blocker as previous run |
| **Human Only** | Physical tasks, calls, sensitive decisions |

## Scheduling

Set up a cron job to run Autodoist automatically:

```bash
# 3x daily at 7am, 12pm, 5pm
0 7,12,17 * * * cd /path/to/project && claude -p "/autodoist"
```

When triggered by cron, autodoist messages you with a summary and waits for your reply before executing anything.

## Customization

Edit `commands/autodoist.md` to configure:

- **Labels** (Signal 1) — map your Todoist labels to YES/NO/PARTIAL
- **Projects** (Signal 2) — define what Claude can do per project
- **Skills** (Signal 3) — add your custom `/commands` and skill chains
- **Chain definitions** — create multi-skill sequences with gates and failure rules
- **Capabilities** — document what your environment can and cannot do
- **Pre-flight checks** — add checks for your specific tools and services

### Defining a Skill Chain

Add chains following this pattern in `commands/autodoist.md`:

```
Chain name: [descriptive name]
Trigger: [task pattern that activates this chain]
Steps:
  1. [skill]     → [what it does]
     HANDOFF: [what passes to next step]
  2. GATE        → [what user sees]
  3. [skill]     → [what it does]
  N. Log + complete
```

## License

MIT
