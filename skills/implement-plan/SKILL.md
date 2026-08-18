---
name: implement-plan
description: "Implement an approved plan from the intempus-brain plan store. Loads the plan, checks approval, executes each step in order, runs local tests before deploying, and tracks progress via plan_step_update. Use when asked to implement, run, or execute a plan by ID or name."
---

# Plan Implementation Agent

You are implementing an approved plan from the intempus-brain plan store. Follow every section exactly — do not skip steps, do not infer commands, do not deploy before testing.

---

## Step 1 — Load the brain MCP tools

Before doing anything else, run:

```
ToolSearch query: "select:mcp__intempus-brain__plan_get,mcp__intempus-brain__plan_list,mcp__intempus-brain__plan_step_update,mcp__intempus-brain__plan_update"
```

Do not call any `mcp__intempus-brain__` tool until ToolSearch has returned its schema. Calling without the schema causes `InputValidationError`.

---

## Step 2 — Find the plan

**If a plan ID was given** (e.g. `implement plan brain-celery-task-viewer`):
```
plan_get(id="brain-celery-task-viewer")
```

**If no plan ID was given**, list active plans and ask the user which to implement:
```
plan_list(status="active")
```

---

## Step 3 — Read every field before touching any file

From the `plan_get` output, note these fields:

| Field | What it tells you |
|---|---|
| `Approval:` | Must say `✓ APPROVED` — see Step 4 |
| `Title` / `Description` | What this plan achieves |
| `Local test:` | **Exact commands** to run before deploying |
| `Deploy:` | **Exact deploy commands** — run verbatim after tests pass |
| `Steps` list | Each step has a number, description, status, and `data` block with file paths, line numbers, and exact changes |

Read the `data` block on every step. It contains the exact change. Do not guess the intent from the description alone.

---

## Step 4 — Check approval — MANDATORY GATE

Read the `Approval:` line in the plan_get output.

- **`✓ APPROVED`** → proceed to Step 5
- **`⏳ AWAITING APPROVAL`** → STOP. Tell the user:
  > "This plan is not approved yet. Approve it at https://intempus-brain.intempus.dk/admin/plans/\<id\>/ before I can proceed."
  Do not implement any step.
- **`✗ REJECTED`** → STOP. Report the rejection reason and ask how to proceed.

---

## Step 5 — Execute steps in order

For each step with status `todo` or `in_progress`:

### 5a — Mark in_progress
```
plan_step_update(id="<plan-id>", step=<n>, status="in_progress")
```

### 5b — Do the work
- Read the step `data` block for exact file paths, line numbers, diffs, and commands
- Use the Read tool at the stated lines before editing — never edit blind
- Make only the stated change — do not refactor surrounding code

### 5c — Mark done or skipped
```
plan_step_update(id="<plan-id>", step=<n>, status="done", notes="<one line: what was done>")
```
If a step is already complete or not applicable:
```
plan_step_update(id="<plan-id>", step=<n>, status="skipped", notes="<reason>")
```

Do not start step N+1 before step N is marked done.

---

## Step 6 — Run local tests before deploying

When you reach a step labelled "Test locally" (or the last step before deploy):

Find the `Local test:` section in the plan_get output. Run every command listed **exactly as written**.

If `Local test:` is blank or `—`, use the stack minimum:

| Stack | Minimum check |
|---|---|
| Django | `python manage.py check` |
| Next.js / TypeScript | `cd <frontend-dir> && npm run build` |
| Python module | `python -c "import <module>; print('OK')"` |
| Docker service (brain, data-platform) | `docker compose up -d` locally, then `curl localhost:<port>/health` |
| Fabric / Ansible | `--dry-run` or `--check` flag |

**If any check fails — fix it and re-run. Never proceed to deploy with a failing test.**

---

## Step 7 — Deploy

Find the `Deploy:` section in the plan_get output. Run every command listed **exactly as written**, in order.

If `Deploy:` is blank or `—`, ask the user for deploy instructions before proceeding.

Common brain deploy (only use if the plan's Deploy section says so):
```bash
cd /home/david/Intempus/Projects/intempus-brain
docker build -t intempus/brain:latest .
docker push intempus/brain:latest
ssh intempus-brain "cd /opt/intempus-brain && docker compose pull && docker compose up -d --force-recreate"
ssh intempus-brain "cd /opt/intempus-brain && docker compose ps"
```

---

## Step 8 — Verify and report

After deploying, run any verification checks from the plan's final step. Then report to the user:
- What was implemented
- Test result (pass/fail)
- Deploy result
- Plan URL: `https://intempus-brain.intempus.dk/admin/plans/<id>/`

---

## Rules (never break these)

- **Test before deploy.** Always. No exceptions.
- **Check approval before any step.** Even if the user says "just do it."
- **Use the plan's commands.** `local_test_notes` and `deploy_notes` are the source of truth. If missing, ask.
- **Update step status as you go.** `in_progress` before starting, `done` right after finishing. Never batch.
- **Never amend a previous commit.** Fix the issue and create a new commit.
- **One step at a time.** Finish and mark done before starting the next.
