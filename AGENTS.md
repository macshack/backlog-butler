# AGENTS.md

Guidance for AI agents working in this repo.

## What this repo is

The source of the `backlog-butler` Claude skill. The skill folder is `backlog-butler/`:

- `backlog-butler/SKILL.md`: the skill itself (the deliverable).
- `backlog-butler/references/example.md`: worked walkthrough of log and cull modes.

`README.md` at the repo root documents it for humans. There is no build step or application code.

## Rules for editing the skill

- `backlog-butler/SKILL.md` needs YAML frontmatter with `name: backlog-butler` and a `description`. All trigger conditions live in the description, not the body.
- Keep the body under 500 lines. Explain why, not just what.
- The backlog format is standard todo.txt plus these key:value tags: `type:`, `src:`, `needs:`, `scope:`, `seen:`, `blocked:`, `fix:`, `resolved:`, `note:`. If you add or rename a tag, update `backlog-butler/SKILL.md`, `backlog-butler/references/example.md` and `README.md` together.
- Monorepo behaviour (one `TODO.txt` per sub-repo, scope file falls back from sub-repo to root, cross-cutting items go to the root backlog) is described in `SKILL.md` Step 0, `README.md` and the example's section 5. Keep the three in sync.
- The skill must stay safe by default: never delete backlog lines, never implement without a scope file, never commit unless told.

## Backlog scope (applies when the skill runs on this repo)

- Real bug: `backlog-butler/SKILL.md` frontmatter invalid, contradicts `README.md` or the example, or instructs unsafe behaviour (deleting lines, committing unasked).
- Worth implementing: clearer instructions, missing edge cases (concurrent writes, missing scope file), format fixes.
- Out of scope: adding scripts or a runtime, support for backlog formats other than todo.txt, renaming the skill.
- Needs user: changes to the tag vocabulary, priority meanings, or when autonomous culling is allowed.
- Verification: no tests. Check by re-reading `backlog-butler/SKILL.md` and, for substantial changes, running the skill on a scratch project with a sample `TODO.txt`.
- Do not commit or push.
