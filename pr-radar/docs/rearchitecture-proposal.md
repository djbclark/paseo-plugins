# PR Radar — re-architecture proposal (v1)

Date: 2026-11-19 · Status: **SUPERSEDED IN DETAIL by the Fable 5.2 design record**
(`~/src/.paseo-fanout/pp-rearch/report.md`, 2026-11-20). Read that first; this file keeps the
problem framing and the finding distribution, which still hold.

### Corrections the design record makes to this proposal

1. **Move 1's mechanism is wrong.** `ade-claim` cannot be asked for a worktree and cannot be
   imported (Paseo forbids plugin-to-plugin imports; `ade-claim` is a hook listener that claims
   whatever *Paseo* creates). The correct seam is: **pr-radar asks Paseo to create the workspace**
   (`paseo.workspaces.create` with `source.kind:"worktree"` + `checkoutSource:{kind:"change_request"}`
   + `idempotencyKey`), and ade-claim claims it through `worktree.create.after` if installed.
   Paseo 0.16 already does PR checkout with fork handling, hooks and idempotency.
2. **"~30 live workspace-ops findings" is stale.** 31 point at the file but only **5 are live** at
   head (F60, F63, F66, F67, F74) — the rest were fixed before merge. The case for Move 1 is
   deleting ~620 lines of security-critical git code + ~1,500 lines of tests, and killing 3
   defects that cannot be fixed locally.
3. **Move 2 is nearly done.** Ownership + parent-dir checks exist. 4 small defects remain
   (incl. one not in the inventory: the *configured* path's own symlink chain is unchecked).
4. **Move 3 should NOT be a component split.** 7 client defects are live, from one stale closure,
   one save queue and one duplicated code path. Extract logic, don't split components.
5. **~60 of the 90 findings are already closed** at head: ~60 fixed/outdated/declined → reply and
   resolve; 8 superseded by deletion; ~14 cherry-pick; 5 test additions; 2 belong to djbclark-ade.
6. **The synced adapter has drifted** from `~/src/djbclark-ade/bin/pr-radar-route` in both
   checkouts — a sync-process gap, its own chore PR.

---

## 1. Problem

PR Radar (the `pr-radar` plugin) is ~5,900 lines of TypeScript in its seven core
files, and its merged PR #618 drew ~90 bot-review findings (40 baz + ~50
coderabbit). The findings are a *signal*, not noise — they concentrate in three
architectural choices, in proportion to how hand-rolled each is:

| Subsystem | Lines | ~Findings | Root cause |
|---|---|---|---|
| `server/workspace-ops.ts` | 1,361 | ~30 | hand-rolled `git fetch/checkout/worktree/cleanup` |
| routing script trust | — | ~10–15 | `exec`s a user Python script with daemon privileges |
| client (`pr-radar.tsx` 1,314 + `radar.ts` 1,069 + `start-agent-modal.tsx` 706) | ~3,000 | ~15–20 | monolithic routing+launch+settings state machine |
| test-quality | — | ~15–20 | too many edge cases to cover hermetically |
| a11y / cosmetic | — | few | minor |

## 2. Target architecture (three moves)

1. **Delegate checkout + ownership to `ade-claim`, not hand-rolled git.** Use
   djbclark-ade's reusable pieces directly: the `plugins/ade-claim/` Paseo plugin
   (importable as a plugin dependency — `registerWorktreeClaimHook` claims a
   worktree on `worktree.create.after`, `registerWorkspaceArchiveReleaseHook`
   releases on archive, `claim-lib.mjs` `planWorktreeClaimCreate` builds the
   claim), the `bin/ade_claims.py` claim registry (new/list/bind-owner/successor-of
   /release/install; CLI-only), and `bin/ade-git-guard` (PATH shim + ref hooks) for
   one-writer enforcement. pr-radar asks for a claimed, prepared worktree and
   receives a path or a refusal, instead of reimplementing `server/workspace-ops.ts`
   (~1,361 lines). Kills ~30 findings.
2. **Keep the router; fix only the local-script trust.** Routing is already solved
   and must NOT be re-implemented: `bin/pr-radar-route` (djbclark-ade) reads
   pr-radar context JSON on stdin and emits launch overrides via
   `bin/route_agent.py route`; pr-radar's `examples/pr-radar-route.py` is a synced
   byte-identical copy that must stay so (`docs/model-routing.md` A2b,
   `tests/test_pr_radar_route.py`). It is stdlib-only and already constrained
   (reads only named fields, refuses template dispatches, emits nothing on failure).
   The residual ~10 security findings are pr-radar's own `resolveRoutingScript` /
   `runRoutingScript` (unverified ownership, parent-dir writability, concurrent-run
   guard) — fix those, or restrict the plugin to the synced preset, rather than
   re-sandboxing the router.
3. **Split the client monolith.** Separate viewer / routing / launch / settings
   into their own state machines (XState-style or plain reducers) instead of one
   700-line modal. Removes the race class (stale routing, double-launch,
   `ComboSelect` drop). Each split ships as its own reviewable PR.

## 3. Which move kills which findings

- **Delegated checkout** → `workspace-ops.ts` findings: shared `FETCH_HEAD`,
  stale-head reuse, `--no-checkout` sequencing, submodule/gitlink, concurrent
  prep, dead-worktree cleanup, CWD-BASE starts-from-stale-HEAD, XDG_CONFIG_HOME
  git config, hostname-handled-remotes.
- **Constrained routing** → routing-script trust findings: ownership never
  verified, parent-dir writability, guard released while running / before kill,
  remote-host ignored in repo matching.
- **Split client** → race findings: auto-launch guard released early, rapid
  `ComboSelect` edits, routed model after provider edit, stale routing after
  disable, routed profile dropped on uncached config.
- **Collapses with the surface** → most "test doesn't cover X" findings, because
  there is far less bespoke behavior to hermetic-test.

## 4. Execution plan

- **Phase 0 (now):** freeze the piecemeal 90-finding follow-up work ([#662](https://github.com/omercnet/paseo-plugins/pull/662) and [#618](https://github.com/omercnet/paseo-plugins/pull/618) review-follow-ups are superseded by this). Queue this proposal.
- **Phase 1 (Fable 5.2, architect):** after the Claude/Fable weekly reset (~2026-11-20T10:00Z), Fable produces the detailed design — the checkout-delegation interface against `@getpaseo/plugin`/Paseo's APIs, the routing-sandbox/allowlist contract, and the component split.
- **Phase 2 (implement, sliced):** fan out small PRs per move, each validated locally (install deps + typecheck + vitest + biome via `bg`).
- **Phase 3 (verify):** re-run bot review; expect findings to drop by an order of magnitude.

## 5. Risks / unknowns to resolve in Phase 1

- Does `@getpaseo/plugin` expose enough for the plugin to *request* a worktree
  from Paseo, or does checkout stay outside the plugin and we hand it a cwd?
- The routing script is a real cross-repo dependency (djbclark-ade
  `route_agent.py`): the sandbox/CLI-trust decision must not break that reuse.
- Whether Fable can produce the whole design in one architect session, or should
  hand off per-move to implementation workers.

## 6. Decision

djbclark (2026-11-19): leaning to wait for the Fable 5.2 reset, then re-architect
the whole thing rather than grind through ~90 review findings.

## 7. Confirmed reuse (djbclark-ade lead, 2026-11-19)

"Most of this exists; do not fork it." Piece → reuse map:

- Routing (move 2) → `bin/pr-radar-route` (CLI invocation, not an import); keep the synced `examples/pr-radar-route.py` byte-identical.
- Worktree + ownership (move 1) → `plugins/ade-claim/` (importable Paseo plugin) + `bin/ade_claims.py` (CLI registry) + `bin/ade-git-guard` (CLI/hook). Reference docs: `docs/git-guard.md`, `docs/workspace-layout.md`, AGENTS.md order 9.
- Claim/origin boundary → `bin/ade_send_guard.py` + `docs/archived-send-guard.md` (send-path only).
- Who already owns "route a PR/issue to an agent in a prepared worktree" → pr-radar itself via `bin/pr-radar-route`; nothing else does it end-to-end.