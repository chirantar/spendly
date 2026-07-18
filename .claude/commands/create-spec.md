---
description: create spec of a feature
argument-hint: <step/feature no and> <feature name>
allowed-tools: Read, Write, Glob, Bash(git:*)
---

Arguments: `$ARGUMENTS` — a step number and a feature name/title.

Follow these steps in order. Do not skip ahead — each step gates the next.

### 1. Check working directory is clean

Run `git status`. If there are any uncommitted changes (staged or unstaged),
stop immediately and tell the user their working directory isn't clean —
ask them to commit or stash before running this command again. Do not
proceed to step 2.

### 2. Parse the arguments

From `$ARGUMENTS`, work out:

- **step_number** — pad to two digits (e.g. `7` → `07`)
- **feature_title** — human-readable title (e.g. "Add Expense Form")
- **feature_slug** — snake_case slug of the title (e.g. `add_expense_form`)
- **branch_name** — inferred short kebab-case branch name for the feature
  (e.g. `feature/07-add-expense-form`)

### 3. Check whether you're already on the target branch

Run `git branch --show-current`. If it already equals `<branch_name>`,
skip straight to step 5 — you're already on the right branch, do not
switch, pull, or create anything.

Otherwise, check the branch name isn't already taken: run
`git branch --list <branch_name>` (and check remotes with
`git branch -r --list origin/<branch_name>` if useful). If it already
exists, stop and tell the user the branch name is taken — suggest they
pick a different feature name or delete/reuse the existing branch. Do not
proceed to step 4.

### 4. Switch, pull, and create the branch

- `git checkout master`
- `git pull`
- `git checkout -b <branch_name>`

All following steps happen on this new branch.

### 5. Read project context

Before writing anything, read:

- `CLAUDE.md` — architecture, code style, tech constraints, rules
- `app.py` — current routes, what's implemented vs stub
- `database/db.py` — current schema, helpers, existing tables/columns
- `.claude/specs/` — list existing specs and skim them so the new spec
  doesn't duplicate work already speced by a prior step

Use this to understand what already exists and what this feature actually
needs to add or change.

### 6. Generate the spec document

Write a markdown document with this exact structure:

```
# Spec: <feature_title>

## 1. Overview
<what this feature does, and how it fits into the app>

## 2. Depends on
<previous step(s) this builds on, or "Nothing — this is the first step.">

## 3. Routes
<if new routes/APIs are needed: method, path, purpose, access level
 (e.g. login required / owner-only) for each.
 If no new routes: state which existing routes are reused instead.>

## 4. Database Changes
<new tables, columns, or constraints needed — cross-check against the
 current schema in database/db.py so nothing is duplicated.
 If none: state "No database changes.">

## 5. Templates
<templates to create or modify — must extend base.html>

## 6. Files to Change
<existing files touched>

## 7. Files to Create
<new files>

## 8. Dependencies
<new pip packages, or "No new dependencies">

## 9. Rules for Implementation
- No SQLAlchemy or other ORMs
- Parameterized queries only — never string-format SQL
- Passwords hashed with werkzeug's generate_password_hash
- Styles use CSS variables — never hardcode hex values
- All templates extend base.html

## 10. Definition of Done
- [ ] <testable item — verifiable by running the app>
- [ ] ...
```

Every Definition of Done item must be something that can be checked by
actually running the app (not just "code compiles").

### 7. Save the spec

Write the file to `.claude/specs/<step_number>-<feature_slug>.md`.

### 8. Report back

Do not print the spec document's contents in chat unless the user asks
to see it. Only output:

- Branch name
- Feature title
- Spec doc path
