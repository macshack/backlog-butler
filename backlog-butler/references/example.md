# Example: backlog-butler in action

A walkthrough on a small project called `shop-api`, which has this `AGENTS.md`:

```markdown
## Backlog scope

- Real bug: failing tests, crashes, wrong totals, security issues.
- Worth implementing: small fixes, missing tests, docs gaps.
- Out of scope: new dependencies, style preferences, redesigns.
- Needs user: API changes, deleting data, anything affecting pricing rules.
- Verify with usual tools and tests (`npm test`). If they fail, revert and mark `blocked:<reason>`.
```

## 1. Logging

An agent is adding a discount endpoint. While reading `src/cart.ts` it notices two things unrelated to its task and calls backlog-butler. It appends to `TODO.txt`:

```
(B) 2026-10-06 Cart total rounds per item instead of per order, off by 1 cent src/cart.ts:54 +cart @bug type:bug src:feature-agent needs:none
(C) 2026-10-06 Add test for empty cart checkout src/checkout.ts:12 +checkout @test type:suggestion src:feature-agent needs:none
```

The agent says: "Logged 2 items to TODO.txt." and returns to the discount endpoint.

Later, a reviewer agent finds the same rounding problem. The skill greps `TODO.txt`, finds the existing line, and adds `seen:2026-10-07` to it instead of a duplicate.

A third agent suggests changing the free-shipping threshold from 50 to 40. That is a pricing rule, so it is logged as a decision:

```
(C) 2026-10-07 Consider lowering free-shipping threshold from 50 to 40 src/shipping.ts:8 +shipping @suggestion type:suggestion src:review-agent needs:user
```

## 2. Culling

The user says: "cull the backlog".

The skill reads `AGENTS.md`, then `TODO.txt`, and triages:

| Item | Decision |
|---|---|
| Cart rounding (B) | In scope, `needs:none`: implement |
| Empty cart test (C) | In scope, `needs:none`: implement |
| Free-shipping threshold (C) | Pricing rule, `needs:user`: ask |

It fixes the rounding, adds the test, runs `npm test` (green), and marks both done:

```
x 2026-10-08 2026-10-06 Cart total rounds per item instead of per order, off by 1 cent src/cart.ts:54 +cart @bug type:bug src:feature-agent needs:none seen:2026-10-07 fix:round once on order total
x 2026-10-08 2026-10-06 Add test for empty cart checkout src/checkout.ts:12 +checkout @test type:suggestion src:feature-agent needs:none fix:added checkout.empty.test.ts
```

Then it asks in chat:

> **Free-shipping threshold**: `src/shipping.ts:8`, logged by review-agent. Lower from 50 to 40?
> - Yes, change to 40
> - No, keep 50 (recommended: this is a pricing rule, confirm with the business first)
> - Drop this item

The user picks "No, keep 50". The skill closes the line:

```
x 2026-10-08 2026-10-07 Consider lowering free-shipping threshold from 50 to 40 src/shipping.ts:8 +shipping @suggestion type:suggestion src:review-agent needs:user resolved:declined
```

Final report:

> Done: 3 (2 fixed, 1 declined). Blocked: 0. Waiting on you: 0.
> Changed: `src/cart.ts`, `test/checkout.empty.test.ts`.

## 3. A blocked item

Had `npm test` failed after the rounding fix, the skill would revert the change, leave the line open, and tag it:

```
(B) 2026-10-06 Cart total rounds per item instead of per order, off by 1 cent src/cart.ts:54 +cart @bug type:bug src:feature-agent needs:none blocked:fix breaks cart.discount.test.ts
```

It would then report it as blocked, so a human or a later run can pick it up.

## 4. No scope file

In a project with neither `AGENTS.md` nor `CLAUDE.md`, logging still works but tags the line:

```
(C) 2026-10-06 Rename misleading variable `tmp` in utils.py:31 +utils @suggestion type:suggestion src:claude needs:none scope:unchecked
```

Culling stops: the skill tells the user it cannot judge scope and offers to draft an `AGENTS.md` first.

## 5. Monorepo

`shop` is a monorepo with `apps/web`, `apps/api` and `packages/ui`. Root `AGENTS.md` exists; only `apps/api` has its own.

```
shop/
  AGENTS.md          <- fallback scope for web and ui
  TODO.txt           <- cross-cutting items
  apps/api/AGENTS.md <- scope for api
  apps/api/TODO.txt
  apps/web/TODO.txt
  packages/ui/TODO.txt
```

An agent working in `apps/web` finds a bug in `packages/ui/src/Button.tsx`. The file belongs to the `packages/ui` sub-repo, so the line goes in `packages/ui/TODO.txt`, with the path relative to that folder:

```
(B) 2026-10-06 Button ignores disabled prop src/Button.tsx:21 +ui @bug type:bug src:feature-agent needs:none
```

A finding about the CI workflow touches no single sub-repo, so it goes in the root `TODO.txt`.

On "cull the backlog" from the root, the skill walks each `TODO.txt` in turn:

| Backlog | Scope file used |
|---|---|
| `TODO.txt` | `AGENTS.md` (root) |
| `apps/api/TODO.txt` | `apps/api/AGENTS.md` |
| `apps/web/TODO.txt` | `AGENTS.md` (root fallback) |
| `packages/ui/TODO.txt` | `AGENTS.md` (root fallback) |

Run from inside `apps/api`, only `apps/api/TODO.txt` is culled. The report lists changed files per sub-repo.
