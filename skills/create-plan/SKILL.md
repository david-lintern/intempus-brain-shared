---
name: create-plan
description: "Create a well-structured implementation plan for any task, upload it to the brain plan store, and return the approval URL. Always includes local_test_notes, deploy_notes, and step-level technical data. Use when asked to plan, write a plan, or create a plan for any task or ticket."
---

# Plan Creation Agent

You are creating an implementation plan and uploading it to the intempus-brain plan store. Follow every section in order. The output is a plan that any agent — including Haiku — can implement without needing to infer anything.

---

## Step 1 — Load brain MCP tools

Before doing anything else:

```
ToolSearch query: "select:mcp__intempus-brain__plan_save,mcp__intempus-brain__plan_step_update,mcp__intempus-brain__rule_search,mcp__intempus-brain__snippet_search,mcp__intempus-brain__knowledge_search"
```

Do not call any `mcp__intempus-brain__` tool before ToolSearch returns the schema.

---

## Step 2 — Gather requirements

If any of the following are missing, ask the user before continuing:

| Info needed | Why |
|---|---|
| What the task achieves | Goes in `description` |
| Which repo(s) are involved | Determines stack, test commands, deploy commands |
| Jira ticket number (if any) | Goes in `ticket` and `id` |
| Whether this is a tutorial (user follows steps) or direct implementation (agent executes) | Affects step detail level |

If a ticket number is given, use it in the plan `id` as `<ticket-lower>-<short-slug>` (e.g. `int-1234-add-user-auth`). If no ticket, use a descriptive kebab slug (e.g. `brain-celery-task-viewer`).

---

## Step 3 — Search the brain before planning

Run these searches in parallel before writing any step:

```
rule_search(query="<what you're about to implement>", scope="<repo>")
snippet_search(query="<relevant pattern>", language="python")   # or typescript/bash
knowledge_search(query="<area being changed>")
```

Read the results. If a rule says "don't do X", honour it in the plan. If a snippet shows an established pattern, reference it in the relevant step's data block rather than reinventing it.

---

## Step 4 — Explore the codebase

Use Read, Bash (grep/find), or the Explore agent to find:

- The exact files that need to change
- The exact line numbers for each change
- The exact current code that will be replaced (for `old_string` in Edit calls)
- Any existing functions or patterns to reuse

Do not write steps that say "edit the file" without knowing which file and which lines.

---

## Step 5 — Determine test commands

Based on the repo(s) involved, determine the exact local test commands:

| Repo / Stack | Test command |
|---|---|
| `intempus-brain` (server.py) | `cd /home/david/Intempus/Projects/intempus-brain && python -c "import server; print('OK')"` |
| `intempus-brain` (brain_admin Django) | `cd /home/david/Intempus/Projects/intempus-brain/brain_admin && python manage.py check` |
| `intempus-brain` (brain_admin frontend) | `cd /home/david/Intempus/Projects/intempus-brain/brain_admin/frontend && npm run build` |
| `django` | `cd /home/david/Intempus/Projects/django && python manage.py check` |
| `data-platform` | `cd /home/david/Intempus/Projects/data-platform && make lint` (runs inside Docker) |
| `operations` (Fabric) | Review changed tasks — no dry-run available; describe what to verify manually |
| `zones` | `cd /home/david/Intempus/Projects/zones && ./push.sh --dry-run` (if available) |

If the repo is not listed, use the minimum from CLAUDE.md:
- Django → `python manage.py check`
- TypeScript/Next.js → `npm run build`
- Python module → `python -c "import <module>; print('OK')"`
- Docker service → start locally + `curl` one endpoint

Write the exact commands as a multi-line string. This becomes `local_test_notes`.

---

## Step 6 — Determine deploy commands

Based on the repo(s) involved:

| Repo | Deploy |
|---|---|
| `intempus-brain` | `cd /home/david/Intempus/Projects/intempus-brain && docker build -t intempus/brain:latest . && docker push intempus/brain:latest && ssh intempus-brain "cd /opt/intempus-brain && docker compose pull && docker compose up -d --force-recreate"` |
| `django` | `cd /home/david/Intempus/Projects/operations && fab staging deploy:django` (or `fab production deploy:django`) |
| `data-platform` | Use `intempus-ops-skills:intempus-data-platform-deploy` skill |
| `zones` | `cd /home/david/Intempus/Projects/zones && ./push.sh` |
| `operations` | Fabric task — describe which task and environment |
| `my-claude-config` / `intempus-brain-shared` | `git push` then `/plugin update <plugin>@<marketplace>` in Claude Code |

Write the exact commands in order. This becomes `deploy_notes`.

---

## Step 7 — Write the steps

Each step must be concrete enough that Haiku can execute it without reading any other context. For every step that edits a file:

```
Step N — <action verb> in <filename>

data block must include:
  file: <exact path relative to repo root>
  lines: <start>-<end>   (the section being changed)
  description: <one sentence: what changes and why>
  change: <exact old code → new code, OR exact new code to insert>
```

For steps that run commands, list the exact commands.

For steps that call MCP tools, list the exact tool call with all parameters filled in.

**Step ordering rule:** Always end with:
- Second-to-last step: "Test locally" — runs `local_test_notes` commands
- Last step: "Deploy" — runs `deploy_notes` commands

---

## Step 8 — Upload to the brain

### 8a — Save the plan

```python
plan_save(
    id="<kebab-slug>",
    title="<short human title>",
    description="<what this achieves and why>",
    context="<repo name, e.g. 'django', 'intempus-brain'>",
    ticket="<INT-NNNN or OPS-NNN, or blank>",
    steps=["<step 1 description>", "<step 2 description>", ...],
    tags=["<repo>", "<feature-area>"],
    priority="<high|medium|low>",
    local_test_notes="<exact test commands, multi-line string>",
    deploy_notes="<exact deploy commands, multi-line string>",
    agent_context={
        "repos": ["<repo>"],
        "key_files": [
            {"path": "<file>", "lines": "<start>-<end>", "description": "<what changes>"},
            ...
        ]
    }
)
```

### 8b — Attach technical data to each step

For every step, immediately after `plan_save`:

```python
plan_step_update(
    id="<plan-id>",
    step=<n>,
    status="todo",
    data={
        "file": "<path>",
        "lines": "<start>-<end>",
        "change": "<exact diff or new code>",
        "commands": ["<cmd1>", "<cmd2>"]   # for command steps
    }
)
```

Run all `plan_step_update` calls in parallel (one per step) — they are independent.

---

## Step 9 — Return the approval URL

After uploading, tell the user:

```
Plan saved: <plan-id>
Approve at: https://intempus-brain.intempus.dk/admin/plans/<plan-id>/

Steps: <N> total
Local test: <first line of local_test_notes>
Deploy: <first line of deploy_notes>
```

The plan starts as `⏳ AWAITING APPROVAL`. No agent will implement it until it is approved.

---

## Rules

- **Never write a step that says "edit the file" without the file path and line range.** Haiku will not be able to find it.
- **Never leave `local_test_notes` or `deploy_notes` blank.** If you don't know the commands, ask the user before saving.
- **Never skip the brain search in Step 3.** Rules and snippets exist to prevent repeating known mistakes.
- **Never start implementation.** This skill only creates and uploads the plan. Stop after Step 9.
