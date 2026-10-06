---
name: backlog-butler
description: Keep a project backlog in TODO.txt at the project root (or one TODO.txt per sub-repo in a monorepo). Two modes. LOG - any agent that notices a bug, error, flaky test, tech-debt item or improvement idea it is not fixing right now must call this skill to write it down instead of derailing the current task. CULL - when the user asks to "clear/work through/cull the backlog", "do the TODOs", "run the butler", or an agent has idle capacity, implement the backlog items that need no human decision and put the rest to the user as in-chat questions. Use it whenever you spot a miscellaneous problem or suggestion outside your current scope, even if the user never mentions a backlog or TODO file.
---

# Backlog Butler

A project keeps its backlog in `TODO.txt` at the project root; in a monorepo, each sub-repo keeps its own (see Step 0). This skill has two jobs: **log** items into it, and **cull** items out of it. The file is plain todo.txt, so humans and tools (e.g. tuxedo, todo.txt-cli) can read and edit it too.

## Step 0 - Find the backlog and its scope file (both modes)

**Pick the backlog.** Decide whether this is a monorepo: the root has workspace config (`pnpm-workspace.yaml`, `package.json` `workspaces`, `Cargo.toml` `[workspace]`, `go.work`, `nx.json`, `turbo.json`, `lerna.json`) or several independent packages/apps under dirs like `apps/`, `packages/`, `services/`. If not, the backlog is `<root>/TODO.txt`. If so, a **sub-repo** is the nearest enclosing directory, below the root, that has its own manifest (`package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, ...) or its own `AGENTS.md`/`CLAUDE.md`.

- The backlog is `<sub-repo>/TODO.txt` for the sub-repo that contains the file the finding is about.
- Cross-cutting findings (CI, root config, several sub-repos at once) go to `<root>/TODO.txt`.
- Keep one backlog per sub-repo: never log a sub-repo's finding in the root file. Paths in the line are relative to the `TODO.txt` that holds it.

**Pick the scope file.** Look in the backlog's directory for `AGENTS.md`, then `CLAUDE.md`. If the sub-repo has neither, fall back to the root's `AGENTS.md`, then `CLAUDE.md`. Read it before touching the backlog. It tells you what counts as a real bug, what is worth implementing, what is off limits, and how to run tests. If you find none anywhere:

- LOG mode: still log, but add `scope:unchecked` to the line.
- CULL mode: do not implement anything in that backlog. Tell the user no scope file exists and offer to draft one. Without it you cannot tell a worthwhile task from noise, and unattended edits are risky.

Also respect any `## Backlog scope` section in that file; it overrides the defaults below.

## Mode 1 - LOG

Goal: capture the finding in seconds and go back to your task. Do not fix it now unless it blocks you.

1. **Check scope.** Log only if it is a genuine bug/error, or a suggestion that fits the project per the scope file. Skip taste preferences, rewrites nobody asked for, and anything the scope file excludes.
2. **Deduplicate.** Grep `TODO.txt` for the key file, symbol or error text. If a similar open line exists, do not add another; append a short `+1` note via `seen:YYYY-MM-DD` instead.
3. **Append one line** (create the file if missing) in this format:

```
(B) 2026-10-06 Short imperative summary, with file:line or error text +project @bug type:bug src:agent-name needs:none
```

| Part | Rule |
|---|---|
| `(A)`-`(D)` | A = breaks build/data/security, B = real bug, C = improvement, D = nice-to-have |
| date | creation date, ISO |
| text | one line, imperative, include `path:line`, symbol or exact error string so a later agent can act without re-investigating |
| `+project` | module or area (`+auth`, `+docs`) |
| `@context` | `@bug`, `@suggestion`, `@test`, `@docs`, `@perf` |
| `type:` | `bug` or `suggestion` |
| `src:` | who logged it (agent or tool name) |
| `needs:` | `none` if it can be done autonomously, `user` if it needs a decision, credentials, money, product direction or approval |

Never rewrite or reorder other lines in LOG mode. One line per item, no multiline text. If detail is long, put it in `notes/<slug>.md` and add `note:notes/<slug>.md`.

4. Say in one line what you logged, then continue your original task.

## Mode 2 - CULL

Goal: shrink the backlog safely without needing the user for everything.

0. **Pick the backlogs.** Run from inside a sub-repo, cull only that sub-repo's `TODO.txt`. Run from the monorepo root, cull the root `TODO.txt` and every sub-repo `TODO.txt` (find them with `find . -name TODO.txt -not -path '*/node_modules/*'`), one backlog at a time, each against its own scope file from Step 0. Stay inside the sub-repo you are culling; touching another one means the item is cross-cutting, so move it to the root backlog and ask.
1. **Load** the backlog. Open items are lines not starting with `x `. Sort by priority, then age.
2. **Triage each item** against the scope file:
   - Stale, already fixed, or out of scope: mark done with a reason (`x DATE ... resolved:stale`) or ask the user if unsure about dropping it.
   - `needs:none` and clear: implement.
   - `needs:user`, or you discover it really needs a decision: do not implement. Collect it for step 4.
3. **Implement the autonomous items**, one at a time:
   - Make the smallest change that fixes the item. Do not expand scope or refactor neighbours.
   - Run the project's tests/lint per the scope file. If they fail and you cannot fix it quickly, revert your change, keep the item open and add `blocked:<reason>`.
   - Mark done: prefix `x YYYY-MM-DD ` (completion date) before the original line, keep the rest intact, add `fix:<short note or commit>`.
   - Never commit, push or touch anything the scope file forbids unless it says you may.
4. **Ask about human items.** Use in-chat questions (the AskUserQuestion tool when available), batching up to 4 per call. Each question states the item, the options and your recommendation. Apply the answer immediately: implement, reprioritise, or drop with `x ... resolved:declined`.
5. **Report**: counts of done / blocked / waiting on the user, plus the list of changed files (per sub-repo in a monorepo). Keep it short.

## Rules of thumb

- Each `TODO.txt` is shared state. Edit only the lines you own for the current action, and re-read right before writing in case another agent appended.
- When unsure whether something is a bug or intended behaviour, log it as `needs:user` rather than guessing.
- Never delete lines, and never move one between backlogs except a cross-cutting item to the root. Completed items stay as `x` lines; the user archives them (e.g. `tuxedo archive`).
- Never put secrets or tokens in the backlog.
