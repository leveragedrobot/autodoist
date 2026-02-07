---
name: autodoist
description: Autonomous Todoist task execution. Scans all tasks, classifies using 5-signal priority system, routes to skills, and completes what it can — with your approval.
version: 2.0.0
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

# Software project backlogs (undated tasks Claude can tackle)
td task list --project "Schedule App" --json
td task list --project "Browser Extension" --json
td task list --project "Trading Alerts" --json
td task list --project "Trading Algo" --json
```

Also fetch project list for ID-to-name mapping:
```bash
td project list --json
```

## Step 2: Classify Using Signals (in priority order)

Use these concrete signals to classify tasks. Check them in order — the first match wins.

### Signal 1: Labels (strongest signal)

| Label | Meaning | Claude? |
|-------|---------|---------|
| `computer` | Done at computer | Almost always YES |
| `work` | Work context | Maybe (check content) |
| `home` | Physical/household | NO |
| `errands` | Out-of-house | NO |
| `iphone` | Phone-only | NO |
| `quick` | Fast task | Check other signals |
| `delegated` | Waiting on someone | NO |
| `leadership meeting` | Meeting prep | Partial (can draft agendas) |
| `district call` | Meeting prep | Partial (can draft notes) |

### Signal 2: Project Context

| Project | Claude Can Do |
|---------|--------------|
| **Schedule App** | Code features, run tests, deploy, browser test |
| **Browser Extension** | Code features, deploy, draft marketing content |
| **Trading Alerts** | Code features (via SSH to remote server), draft content |
| **Trading Algo** | Code features (via SSH), run tests, **but NOT live algo fixes** |
| **Retail Manager** | Only `computer`-labeled tasks, and only prep/drafting |
| **Personal** | Research, writing, planning — NOT physical tasks |

### Signal 3: Skill Mapping (task content → available skill)

Match task content to existing skills that can execute them:

| Task Pattern | Skill Chain | Execution Method |
|-------------|------------|-----------------|
| "posting scans", "scan tweets" | `/content-post` → `/post-to-x` | SSH scan → format tweets → post via browser → mark done |
| "tweet", "post to twitter/X" | `/content-post` → `/post-to-x` | Generate content → post via browser → mark done |
| "reddit", "post to reddit" | `/content-post` | Draft subreddit-specific posts (user submits) |
| "schedule" + Retail Manager project | `/schedule-maker` | Browser automation on example-app.com |
| "deploy" + Schedule Maker | `/deploy` | Git push + Vercel monitoring |
| "deploy" + trading-algo/algo | `/deploy-trading-algo` | SSH + restart algo |
| "health check", "algo status" | `/health` | System health check |
| "trading status", "positions" | `/trading-status` | Check positions/P&L |
| Coding tasks (implement, add, build, fix) | `/code-task` | Read task → implement → test → commit/PR |
| "test" + any software project | `/run-tests` | Run project test suite, report results |
| "Telegram" setup tasks | `/telegram-setup` | Create channels, configure bots, wire up alerts |
| "Product Hunt", "launch", "Hacker News" | `/launch-prep` | Prep all launch content and checklist |
| "landing page", "email list", "waitlist" | `/landing-page` | Build page, deploy to Vercel, wire email |
| GitHub tasks (PR, issue, code) | GitHub MCP tools | Direct API access |
| "Stripe", "subscription" | Stripe CLI + code | Build integration |
| "Rotate token" | Partial — needs biometric on remote server | Document steps, can't execute headlessly |

**Skill chaining:** When a task maps to multiple skills (e.g., `/content-post` → `/post-to-x`), execute them in sequence automatically. Present content for approval between generation and posting steps.

**Browser-heavy skills** (Amazon, schedule-maker, etc.) should be spawned as sub-agents to avoid burning autodoist's context:
```
sessions_spawn task="Order [item] from Amazon. Use amazon-ordering skill. Confirm with user before checkout. Mark Todoist task [id] complete when done."
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
3. Execute skill-routed tasks via their dedicated skills

**For each approved task:**

1. **Check if task maps to a skill chain** — invoke skills in sequence:
   - Scan/tweet posting → `/content-post` (generate) → present for approval → `/post-to-x` (post via browser)
   - Schedule tasks → `/schedule-maker`
   - Deploy tasks → `/deploy` or `/deploy-trading-algo`
   - Code tasks → `/code-task`
   - Test tasks → `/run-tests`

2. **Code tasks** — navigate to project, read existing code, implement:
   - Schedule Maker: `~/projects/schedule-app`
   - Browser Extension: `~/projects/browser-extension`
   - Trading Algo/Trading Alerts: SSH to remote server `~/projects/trading-algo`

3. **Content tasks** — generate via `/content-post`, present for approval, then post via `/post-to-x` if user confirms

4. **Research tasks** — search web/notes, compile findings

5. **Log the work**: Add comment summarizing what was done
   ```bash
   td comment add --task-id <id> "Completed: [brief summary of work done]"
   ```

6. **Handle subtasks**: If task is complex, create subtasks instead of partial completion
   ```bash
   td add "Subtask description" --parent-id <id>
   ```

7. **Mark complete**: Only after fully done
   ```bash
   td task complete "id:<id>"  # id: prefix required!
   ```

8. **Add follow-up tasks** if work spawns new items:
   ```bash
   td add "Follow-up task #ProjectName"
   ```

**Never mark Human Required tasks complete** — only report them.

## Step 6: Sweep Project Backlogs

After handling today's tasks, scan software project backlogs for undated tasks Claude can knock out:

```bash
td task list --project "Schedule App" --json
td task list --project "Browser Extension" --json
td task list --project "Trading Alerts" --json
td task list --project "Trading Algo" --json
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

## Capabilities Reference

**What Claude has access to:**
- `td` CLI (Todoist)
- `gh` CLI (GitHub)
- `vercel` CLI (deployments)
- `stripe` CLI (payments)
- Browser automation (Chrome via MCP)
- SSH to remote server (user@remote-server)
- All local codebases (Schedule Maker, Browser Extension)
- QMD knowledge base (notes, learnings, reference docs)
- GitHub MCP (PRs, issues, code search)

**What Claude CANNOT do:**
- 1Password on remote server (biometric auth required headlessly)
- Submit forms on employer-system, employer-system, or other employer systems
- Physical tasks
- Phone calls
- Financial transactions (trading, purchases)
- Post to social media without user confirmation (always confirms before clicking Post)

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
- **Live trading system**: NEVER modify algo behavior. Observe and report only.
- **SSH to remote server**: Credential manager requires biometric auth. Only .env files work for creds.
- **Priority order**: Overdue P1 → Today P1 → Overdue P2 → Today P2 → Backlogs
