# Autodoist

Autonomous Todoist task execution for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [Clawdbot](https://clawd.bot). Scans your tasks, classifies them using a 5-signal priority system, routes to dedicated skills with chained execution, and completes what it can — with your approval.

## Features

- **5-signal classification**: Cascading priority system (labels → project context → skill mapping → task descriptions → content keywords) for accurate task categorization
- **Skill chaining**: Multi-step execution pipelines with approval gates, handoff data between steps, and per-step failure handling
- **Smart categorization**: Separates tasks into Skill Routed, Can Complete, Need Approval, and Human Required
- **Approval gates**: Pauses before irreversible actions (posting, deploying, sending) with yes/edit/skip options
- **Failure handling**: Per-step-type rules — tests fail? never deploy. Post fails? retry once, then save content for manual posting
- **Context strategy**: Automatically decides inline vs sub-agent execution based on chain type (browser-heavy = sub-agent)
- **Priority ordering**: P1 tasks first, with URGENT flags for overdue items
- **Work logging**: Adds comments to completed tasks documenting what was done
- **Batch execution**: Groups similar tasks for efficiency
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
🤖 AUTODOIST

🔀 SKILL ROUTED
1. Complete managers schedule → /schedule-maker
   ⚠️ OVERDUE (Jan 19)

✅ CAN COMPLETE
2. Research competitor pricing
   Work | P1
3. Write README for new project
   Side Projects | P2

⚠️ NEED APPROVAL
4. Post update to Twitter
   P2 | ⚠️ OVERDUE (Today)

🚫 HUMAN REQUIRED
• Call dentist — P1 ⚠️ URGENT (Tomorrow)
• Gym workout — P4 Today

📊 STATS
Total: 8 | Claude: 4 (50%) | Overdue: 2 | Blocked: 1

Reply with numbers to execute (e.g., "1, 2") or "all".
```

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

**Context strategy** keeps autodoist efficient:

| Chain Type | Strategy |
|-----------|----------|
| Text-only (research, content) | Inline |
| Code → Test → Deploy | Inline |
| Browser-heavy (scheduling, ordering) | Sub-agent |

### Task Categories

| Category | Source | Description |
|----------|--------|-------------|
| **Skill Routed** | All tasks | Delegated to skill chains |
| **Can Complete** | All tasks | Research, writing, code, planning |
| **Need Approval** | Today/Inbox/Upcoming | Posts, emails, purchases, deletions |
| **Human Required** | Today/Inbox/Upcoming | Physical tasks, calls, sensitive decisions |

### Execution Flow

1. Fetches tasks via `td` CLI (today, inbox, upcoming, backlogs)
2. Classifies each task using the 5-signal cascade
3. Categorizes into buckets, sorted by priority
4. Presents summary with stats
5. Executes approved tasks — single skills or full chains
6. Logs work, marks complete, creates follow-up tasks
7. Sweeps project backlogs for low-hanging fruit

## Scheduling (Clawdbot)

If using with [Clawdbot](https://clawd.bot), set up a cron job to run Autodoist automatically:

**Recommended: 3x daily** (saves tokens vs hourly)
```bash
# Cron schedule: 7am, 12pm, 5pm
clawdbot cron add --name autodoist --schedule "0 7,12,17 * * *" --tz America/New_York
```

**Alternative: Hourly during work hours**
```bash
# Every hour from 7am-8pm
clawdbot cron add --name autodoist --schedule "0 7-20 * * *" --tz America/New_York
```

## Customization

Edit `commands/autodoist.md` to configure:

- **Labels** (Signal 1) — map your Todoist labels to YES/NO/PARTIAL
- **Projects** (Signal 2) — define what Claude can do per project
- **Skills** (Signal 3) — add your custom `/commands` and skill chains
- **Chain definitions** — create multi-skill sequences with gates and failure rules
- **Capabilities** — document what your environment can and cannot do

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
