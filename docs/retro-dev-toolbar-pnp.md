# Retro: dev server unhandled rejection (`astro:toolbar:internal`)

## Summary

`yarn dev` threw an `UnhandledRejection` on every startup:

```
[RESOLVE_ERROR] Could not resolve 'astro:toolbar:internal'
... astro tried to access astro:toolbar:internal, but it isn't declared in
its dependencies; this makes the require call ambiguous and unsound.
```

The dev server kept running and pages still returned 200, but every dev
session logged a scary top-level error and the failure mode wasn't obvious
from the message alone.

**Fix**: added `vite.optimizeDeps.exclude: ['astro:toolbar:internal']` to
`astro.config.mjs`, keeping the dev toolbar enabled and avoiding a full
version pin/downgrade.

## Timeline

| Date (assumed) | Commit | Change | Effect |
|---|---|---|---|
| — | `67ccc5b` — "fix(deps): update dependency astro to v7 [security] (#152)" | Renovate bumped `astro` 6.3.6 → 7.1.0 | **Regression introduced here.** Astro 7 pulls in Vite 8, whose default dependency optimizer is rolldown-based. |
| — | `a8c5720`, `4b645bb`, `a68fe21`, `728c717`, `0ff42f7`, `c24a67e` | Further routine dep bumps (astro → 7.1.1, typescript → v7, misc transitive bumps) | Bug persists unnoticed (none of these ran/verified `yarn dev`). |
| 2026-08-18 | `a27a3ee` — "fix(deps): update astro monorepo" | Renovate bumped `astro` 7.1.1 → 7.2.3 | Bug persists (confirmed independent of this bump — see Bisection). |
| 2026-08-18 | `e651a59` — "chore(deps): bump" | Manual commit walked `astro` back 7.2.3 → 7.2.2, alongside unrelated transitive dep bumps; no message explaining the downgrade | Bug persists; likely someone hit an issue with 7.2.3 and reverted, but the reasoning wasn't recorded. |
| 2026-08-19 | (this session) | User ran `yarn dev`, saw the unhandled rejection in logs | Investigated, bisected, fixed. |

## Bisection

Tested `yarn dev` against isolated worktrees at each relevant commit:

| Astro version | Commit | Result |
|---|---|---|
| 6.3.6 | `709b0d3` | **Clean** — no unhandled rejection |
| 7.1.1 | `a68fe21` | **Broken** — same `RESOLVE_ERROR` |
| 7.2.2 (HEAD) | `e651a59` | **Broken** — same `RESOLVE_ERROR` |

Conclusion: the regression was introduced by the astro 6.3.6 → 7.1.0 major
bump (`67ccc5b`), not by the branch's own 7.1.1 → 7.2.x changes. Every astro
7.x release tested reproduces it. The later `a27a3ee`/`e651a59` commits are
incidental — the bug was already present when this branch started.

## Root cause

Astro's dev toolbar registers a virtual module, `astro:toolbar:internal`,
resolved via a Vite plugin's `resolveId` hook
(`node_modules/astro/dist/toolbar/vite-plugin-dev-toolbar.js`). That hook is
always registered by `astro/dist/core/create-vite.js` regardless of whether
`devToolbar.enabled` is set — confirmed experimentally (disabling the toolbar
did not fix the error).

Astro 7's Vite dependency moved to Vite 8, whose default dependency optimizer
uses **rolldown** for its pre-bundling scan. During that scan phase, rolldown
resolves bare import specifiers itself rather than routing every one through
Vite's plugin `resolveId` chain the way the older esbuild-based optimizer did.
Under Yarn **PnP** (this project's linker), that raw resolution attempt hits
PnP's strict resolver, which refuses anything that isn't a declared package
dependency on disk — producing the "isn't declared in its dependencies"
error. Astro's own dev server also warns: *"Using Yarn PnP with Vite is
discouraged and PnP-specific bugs will no longer be actively worked on."*

So the failure is a three-way contract gap: Astro's toolbar assumes Vite
routes virtual-module resolution through plugins during optimization scans;
Vite 8's rolldown-based scanner doesn't always do that; and Yarn PnP has zero
tolerance for resolution attempts outside the declared dependency graph. None
of the three components is "wrong" in isolation.

## STAMP control structure

Controllers/processes involved, and the control/feedback loops that were
missing:

- **Renovate bot** — proposes dependency version bumps (control action: open
  PR with new `astro`/`typescript` versions). Feedback it receives: none
  beyond CI status (if CI doesn't boot the dev server, Renovate has no signal
  that a bump breaks `yarn dev`).
- **CI / merge gate** — the control that should catch "does the app still
  run" before a bump lands. No feedback loop existed for `astro dev`
  specifically; presumably only `astro check`/`astro build` (if anything) ran.
- **Human merging dep-bump PRs** — approves/merges Renovate PRs, and in this
  case also made a manual, unexplained version-pin correction
  (7.2.3 → 7.2.2). No record of *why* was left, so the corrective action
  itself is now undocumented process knowledge.
- **Yarn PnP linker** — enforces strict resolution; behaves correctly per its
  own contract, but that contract is stricter than what Vite 8/rolldown's
  scanner assumes.
- **Astro dev-toolbar plugin / Vite optimizer** — the components whose
  hook-routing assumption silently broke under PnP.

## STPA-style unsafe control actions

- **UCA1**: Renovate provides a dependency-bump control action (PR) without
  any accompanying feedback that `yarn dev` still boots — a missing
  verification loop between "dependency proposed" and "dependency safe to
  merge."
- **UCA2**: The human control action of merging `e651a59` (downgrading
  7.2.3 → 7.2.2) was taken without recording the triggering observation
  (what broke at 7.2.3?) — feedback that would have shortened this
  investigation was discarded.
- **UCA3**: No control action exists to reconcile "Yarn PnP is used" with
  "Vite 8/rolldown is now a transitive dependency of the framework" — nobody
  owns checking new tooling against the project's chosen package manager
  before it lands.

## Contributing factors

- No CI/pre-merge smoke check runs `astro dev` (or equivalent headless boot
  check) — only `astro check`/`astro build`, if that, would run, and neither
  exercises the dev server's optimizer path the way `astro dev` does.
- The manual `chore(deps): bump` commit that reverted `astro` 7.2.3 → 7.2.2
  had no message explaining why, so intent was lost.
- Astro's own upstream warning ("PnP-specific bugs will no longer be actively
  worked on") means this class of bug is not likely to be fixed upstream —
  it needs to be worked around locally, and that workaround needs to survive
  future astro bumps.

## Recommendations

1. Add a minimal smoke test to CI (and ideally to Renovate's PR checks) that
   boots `astro dev` headlessly (or runs `astro build`) and fails the PR if
   it throws. This is the single highest-leverage fix — it would have caught
   this at the very first `astro` 7.0 bump.
2. Require commit messages on manual dependency corrections (like the
   7.2.3 → 7.2.2 downgrade) to state what was observed/tested, not just the
   version delta.
3. Keep the `vite.optimizeDeps.exclude: ['astro:toolbar:internal']` workaround
   in `astro.config.mjs`; re-verify it on every future astro major/minor bump,
   since upstream isn't planning to fix the underlying PnP interaction.
4. Given upstream's explicit "PnP is discouraged" stance for Vite-based
   tooling, consider (separately, not urgent) whether staying on Yarn PnP is
   worth the recurring friction versus `nodeLinker: node-modules`.
