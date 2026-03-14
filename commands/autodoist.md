---
name: autodoist
description: Autonomous Todoist task execution. Scans all tasks, classifies using 5-signal priority system, routes to skills with chained execution, and completes what it can. Supports inline execution and batch mode for walk-away autonomy.
version: 5.0.0
---

# Autodoist

Autonomous task review and execution for Todoist using the `td` CLI.

## Step 0: Pre-Flight Checks

Before fetching tasks, verify the tools you'll need are available. Run all in parallel:

```bash
# td CLI works
td today --json 2>&1 | head -1

# SSH to remote servers (if applicable)
# ssh -o ConnectTimeout=3 user@host 'echo ok' 2>&1

# Check if Chrome extension is responsive (needed for posting, scheduling)
# Skip if no content/posting tasks end up in the queue
```

Record which capabilities are available this run:

| Capability | Check | Fallback if down |
|------------|-------|-----------------|
| SSH to remote server | `ssh -o ConnectTimeout=3 ...` | Skip remote tasks, note as blocked |
| Browser automation | MCP `tabs_context_mcp` | Skip posting tasks, save content as drafts |
| Deploy CLI (Vercel, etc.) | `vercel --version` | Skip deploy tasks |
| Payment CLI (Stripe, etc.) | `stripe version` | Skip payment tasks |

**If a critical dependency is down**, immediately move all tasks that require it to "Blocked" with the reason — don't waste time classifying them.

## Step 1: Fetch All Tasks + Previous Results

### Fetch tasks
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

### Check previous run results

Check for tasks that were blocked or had questions from previous runs:
```bash
# Previous batch results
cat ~/.claude/daily-batch.md 2>/dev/null
```

**Cross-session rules:**
- If a task appears in the previous Blocked section with the same reason, **skip it** — don't re-attempt unless the blocker is resolved
- If a task had a clarification question posted as a Todoist comment, check if the user replied:
  ```bash
  td comment list --task-id <id>
  ```
  If user replied → task is now unblocked, move to "Execute Now"
- If a task was partially completed last run, check the comment for progress notes and continue from where it left off

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

**Important:** Cross-reference with pre-flight results. If a remote server is down, all tasks that require it go straight to Blocked.

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
| "content", "draft", "write post" | `/content-post` | Brainstorm or generate content interactively |
| "deploy" + remote project | `/remote-control-spawn` → `/deploy` | Spawn Claude on remote host → deploy → verify |
| "fix"/"debug" + server/backend | `/remote-control-spawn` → `/code-task` | Spawn on remote → implement fix → test |
| "check logs", "investigate" + server | `/remote-control-spawn` | Spawn on remote → analyze → report findings |
| "restart", "configure" + service | `/remote-control-spawn` | Spawn on remote → execute → verify |

**Skill chaining:** When a task maps to multiple skills, execute them in sequence automatically. Present content for approval between generation and posting steps.

**Discovering your skills:**
```bash
ls ~/.claude/commands/  # List all available skills
```

Add your own skill mappings here as you create new `/commands`.

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

## Step 3: Present Summary

After classification, sort each bucket by priority (P1 first → P4 last). Flag overdue P1/P2 as URGENT.

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

### Blocked (from pre-flight or previous run)
| Task | Project | Reason |
|------|---------|--------|
| Deploy backend fix | Backend | Remote server unreachable |
| Fix scanner timeout | Backend | Same blocker as yesterday |

### Human Only
| Task | Project | Why |
|------|---------|-----|
| Call dentist | Personal | Phone call |
| Gym workout | Personal | Physical task |

### Quick Stats
- Total reviewed: X
- Claude can execute: X (Y%)
- Overdue: X (list them)
- Blocked by pre-flight: X
- Carried over from previous run: X
```

**Priority order**: Overdue P1 → Today P1 → Overdue P2 → Today P2 → Backlogs

## Step 4: Handle "Needs Your Input" Tasks

Before executing approved tasks, handle clarification needs **asynchronously** so they don't stay stuck:

For each task in "Needs Your Input":
1. Post the question as a Todoist comment:
   ```bash
   td comment add --task-id <id> "Question from Claude: [specific question]. Reply here and I'll pick it up next run."
   ```
2. Reschedule to tomorrow so it resurfaces:
   ```bash
   td task update <id> --due "tomorrow"
   ```
3. Move on — don't wait for an answer this run.

On the **next autodoist run**, Step 1 will check for replies to these comments. If the user answered, the task moves to "Execute Now."

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
   td comment add --task-id <task-id> "Completed by Claude: [what was done]"
   ```

## Skill Chains

A skill chain is an ordered sequence of skills where each step's output feeds the next. Chains have **approval gates** and **failure rules**.

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
- Sending messages (email, Slack, etc.)
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
| Remote server tasks | **Remote spawn** (`/remote-control-spawn`) | Spawns Claude on the remote host via SSH — runs entirely on-server, returns structured result |

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

**Remote Fix → Test chain:**
```
1. /remote-control-spawn → spawns Claude on remote host with task payload
   HANDOFF: result block (status, summary, files changed)
   ON FAIL: log blocker, stop chain
2. Parse result          → check status field
   ON FAIL: log remote error, keep task open
3. Log + complete        → comment on task with remote summary, mark done
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

### Blocked tasks
1. Do NOT mark complete
2. Add comment explaining the blocker:
   ```bash
   td comment add --task-id <id> "Blocked: [reason]. Needs: [what's required to proceed]"
   ```
3. The task stays open — next autodoist run will check if the blocker is resolved
4. Optionally reschedule if appropriate:
   ```bash
   td task update <id> --due "tomorrow"
   ```

### Partially complete tasks
1. Create subtasks for remaining work:
   ```bash
   td add "Remaining: [what's left]" --parent-id <id>
   ```
2. Add comment noting progress made
3. Complete only the subtasks that are done, not the parent

### Chain failures
Follow the failure handling table in Skill Chains. Always log what step failed and why. Never mark a task complete if the chain didn't finish.

## Scheduled Run Behavior

When triggered by cron or scheduled automation:
- Run pre-flight checks
- Classify all tasks
- Message user with summary (via Telegram, Slack, or preferred channel)
- **Wait for reply before executing anything**
- If no actionable tasks, send brief "Nothing needs attention" or skip message entirely
- Prioritize URGENT (overdue P1/P2) items at top of message

## Capabilities Reference

Document what your setup can and cannot do. This helps autodoist make accurate classification decisions.

**Example capabilities:**
- `td` CLI (Todoist)
- `gh` CLI (GitHub)
- Deploy CLI (Vercel, Netlify, etc.)
- Browser automation (Chrome via MCP)
- SSH to remote servers
- Local codebases
- GitHub MCP (PRs, issues, code search)

**Example limitations:**
- Password manager with biometric auth (can't unlock headlessly)
- Employer-specific systems (VPN, HR portals)
- Physical tasks
- Phone calls
- Financial transactions (trading, purchases)
- Post to social media without user confirmation

Customize this section with your actual environment.

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
- **Vague tasks**: Ask clarification via Todoist comment, reschedule to tomorrow, move on
- **Never mark Human Required tasks complete** — only report them.
- **Priority order**: Overdue P1 → Today P1 → Overdue P2 → Today P2 → Backlogs
