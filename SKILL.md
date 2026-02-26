---
name: codex-director
description: Orchestrates coding tasks by delegating to Codex CLI sub-agents. Use this skill when acting as a Principal Lead Programmer to manage and oversee coding projects performed by other AI agents using the Codex CLI.
---

# Codex Director

## Overview

As the Principal Lead Programmer, this skill enables me to direct end-to-end delivery through Codex CLI sub-agents in **headless mode**. The goal is to produce reliable code with minimal prompt rounds (preferably one-shot), enforce guardrails, and keep all outputs traceable through GitHub and `gh` CLI.

This skill is optimized for OpenClaw-style orchestration where multiple upstream agents provide assets, narratives, and data definitions before coding starts.

---

## Core Operating Model (Asset-Gated Delegation)

Use this sequence every time:

1. **Create/prepare GitHub repo**
   - Create repo (or confirm existing one).
   - Initialize branch strategy (`main` protected; feature branches for Codex runs).
   - Configure required secrets/access for Codex execution.

2. **Wait for external deliverables**
   - Required gates:
     - `/assets/**` complete and uploaded.
     - narrative `.md` complete.
     - optional schema/data structure docs complete.
   - Do **not** start coding until required gates are satisfied.

3. **Stage all upstream artifacts in repo**
   - Commit assets and docs first.
   - Make paths explicit so Codex can reference them deterministically.

4. **One-shot / few-shot Codex delegation**
   - Submit a concise, constraint-heavy prompt with:
     - objective
     - allowed files/paths
     - acceptance tests
     - forbidden actions
     - output contract (commit/PR/checklist)

5. **Monitor every 1 minute**
   - Check whether Codex is done, stalled, or blocked.
   - If stalled, unblock with minimal clarifications and re-run with same guardrails.

6. **Validate + push + PR via `gh`**
   - Run checks/tests.
   - Push branch.
   - Create/manage PR with `gh pr create`, `gh pr view`, `gh pr checks`.

7. **Bugfix loop**
   - If QA fails, delegate focused fix prompts to Codex referencing failing tests/logs.
   - Repeat until all acceptance checks pass.

---

## Guardrailed Prompting Standard for Codex CLI

### Prompt shape (default)

Use this template whenever possible:

```text
You are Codex running in headless CLI mode.

MISSION:
<single concrete outcome>

CONTEXT:
- Repo: <owner/repo>
- Branch: <feature branch>
- Must use these inputs:
  - assets: /assets/...
  - narrative: /docs/narrative.md
  - schema: /docs/schema.md (if present)

CONSTRAINTS:
- Make the smallest correct change.
- Do not modify files outside: <allowed paths>
- Do not remove existing tests unless explicitly instructed.
- Preserve backward compatibility unless requirement says otherwise.
- Run project checks before finishing.

REQUIRED COMMANDS:
1) <build/lint/test commands>
2) <additional domain checks>

OUTPUT CONTRACT:
- Provide summary of changes.
- Provide unified diff or commit(s).
- Provide exact commands run + results.
- If blocked, stop and print: BLOCKED: <reason> + minimal required input.
```

### Guardrail additions (recommended)

- Always specify **allowed paths** and **non-goals**.
- Require Codex to print **assumptions** first for quick validation.
- Require a **rollback note** for risky migrations.
- Require **idempotent scripts** for setup/data tasks.
- Require explicit **"no fake implementation"** policy.

---

## Headless Execution Policy

When invoking Codex CLI, prefer maximum automation and permissions so execution does not stall on avoidable prompts.

- Default to fully automated/headless mode.
- Grant broad permissions by default for filesystem/git/build operations.
- Only switch to interactive/limited mode when sensitive input or human confirmation is truly required.

> Principle: “Autonomous by default, interrupt only for missing business decisions or unavailable external data.”

---

## GitHub + `gh` CLI Operations

Use `gh` as the control plane for all PR lifecycle steps.

### Minimum command set

```bash
# repo/bootstrap
gh repo create <name> --private --source=. --remote=origin --push

# branch/push
git checkout -b feat/<topic>
git push -u origin feat/<topic>

# PR lifecycle
gh pr create --title "<title>" --body-file <pr.md>
gh pr view --web
gh pr checks <pr-number> --watch
gh pr comment <pr-number> --body "<status/update>"
```

### Required PR hygiene

- PR description must include:
  - objective
  - scope / non-scope
  - validation evidence
  - risk + rollback
- Keep PRs small and reviewable.
- Link upstream asset/narrative commits when relevant.

---

## Delegation Playbooks

### 1) Greenfield implementation

- Gate assets/docs → one-shot feature prompt → run full checks → open PR.

### 2) Bugfix/revision

- Reproduce failure first.
- Give Codex failing test/log snippet and target behavior.
- Ask for minimal fix + regression test.

### 3) Asset-driven update

- Commit newly arrived assets first.
- Ask Codex to integrate by referencing exact paths.
- Require before/after behavior summary.

---

## `/script/bash` Convention

Bash automation is encouraged under `/script/bash`.

Recommended scripts:

- `script/bash/preflight.sh`
  - verify tools (`git`, `gh`, `codex`), auth (`gh auth status`), clean working tree.
- `script/bash/wait-for-inputs.sh`
  - loop until required files exist (assets/narrative/schema).
- `script/bash/poll-codex.sh`
  - 60-second status polling loop with timeout/escalation.
- `script/bash/pr-sync.sh`
  - push branch, create/update PR, print checks status.

Script standards:

- `set -euo pipefail`
- clear exit codes/messages
- idempotent where feasible
- log timestamped steps

---

## 1-Minute Monitoring Loop (Reference)

Use a strict loop:

1. Read Codex run status/logs.
2. Classify: `DONE`, `RUNNING`, `STALLED`, `BLOCKED`.
3. If `STALLED` for >2 cycles, send constrained corrective prompt.
4. If `BLOCKED`, provide missing artifact/decision only.
5. Stop only at `DONE` + all checks passing.

---

## Quality Gates Before Merge

Must pass:

- Build/lint/tests defined by repo.
- Security/static checks (if configured).
- Manual sanity check for changed user flows.
- PR checks green in GitHub.

If any fail, run bugfix delegation loop.

---

## Anti-Patterns to Avoid

- Delegating before assets/narrative are ready.
- Vague prompts without acceptance criteria.
- Letting Codex edit unrestricted paths.
- Merging without reproducible command output.
- Large, mixed-purpose PRs.

---

## Quick Improvement Additions (Recommended)

To further strengthen your workflow:

1. Add a **Definition of Ready (DoR)** checklist file in repo (`docs/DoR.md`).
2. Add a **Definition of Done (DoD)** checklist for every PR.
3. Maintain a reusable **prompt library** (`docs/prompts/*.md`) for common tasks.
4. Add CI-required status checks before merge.
5. Add lightweight telemetry (run duration, retries, failure reason) per Codex task.
6. Use branch naming conventions tied to ticket IDs for traceability.

These additions improve reliability, auditability, and one-shot success rate.
