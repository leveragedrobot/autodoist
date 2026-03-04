---
name: autodoist
description: Autonomous Todoist task execution. Scans all tasks, classifies using 5-signal priority system, routes to skills with chained execution, and completes what it can. Supports inline execution and batch mode for walk-away autonomy.
version: 4.0.0
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

| Task Pattern | Skill Chain | Execution Method |
|-------------|------------|-----------------|
| "tweet", "post to twitter/X" | `/content-post` → `/post-to-x` | Generate content → post via browser → mark done |
| "reddit", "post to reddit" | `/content-post` | Draft subreddit-specific posts (user submits) |
| "deploy" + web project | `/deploy` | Git push + monitoring |
| "deploy" + backend/algo | `/deploy-backend` | SSH + restart service |
| Coding tasks (implement, add, build, fix) | `/code-task` | Read task → implement → test → commit/PR |
| "test" + software project | `/run-tests` | Run project test suite, report results |
| GitHub tasks (PR, issue, code) | GitHub MCP tools | Direct API access |

When a task matches a multi-skill pattern (e.g., content → post), execute it as a **skill chain**. See [Skill Chains](#skill-chains) for the full protocol.

**Skill chaining:** When a task maps to multiple skills, execute them in sequence automatically. Present content for approval between generation and posting steps.

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

```
## Todoist Daily Review - [Today's Date]

### Execute Now (Claude does these autonomously)
| # | Task | Project | How Claude Does It |
|---|------|---------|-------------------|
| 1 | Write unit tests | Backend | `/code-task` → `/run-tests` |
| 2 | Research pricing tools | Side Project | Web search + compile findings |

**Say "go" or pick numbers for inline execution. Say "batch" for walk-away mode.**

### Execute With Approval (Claude does the work, you click "send/post/submit")
| # | Task | Project | What Claude Prepares | You Do |
|---|------|---------|---------------------|--------|
| 1 | Post to Reddit | Product | Draft 3 subreddit posts | Review & submit |
| 2 | Product Hunt launch | Product | Prep submission content | Submit on PH |

### Needs Your Input First (Claude can do it but needs clarification)
| # | Task | Project | Question |
|---|------|---------|----------|
| 1 | Feature upgrades | App | Which features? |

### Human Only
| Task | Project | Why |
|------|---------|-----|
| Call dentist | Personal | Phone call |
| Gym workout | Personal | Physical task |

### Quick Stats
- Total reviewed: X
- Claude can execute: X (Y%)
- Overdue: X (list them)
- Blocked by human input: X
```

**Priority order**: Overdue P1 → Today P1 → Overdue P2 → Today P2 → Backlogs

## Step 5: Execute Approved Tasks

When user approves, check which execution mode they chose:

### Batch Mode (user says "batch")

Only **"Execute Now"** tasks go into the batch. "Execute With Approval" tasks require user interaction and cannot be batched.

1. Write approved tasks to `.claude/daily-batch.md`:

```markdown
# Daily Batch - [today's date]

## Pending
- [ ] <todoist-id> | <task title> | <project> | <skill>
- [ ] <todoist-id> | <task title> | <project> | <skill>

## Completed
(none yet)

## Blocked
(none yet)
```

Each line includes the Todoist task ID, title, project name, and the skill or method to execute it.

2. Execute each task sequentially. On success, move to Completed. On failure, move to Blocked with reason.

3. Mark completed tasks in Todoist:
   ```bash
   td task complete <task-id>
   ```

4. If there are also "Execute With Approval" tasks, mention them after the batch completes.

### Inline Mode (user says "go", picks numbers, or anything else)

Execute tasks in the current session:

1. **Check if task maps to a skill chain** — invoke skills in sequence:
   - Content → post: `/content-post` (generate) → present for approval → `/post-to-x` (post)
   - Deploy: `/deploy` or project-specific deploy command
   - Code tasks: `/code-task`
   - Test tasks: `/run-tests`

2. **Code tasks** — navigate to project, read existing code, implement

3. **Content tasks** — generate via `/content-post`, present for approval, then post if confirmed

4. **Research tasks** — search web/notes, compile findings

5. **Mark complete** after verified done:
   ```bash
   td task complete <task-id>
   ```

6. **Add follow-up tasks** if work spawns new items:
   ```bash
   td add "Follow-up task #ProjectName"
   ```

7. **Add completion notes** to tasks with context:
   ```bash
   td comment add --task-id <task-id> "Completed: [what was done]"
   ```

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
| Browser automation | **Sub-agent** | Heavy DOM interaction, protect autodoist context |

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

After handling today's tasks, scan software project backlogs for undated tasks Claude can knock out:

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
   td task update <id> --due "tomorrow"
   ```

**When partially complete:**
1. Create subtasks for remaining work
2. Add comment noting progress made
3. Complete only the subtasks that are done, not the parent

## Scheduled Run Behavior

When triggered by cron or scheduled automation:
- Message user with summary (via Telegram, Slack, or preferred channel)
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

# Execute
td task complete <id>                # Mark done
td add "Task text #Project"          # Create task
td add "Subtask" --parent-id <id>   # Add subtask
td task update <id> --due "tomorrow" # Reschedule

# Context
td comment add --task-id <id> "text" # Add note
td comment list --task-id <id>       # Read notes
td project list --json               # All projects
```

## Notes

- **Recurring tasks**: Completing creates the next occurrence automatically
- **Overdue tasks**: Always surface these first — they need attention
- **Vague tasks**: Ask for clarification, don't guess. Put in "Needs Input" bucket.
- **Never mark Human Required tasks complete** — only report them.
- **Priority order**: Overdue P1 → Today P1 → Overdue P2 → Today P2 → Backlogs
