---
name: autodoist
description: Autonomous Todoist task execution. Scans all tasks, classifies using 5-signal priority system, routes to skills, and completes what it can — with your approval.
version: 2.1.0
---

# Autodoist

Autonomous task review and execution for Todoist using the `td` CLI.

## Step 1: Fetch All Tasks

```bash
# Today + overdue (always)
td today --json

# Inbox (captured items)
td inbox --json

# Upcoming 7 days
td upcoming 7 --json

# All tasks (for backlog scanning)
td task list --json
```

Also fetch project list for ID-to-name mapping:
```bash
td project list --json
```

**Optional: Per-project backlog fetching** for targeted scanning:
```bash
td task list --project "Project Name" --json
```

## Step 2: Classify Using Signals (in priority order)

Use these concrete signals to classify tasks. Check them in order — the first match wins.

### Signal 1: Labels (strongest signal)

Labels are the most reliable indicator. Map your Todoist labels to outcomes:

| Label | Meaning | Claude? |
|-------|---------|---------|
| `computer` | Done at computer | Almost always YES |
| `work` | Work context | Maybe (check content) |
| `home` | Physical/household | NO |
| `errands` | Out-of-house | NO |
| `phone` | Phone-only | NO |
| `quick` | Fast task | Check other signals |
| `delegated` | Waiting on someone | NO |
| `meeting` | Meeting prep | Partial (can draft agendas/notes) |

Customize this table with your own labels.

### Signal 2: Project Context

Define what Claude can do per project. Examples:

| Project | Claude Can Do |
|---------|--------------|
| **Software projects** | Code features, run tests, deploy, browser test |
| **Content projects** | Draft posts, marketing content, documentation |
| **Work projects** | Only `computer`-labeled tasks, prep/drafting only |
| **Personal** | Research, writing, planning — NOT physical tasks |

Customize with your own projects and boundaries.

### Signal 3: Skill Mapping (task content → available skill)

Match task content to existing skills/commands that can execute them:

| Task Pattern | Skill | Execution Method |
|-------------|-------|-----------------|
| "tweet", "post to twitter/X" | `/content-post` → `/post-to-x` | Generate content → post via browser → mark done |
| "reddit", "post to reddit" | `/content-post` | Draft posts (user submits) |
| "deploy" | `/deploy` | Git push + monitoring |
| Coding tasks (implement, add, build, fix) | `/code-task` | Read task → implement → test → commit/PR |
| "test" + software project | `/run-tests` | Run test suite, report results |
| GitHub tasks (PR, issue, code) | GitHub MCP tools | Direct API access |

When a task matches a multi-skill pattern (e.g., content → post), execute it as a **skill chain**. See [Skill Chains](#skill-chains) for the full protocol.

**Discovering your skills:**
```bash
ls ~/.claude/commands/  # List all available skills
```

### Signal 4: Task Description

If a task has a `description` field with step-by-step instructions, Claude can almost certainly follow them. Fetch full details:
```bash
td task view <id> --json
```

### Signal 5: Content Keywords (fallback)

Only use if signals 1-4 are inconclusive:
- Physical verbs: "call", "buy", "go to", "pick up", "clean", "change" → Human
- Digital verbs: "create", "build", "write", "research", "set up", "configure", "test", "draft", "update code" → Claude
- Ambiguous: "set up" (digital setup = Claude, physical setup = Human)

## Step 3: Categorize into Buckets

After classification, sort each bucket by priority (P1 first → P4 last). Flag overdue P1/P2 as URGENT.

**🔀 Skill Routed** (delegated to dedicated skills — most reliable execution)

**✅ Can Complete** (from ALL tasks — execute autonomously after approval):
- Research, lookups, fact-finding
- Writing, drafting, summarizing
- Code tasks, file operations, debugging
- Data analysis, calculations
- Planning, scheduling, organizing
- Building/creating things (websites, scripts, skills)

**⚠️ Need Approval** (from today/inbox/upcoming — require explicit confirmation):
- Sending messages/emails
- Purchases, payments
- Deletions, destructive actions
- External service calls (posting to social media, etc.)

**🚫 Human Required** (from today/inbox/upcoming — cannot be done by agent):
- Physical tasks (errands, cleaning, exercise, medical)
- Sensitive personal decisions
- Waiting on others / blocked tasks
- In-person meetings or calls
- Work systems requiring VPN or special login

## Step 4: Present Summary

Format for Telegram (no markdown tables). Use invisible character (U+200E) before blank lines to preserve spacing between sections:

```
🤖 **AUTODOIST**

🔀 **SKILL ROUTED**
1. Complete managers schedule → `/schedule-maker`
   ⚠️ OVERDUE (Jan 19)
‎
✅ **CAN COMPLETE**
2. Research competitor pricing
   Work | P1
3. Write README for new project
   Side Projects | P2
‎
⚠️ **NEED APPROVAL**
4. Post update to Twitter
   P2 | ⚠️ OVERDUE (Today)
5. Order office supplies
   P3
‎
🚫 **HUMAN REQUIRED**
• Call dentist — P1 ⚠️ URGENT (Tomorrow)
• Gym workout — P4 Today
‎
📊 **STATS**
Total: X | Claude: X (Y%) | Overdue: X | Blocked: X

Reply with numbers to execute (e.g., "1, 2") or "all".
```

**Formatting rules:**
- Headers: Bold with emoji prefix, no description text below
- Numbered items (1-N): Executable tasks (Skill Routed, Can Complete, Need Approval)
- Bullet items (•): Human Required (not numbered since not executable)
- Blank line with U+200E character before each section header
- No blank line between header and its items
- Task details on second line, indented with spaces
- Priority order: Overdue P1 → Today P1 → Overdue P2 → Today P2 → Backlogs

## Step 5: Execute Approved Tasks

When user approves (by number, "all", or "go"):

**Batch similar tasks** for efficiency:
1. Group research tasks together (shared context)
2. Group code tasks in same project together
3. Execute skill-routed tasks via their skill chains

**For each approved task:**

1. **Check if task maps to a skill chain** — follow the chain protocol in [Skill Chains](#skill-chains)

2. **Single-skill tasks** — invoke the skill directly

3. **Non-skill tasks** (research, writing, etc.) — execute inline

4. **Log the work**: Add comment summarizing what was done
   ```bash
   td comment add --task-id <id> "Completed: [brief summary of work done]"
   ```

5. **Handle subtasks**: If task is complex, create subtasks instead of partial completion
   ```bash
   td add "Subtask description" --parent-id <id>
   ```

6. **Mark complete**: Only after fully done
   ```bash
   td task complete "id:<id>"  # id: prefix required!
   ```

7. **Add follow-up tasks** if work spawns new items:
   ```bash
   td add "Follow-up task #ProjectName"
   ```

**Never mark Human Required tasks complete** — only report them.

## Skill Chains

A skill chain is an ordered sequence of skills where each step's output feeds the next. Chains have **approval gates** (pause for user confirmation) and **failure rules** (what happens when a step breaks).

### Chain Protocol

For every chain execution, follow this loop:

```
for each step in chain:
  1. Execute the skill
  2. Capture the output (content, result, artifact)
  3. If step has an APPROVAL GATE:
     → Present output to user
     → Wait for approval
     → If rejected: log rejection, STOP chain, keep task open
  4. If step FAILED:
     → Follow the failure rule for that step type
  5. Pass output as input to next step
after all steps complete:
  → Log work as comment on Todoist task
  → Mark task complete
```

### Approval Gates

Gates prevent irreversible actions without user consent. A gate pauses the chain and shows the user what's about to happen.

**Always gate before:**
- Posting to social media (Twitter, Reddit, etc.)
- Sending messages (email, Slack, Telegram)
- Deploying to production
- Publishing content
- Any action visible to others

**Never gate on:**
- Research/analysis (internal only)
- Code generation (can be reviewed before commit)
- Running tests (non-destructive)
- Fetching data

**Gate format:**
```
Ready to [action]. Here's what I'll [post/send/deploy]:

[content preview]

Proceed? (yes / edit / skip)
```

- **yes** → continue chain
- **edit** → user provides corrections, re-run current step with edits
- **skip** → skip this step, continue chain if possible (otherwise stop)

### Failure Handling

What happens when a step in the chain fails:

| Step Type | On Failure | Action |
|-----------|-----------|--------|
| **Generate** (content, code) | Retry once with adjusted approach | If second attempt fails, log blocker and stop |
| **Test/Validate** | Stop chain, do not proceed to deploy | Log failures as comment, keep task open |
| **Post/Send** | Retry once | If retry fails, log error, keep content for manual posting |
| **Deploy** | Stop chain | Log error, do not mark complete |
| **Browser automation** | Retry once | If retry fails, stop and report — browser state may be stale |

**On any failure:**
```bash
td comment add --task-id <id> "Chain stopped at [step]: [error]. Output so far: [summary]"
```

### Context Strategy

Not all chains should run in the main autodoist context. Browser-heavy chains burn tokens on screenshots and DOM reads.

| Chain Type | Strategy | Why |
|-----------|----------|-----|
| Research → Write | **Inline** | Text-only, fast, benefits from autodoist context |
| Code → Test | **Inline** | Needs project context already loaded |
| Content → Post (browser) | **Inline if short**, sub-agent if complex | Simple tweet = inline; multi-platform campaign = sub-agent |
| Browser automation (scheduling, ordering) | **Sub-agent** | Heavy DOM interaction, protect autodoist context |

**Spawning a sub-agent:**
```
sessions_spawn task="Execute [chain description]. Task ID: [id]. Confirm with user before [gate actions]. Mark Todoist task complete when done: td task complete 'id:<id>'"
```

The sub-agent gets the full chain instructions and handles it independently. Autodoist moves on to the next task.

### Example Chains

**Content → Post chain:**
```
1. /content-post    → generates tweet content
   HANDOFF: tweet text + optional image
2. GATE             → show user the tweet, wait for approval
3. /post-to-x      → posts via browser automation
   HANDOFF: post URL
4. Log + complete   → add comment with post URL, mark task done
```

**Code → Test → Deploy chain:**
```
1. /code-task       → implements feature/fix
   HANDOFF: list of changed files
2. /run-tests       → runs project test suite
   HANDOFF: pass/fail + test output
   ON FAIL: stop chain, log failures
3. GATE             → show test results, ask "deploy?"
4. /deploy          → pushes to production
   HANDOFF: deploy URL
5. Log + complete   → add comment with deploy URL, mark task done
```

**Defining your own chains:**

Add new chains by following this pattern:
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

## Step 6: Sweep Project Backlogs

After handling today's tasks, scan project backlogs for undated tasks Claude can knock out:

```bash
td task list --project "Project Name" --json
```

Present any low-hanging fruit: tasks with clear descriptions, test tasks, small code changes.

## Step 7: Error Handling

**When blocked on a "Can Complete" task:**
1. Do NOT mark complete
2. Add comment explaining the blocker:
   ```bash
   td comment add --task-id <id> "Blocked: [reason]. Needs: [what's required to proceed]"
   ```
3. Move to "Need Approval" category in next review with explanation
4. Optionally reschedule if appropriate:
   ```bash
   td task update "id:<id>" --due "tomorrow"
   ```

**When partially complete:**
1. Create subtasks for remaining work
2. Add comment noting progress made
3. Complete only the subtasks that are done, not the parent

## Scheduled Run Behavior

When triggered by cron:
- Message user on Telegram with summary
- Wait for reply before executing anything
- If no actionable tasks, send brief "Nothing needs attention" or skip message entirely
- Prioritize URGENT (overdue P1/P2) items at top of message

## Customization

Edit this file to add your own:
- **Labels** (Signal 1): Map your Todoist labels
- **Projects** (Signal 2): Define per-project capabilities
- **Skills** (Signal 3): Add your custom `/commands`
- **Chains**: Define multi-skill sequences with gates
- **Capabilities**: What your setup can/cannot do

## Quick Reference: td CLI Commands

```bash
# Fetch
td today --json                      # Today + overdue
td inbox --json                      # Inbox
td upcoming 7 --json                 # Next 7 days
td task list --project "Name" --json # Project tasks
td task view <id> --json             # Full task details
td task list --json                  # All tasks

# Execute (IMPORTANT: use "id:<id>" prefix for task operations!)
td task complete "id:<id>"           # Mark complete
td add "Task" --parent-id <id>      # Add subtask
td task update "id:<id>" --due "tomorrow"  # Reschedule

# Context
td comment add --task-id <id> "text" # Add note
td comment list --task-id <id>       # Read notes
td project list --json               # All projects
```

## Notes

- **Recurring tasks**: Completing creates the next occurrence automatically
- **Overdue tasks**: Always surface these first — they need attention
- **Vague tasks**: Ask for clarification, don't guess. Put in "Needs Input" bucket.
- **Priority order**: Overdue P1 → Today P1 → Overdue P2 → Today P2 → Backlogs
