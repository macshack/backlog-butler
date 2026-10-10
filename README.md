# backlog-butler

A Claude skill that keeps a project's backlog in a plain `TODO.txt` and works it down for you.

Agents notice things all the time: a flaky test, a typo in the docs, an error in a log, a nicer way to structure a module. Fixing them mid-task derails the work; ignoring them loses them. backlog-butler gives them somewhere to go, and gives you a way to clear that list later.

## Two modes

**Log.** Any agent calls the skill when it spots a bug or has a suggestion outside its current task. The skill appends one line to `TODO.txt` at the project root and the agent carries on.

**Cull.** You (or an agent with spare time) call the skill to work through the backlog. It implements items that need no human input, runs the project's checks, marks them done, and asks you in chat about anything that needs a decision.

## Install

Copy the `backlog-butler/` folder from this repo into a skills folder:

```
~/.claude/skills/backlog-butler/              # all projects
<project>/.claude/skills/backlog-butler/      # one project
```

## Set up a project

The butler needs to know what really counts as a bug and what is worth building. Add an `AGENTS.md` or `CLAUDE.md` to the project root. A short `## Backlog scope` section is enough:

```markdown
## Backlog scope

- Real bug: failing tests, crashes, wrong output, security issues.
- Worth implementing: small fixes, missing tests, docs gaps.
- Out of scope: new dependencies, redesigns, style preferences.
- Needs user: API changes, deleting data, anything costing money.
- Verify with usual tools and tests. If they fail, revert and mark `blocked:<reason>`.
- `SHOULD` use the backlog-butler SKILL if possible to handle adding issues to the backlog and culling them.
```

Without this file the butler still logs (flagged `scope:unchecked`) but refuses to cull.

## Monorepos

In a monorepo, each sub-repo (a package or app with its own manifest or `AGENTS.md`/`CLAUDE.md`) gets its own `TODO.txt`. Findings go in the backlog of the sub-repo that owns the file; cross-cutting ones go in the root `TODO.txt`.

Scope file lookup per backlog: the sub-repo's own `AGENTS.md`, then its `CLAUDE.md`, otherwise the root `AGENTS.md`, then `CLAUDE.md`. Culling from the root walks every backlog in turn; culling from inside a sub-repo handles only that one. See `backlog-butler/references/example.md`.

## TODO.txt format

Standard [todo.txt](http://todotxt.org/), so tools like [tuxedo](https://github.com/webstonehq/tuxedo) or todo.txt-cli can read it:

```
(B) 2026-10-06 Fix off-by-one in pager src/list.ts:88 +ui @bug type:bug src:code-reviewer needs:none
(C) 2026-10-06 Cache parsed config between requests +core @perf type:suggestion src:claude needs:user
x 2026-10-07 2026-10-06 Fix off-by-one in pager src/list.ts:88 +ui @bug type:bug needs:none fix:clamp index
```

| Tag                                   | Meaning                                                            |
| ------------------------------------- | ------------------------------------------------------------------ |
| `(A)`..`(D)`                          | A breaks build/data/security, B bug, C improvement, D nice-to-have |
| `type:`                               | `bug` or `suggestion`                                              |
| `src:`                                | which agent logged it                                              |
| `needs:`                              | `none` (autonomous) or `user` (needs a decision)                   |
| `scope:unchecked`                     | logged with no AGENTS.md/CLAUDE.md                                 |
| `blocked:` `fix:` `resolved:` `note:` | set during culling or for long notes                               |

## Usage

- Agents: triggered automatically when they find something out of scope. You can also say "log that to the backlog".
- You: "cull the backlog", "run the butler", "work through the TODOs".

## Safety

Never deletes backlog lines (done items become `x` lines). Never commits or pushes unless the scope file allows it. Reverts and marks `blocked:` when checks fail. Asks before dropping anything it is unsure about.

## Repo layout

| File                                   | Purpose                                                        |
| -------------------------------------- | -------------------------------------------------------------- |
| `backlog-butler/SKILL.md`              | The skill                                                      |
| `backlog-butler/references/example.md` | Worked example: logging, culling, blocked items, no scope file |
| `AGENTS.md`                            | Rules for agents editing this repo                             |
| `webstonehq-tuxedo-*.txt`              | Reference dump of tuxedo, for todo.txt conventions             |
