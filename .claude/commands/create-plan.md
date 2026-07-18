---
description: create a technical plan from a spec
argument-hint: <spec file relative path, e.g. .claude/specs/03-login_and_logout.md>
allowed-tools: Read, Write, Glob, Bash(git:*), EnterPlanMode
---

Arguments: `$ARGUMENTS` — relative path to a spec file under `.claude/specs/`.

1. Validate the path with Glob. If missing/empty, stop and ask the user.
2. From the filename `<step>-<slug>.md`, extract `step` and `slug` for
   naming the output file.
3. Call `EnterPlanMode`. No code from here on — plan only.
4. Read the spec, `CLAUDE.md`, and the current state of any existing
   files it lists under "Files to Change"/"Files to Create".
5. Write a technical plan (prose/lists, no code blocks) covering: summary,
   source spec path, implementation order, a file-by-file technical
   breakdown (functions/routes/DB helpers touched and how), edge cases
   from the spec's rules/DoD, and open questions.
6. Save it to `.claude/plans/<step>-<slug>.md` (create the folder if
   needed).
7. Do not print the plan in chat and do not call `ExitPlanMode` to ask
   for approval there. Just report the file path and tell the user to
   review it directly — it'll be used later to implement the changes.
