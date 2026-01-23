---
name: autodoist
description: Autonomous Todoist task execution. Scans all tasks, categorizes by executability, routes to skills, and completes what it can — with your approval.
---

# Autodoist

Autonomous task review and execution for Todoist using the `td` CLI.

## Workflow

### 1. Fetch Tasks

**For "Can Complete" items — scan ALL tasks:**
```bash
td task list --json  # All tasks across all projects
```

**For "Need Approval" and "Human Required" — scan time-sensitive tasks:**
```bash
td today --json      # Today + overdue
td inbox --json      # Captured/unsorted items
td upcoming 7 --json # Next 7 days
```

**Get full context for promising tasks:**
```bash
td task view <id> --json  # Includes description, comments, labels
```

### 2. Check Skill Routing

Before categorizing, check if any task matches a dedicated skill:

| Task Pattern | Skill | Action |
|--------------|-------|--------|
| Matches a `/command` name | That skill | Delegate entirely |
| "create schedule", "managers schedule" | /schedule-maker | Browser automation |
| "check positions", "trading status" | /trading-status | Algo status check |

**To discover available skills:**
```bash
ls ~/.claude/commands/  # List all available skills
```

If a task matches a skill, execute via that skill rather than handling inline. Mark complete only after skill succeeds.

### 3. Categorize Tasks

Sort each category by priority (P1 first → P4 last). Flag overdue P1/P2 as URGENT.

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

### 4. Present Summary

Format as a table, sorted by priority within each category:

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
| 5 | Order office supplies | P3 | — |

🚫 HUMAN REQUIRED:
| Task | Pri | Due | Reason |
|------|-----|-----|--------|
| Call dentist | P1 | Tomorrow ⚠️ URGENT | Phone call |
| Gym workout | P4 | Today | Physical |

Reply with task numbers (e.g., "1, 2, 3") or "all" to execute.
```

### 5. Execute Approved Tasks

**Batch similar tasks** for efficiency:
1. Group research tasks together (shared context)
2. Group code tasks in same project together
3. Execute skill-routed tasks via their dedicated skills

**For each approved task:**

1. **Gather context**: Check task description for specs, links, or requirements
2. **Execute**: Perform the work (research, write, code, delegate to skill, etc.)
3. **Log the work**: Add comment summarizing what was done
   ```bash
   td comment add --task-id <id> "Completed: [brief summary of work done]"
   ```
4. **Handle subtasks**: If task is complex, create subtasks instead of partial completion
   ```bash
   td add "Subtask description" --parent-id <id>
   ```
5. **Mark complete**: Only after fully done
   ```bash
   td task complete <id>
   ```

**Never mark Human Required tasks complete** — only report them.

### 6. Error Handling

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

When triggered by cron:
- Message user on Telegram with summary
- Wait for reply before executing anything
- If no actionable tasks, send brief "Nothing needs attention" or skip message entirely
- Prioritize URGENT (overdue P1/P2) items at top of message

## Key Differences

- **Skill Routed**: Delegated to dedicated skills — most reliable execution
- **Can Complete**: Scanned from ALL tasks (any project, any due date)
- **Need Approval / Human Required**: Only from today, inbox, and upcoming 7 days

## Quick Reference: td CLI Commands

```bash
# Fetch
td today --json              # Today + overdue
td inbox --json              # Inbox
td upcoming 7 --json         # Next 7 days
td task list --json          # All tasks
td task view <id> --json     # Full task with description

# Execute
td task complete <id>        # Mark complete
td add "Task" --parent-id <id>  # Add subtask
td task update <id> --due "tomorrow"  # Reschedule

# Log
td comment add --task-id <id> "Comment text"
td comment list --task-id <id>
```
