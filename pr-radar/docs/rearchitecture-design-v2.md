# PR Radar re-architecture: design record v2

Status: supersedes v1 (`/Users/djbclark/src/.paseo-fanout/pp-rearch/report.md`). Written 2026-11-20. Design only;
no repository was changed.

Base: pr-radar at `9f1f053`, the head of open PR
[omercnet/paseo-plugins#662](https://github.com/omercnet/paseo-plugins/pull/662) (upstream `main` + 3 commits, +96/−1).
The #662 rework is in flight in that same checkout: `client/launch.ts` (`launchNewAgent`, `applyAutoRouting`) and
`shared/routed-labels.ts` are new and uncommitted there, so they may still change. Line numbers below are at `9f1f053`
unless marked. Paths are relative to `pr-radar/` unless they start with `/` or `~`.

Sources: v1 record and `client-races.md`; adversarial review (`pp-rearch-review/report.md`, cited as "Rev §E*n*");
independent validation (`pp-argon-validation/report.md`, cited as "Val §V*n*"); the maintainer's comments on #618,
#619 and #662; Paseo source at `v0.12.0` … `v0.16.0-beta.2` (local tags) and at `v0.16.2` and `v0.17.0-beta.1`
(not tagged locally; four files read from raw.githubusercontent.com at those tags: `github-service.ts`,
`creation/index.ts`, `resolve-worktree-creation-intent.ts`, `change-request-checkout.ts`).

Prior art / docs checked: shared Firecrawl search and scrape, 2026-11-20.
- [getpaseo/paseo#3220](https://github.com/getpaseo/paseo/issues/3220) (closed 2026-09-08, "addressed by #2997"):
  PR worktree checkout fetched `refs/pull/N/head` from `origin` in a fork checkout. The reporter proposed matching the
  fetch remote by repository identity; the fix that shipped tries `origin` and then `upstream`
  (`change-request-checkout.ts:6-7`, unchanged at `v0.16.2` and `v0.17.0-beta.1`). The lookup still ignores the
  fork parent. The gap described in Move 1 below is the residue of that fix.
- [getpaseo/paseo#9340](https://github.com/getpaseo/paseo/issues/9340) (open): Create PR in a fork checkout targets
  the fork. Same class (fork parent ignored), different code path.
- [getpaseo/paseo#9534](https://github.com/getpaseo/paseo/pull/9534) (merged 2026-11-13): the Paseo app's own drafts
  hit `workspace_request_key_conflict` / `agent_request_key_conflict` by reusing one key across edited retries; the
  fix keys on draft id + request hash. Its review records that a resend after an outcome-unknown start creates a
  second agent. Follow-ups: [#9713](https://github.com/getpaseo/paseo/pull/9713) (open PR, "prevent duplicate
  creation after uncertain starts") and [#9715](https://github.com/getpaseo/paseo/issues/9715) (open issue).
- No getpaseo/paseo issue was found for "`getPullRequestCheckoutTarget` ignores the fork parent", or for "a fork's
  own PR #N is fetched instead of the parent's". Both are candidates for a new upstream issue (Move 1, B.1).

## What changed from v1, and why (ranked)

1. **The fork story was wrong (Rev §E2 facts 1–2, Val §V2a–b).** `checkoutSource.projectPath` is accepted and
   dropped; it pins nothing (still true at `v0.17.0-beta.1`, which only adds a forge check). Paseo resolves the PR
   with `gh repo view` and ignores `parent` (`github-service.ts:2139-2141` at `v0.16.2`, `:2407-2409` at
   `v0.17.0-beta.1`), then fetches `origin` before `upstream`. v2 adds a fork rule, a post-create head check, and an
   upstream Paseo issue.
2. **The idempotency key cannot replace the UI lock (Rev §E5, Val §V2c, §V4.1).** Every workspace-stage failure
   except `directory_not_found` / `source_required` is stored as `outcomeUnknown` (`creation/index.ts:201-214`,
   identical at `v0.16.2` and `v0.17.0-beta.1`). Any reuse of a key after a failed attempt returns the stored failure.
   v2: one fresh key per launch execution, used only for transport recovery. One synchronous lock, mirrored into
   state, guards both launch paths.
3. **Census corrected (Rev §E1, Val §V1).** Twelve v1 verdicts were wrong; Val adds F52 and F06/F24 as live and
   shows that F34/F35 are fixed by the standalone example. The tally is now 36 open rows (C.1).
4. **No Paseo floor raise (Rev §E2, Val §V3).** Everything pr-radar would call exists at `v0.12.0`, and the
   `v0.12.0` daemon checkout code matches `v0.16.0-beta.2`. Workspace metadata needs `entityMetadata` (0.16+), so
   v2 treats it as optional.
5. **Move 3 fixes rewritten (Rev §E7, Val §V3).** The `saving`-flag queue replaces "wait for `settings.revision`",
   which would deadlock. `ComboSelect` gets a commit mode as a prop. `resolveRoutedLaunch` follows three rules,
   plus two from Val: clear the model when a routed profile changes the effective provider, and vet the profile at
   launch against fresh config.
6. **Do not re-sync `examples/pr-radar-route.py` (Val §V4.3).** It is a deliberate standalone example; the ADE
   docstring is what is wrong. Ruled and in flight.
7. **The plan now builds on in-flight work.** The #662 rework, the #618 follow-ups PR, the F41 PR, the ADE F39 fix
   and Move 2 are named, and nothing below duplicates them. Two v1 joins are now one PR each: 1+2 (Move 2, in flight)
   and 6+7 (native worktree for PR and issue rows, PR 9).
8. **Real proofs (Rev §E8).** The deletion PR gets checks for the routing stdin contract, dead private helpers, docs
   text and old clients, in addition to typecheck.
9. **New operator decisions surfaced (Rev §E9).** Semver, migration, hosts without metadata, the fork fallback, and
   ade-claim rebind as a blocker for the operator's own setup.

## A. Where things stand

- `server/workspace-ops.ts` hand-rolls every git step: env allow-lists, `GIT_SAFETY_ARGS` with per-call
  `filter.*`/`hook.*` neutralizing, clone, PR fetch into `refs/pr-radar/pull/N`, fast-forward, ignored-file and
  gitlink guards, CWD-BASE branch switch, worktree create/reuse/refresh/cleanup, and the routing-script runner
  (`:39-1361`). Most of the 31 findings on it are fixed in place. Still open: F52, F60, F61 (PR path), F63, F65
  (residue), F66, F67 and F74.
- Routing trust is mostly closed (F70, F71 and F73 fixed). Open: F74, plus two defects in no inventory (C5 symlink
  hops, C7 rejection instead of queueing). All are in Move 2.
- Client: the races come from one stale closure (`profiles`), the settings save chain, and routing-then-launch logic
  written twice (`client-races.md` §4). The #662 rework already pulls launch into `client/launch.ts`.
- Threat model, once: the routing checks defend against other local users. They do not defend against an
  authenticated Paseo client, which can already start agents and terminals.

## B. Move 1: delegate checkout to Paseo's `workspaces.create`

### B.0 Seam (unchanged from v1)

```text
pr-radar client ── rpc resolveBaseCheckout ──▶ pr-radar server  (read-only git; optional clone under a lock)
pr-radar client ── paseo.workspaces.create ──▶ Paseo daemon     (lookup, fetch, worktree add, branch, agent)
                                                 │ workspace.create.before / worktree.create.before/after / …
                                                 ▼
                                             other plugins, e.g. a claim or placement plugin
```

pr-radar names no other plugin. `workspaces.create` exists on the plugin client and on `PluginHandlerContext.paseo`
at every tag from `v0.12.0` (Rev §E2). A third plugin may rewrite the request in `workspace.create.before`; pr-radar
must not assume its request arrives unchanged.

### B.1 Fork checkouts: what Paseo actually does

For `source: { kind: "worktree", action: "checkout", checkoutSource: { kind: "change_request", number } }`:

1. **Lookup.** `getPullRequestCheckoutTarget({ cwd, number })` runs `gh repo view --json owner,name,parent` in `cwd`
   and queries PR N on `repo.owner/repo.name`. **It ignores `parent`**, unlike `resolveCurrentPullRequestView`, which
   reads it (Val §V2b; same at `v0.16.2` `:2132-2146` and `v0.17.0-beta.1` `:2400-2414`). If `gh` resolves `cwd` to
   the fork, the lookup asks the fork: either "Pull request not found", or the fork's own PR N.
2. **Fetch.** `checkoutRefs` is `[origin refs/pull/N/head, upstream refs/pull/N/head]` and the first success wins
   (`change-request-checkout.ts:6-7`, unchanged through `v0.17.0-beta.1`). If the fork has its own PR N, `origin`
   succeeds and the **fork's** PR is checked out without any error. A matching remote named anything other than
   `origin`/`upstream` fails.
3. **`projectPath`** is copied by `resolveInputChangeRequest` and dropped by all three call sites. It is a no-op at
   every version read. Do not send it as a pin.
4. **Local branch.** The local branch is `source.localBranchName ?? headRef`, made unique by suffixing. It is never
   `pr-radar/pr-N` (Rev §E2 fact 2).

pr-radar prefers `upstream` over `origin` (`server/workspace-ops.ts:417-421`). The two disagree in exactly the
fork+upstream topology.

**pr-radar's fork rule (to ship in PR 9):**

- `resolveBaseCheckout` returns `{ baseRoot, matchingRemote, originRepository, cloned }`.
- **Native path** only when `matchingRemote === "origin"`, i.e. `origin` is the row's repository. Here the lookup
  and the first fetch both target the right repository.
- **Fork topology** (`matchingRemote !== "origin"`): do not call native `change_request` checkout. The spike (B.4,
  S2/S3/S8) chooses between:
  - (F-a, preferred if S8 passes) pr-radar's server fetches `refs/pull/N/head` from `matchingRemote` into its private
    ref `refs/pr-radar/pull/N`. This fetch already exists and touches no work tree or branch. pr-radar then asks
    Paseo for `action: "checkout"` of that ref with an explicit `branchName`. Paseo still creates the worktree, so
    other plugins' hooks still run.
  - (F-b) Refuse with a clear message ("this checkout's `origin` is a fork; PR N lives on `upstream`") and offer
    opening the base checkout instead.
  - Not chosen: a pr-radar `worktree.create.before` hook that rewrites `checkoutRefs`. It fixes only step 2, not the
    lookup, and it would force a `^0.15` floor.
- **Post-create head check (all paths).** After creation, compare the worktree `HEAD` with the PR's `headRefOid`.
  pr-radar already has it from `gh`; the check is a read-only `rev-parse HEAD` RPC. On a mismatch, show "Paseo
  checked out a different commit than PR N's head". Archive nothing automatically. This catches the silent
  fork-PR-N case and any third-plugin rewrite.
- **Upstream.** File a getpaseo/paseo issue: "PR checkout in a fork checkout ignores the parent and can fetch the
  fork's own PR N". Cite #3220 (its proposed identity-matching fix) and #9340 (same class). When Paseo fixes it,
  F-a can go for hosts that have the fix.

### B.2 Idempotency key and UI lock

- **Lock.** One `useStartAgent` instance, owned by `PrRadar`, guards both the modal Start and the auto path. It is a
  synchronous `useRef` lock mirrored into `useState`, so every Start button shows "Starting…" during routing (F05),
  and `navigation.openAgent` runs once. It wraps the #662 rework's `launchNewAgent` (`client/launch.ts:77`). It does
  not add a second launch function.
- **Key.** Mint `pr-radar:<random uuid>` **per launch execution**, after the lock is taken. Pass it to
  `workspaces.create` only (PR 9). It exists only so `CreationClient.reconnect()` resumes a creation whose socket
  dropped (Val §V2c). It is never reused across clicks, never derived from content, and never sent on a legacy
  `agents.create` (PR 5 adds no key).
- **Why not reuse.** After any settled workspace-stage failure, the daemon persists `outcomeUnknown: true` and returns
  that stored failure for the same key forever (`creation/index.ts:76`, `:201-214`). With an edited retry it returns
  `workspace_request_key_conflict` instead. Paseo's own app hit the same thing ([#9534](https://github.com/getpaseo/paseo/pull/9534)).
- **Accepted consequence.** A retry after an outcome-unknown failure is a new creation and can make a second worktree.
  The pre-create reuse lookup (B.3) finds the first one if it was registered. Watch upstream
  [#9713](https://github.com/getpaseo/paseo/pull/9713) / [#9715](https://github.com/getpaseo/paseo/issues/9715).
- **What the key does not cover.** Routing, `resolveBaseCheckout`, `setLabel` and navigation run once per execution
  and are guarded only by the lock.

### B.3 Reuse, association, metadata

- **Reuse is a lookup by record, before create.** `workspace.create` never deduplicates by directory (protocol
  comment, Rev §E2 fact 3). The lookup runs in this order:
  1. On hosts with `entityMetadata`: a client-side scan of `workspaces.list` pages for `metadata["pr-radar.item"]`
     equal to `github:<owner>/<repo>#<kind>:<n>`. There is no server filter.
  2. On every host:
     - PR rows: the existing head-ref association (Paseo names the branch after `headRef`).
     - Issue rows: branch `pr-radar/<owner>/<repo>/issue-<n>`, optionally with a `-<k>` suffix.
     - Legacy branches: `pr-radar/issue-<n>` and `pr-radar/pr-<n>`, matched only in a workspace whose repository
       matches the row.
- **Hosts without `entityMetadata` (0.12–0.15).** Omit `metadata` from the request (S4 checks whether an old daemon
  rejects it or ignores it). Rely on step 2. Paseo does not rename a branch it was given explicitly
  (`paseo-worktree-service.ts:290-292`, Val §V4.5), so the issue branch stays matchable. Recommendation: no legacy
  path on these hosts. Accept that a suffix collision can show an extra unlinked workspace.
- **Issue branch reuse is lost by default.** `branch-off` always suffixes on a collision (Rev §E2 fact 6). If S8 shows
  that `action: "checkout"` of an existing local branch works, pr-radar uses that when `pr-radar/<o>/<r>/issue-<n>` (or
  a legacy `pr-radar/issue-<n>`) exists and has no live workspace. Otherwise it branches afresh and says so in the
  launch notice.
- **Reused workspace with only archived agents.** With a claim plugin in enforce mode, a new agent there may get no
  write identity (Rev §E3). pr-radar's rule: reuse a workspace only while it is not archived. If its agents are all
  archived, offer "start in a new worktree" as the primary action. The rebind rule itself belongs to the claim plugin
  (decision Q4).
- **Directory picker.** `listLocalDirsHandler` shows worktrees only under `sourceCodeDir` (`:521`), and Paseo places
  them elsewhere. In PR 9 the picker adds the Paseo workspaces whose repository matches the row (from
  `workspaces.list`).

### B.4 Spike (not merged; gates PR 9)

Setup: isolated daemons at **0.12.x (lowest supported)** and **0.16.2**, plus 0.17.0-beta.1 if it is cheap, each with
explicit `--home`/`--host`. Four checkouts:
- (a) a plain clone, `origin` = the row's repository;
- (b) `origin` = fork, `upstream` = parent, no `gh repo set-default`;
- (c) as (b), plus `gh repo set-default upstream`;
- (d) as (b), where the fork has its own PR with the same number N.

A throwaway test plugin throws in `worktree.create.after`.

| # | Question | Pass | Fail → consequence |
|---|---|---|---|
| S1 | Plugin client `workspaces.create` with `change_request` in (a) | worktree `HEAD` = PR `headRefOid`; branch = `headRef`; workspace registered | Move 1 stops; keep the legacy path |
| S2 | Which repo does `gh repo view` resolve in (b) and (c)? Outcome of native checkout | recorded either way; native path correct in (c) | (b) wrong or failing → fork rule needed (expected) |
| S3 | (d) native checkout | silent wrong checkout reproduced, **and** the post-create head check flags it | the head check is not a sufficient guard → F-b only |
| S4 | `metadata` on create: 0.16.2 stores it and `list` returns it; 0.12.x rejects or ignores it | 0.16.2 round-trips; 0.12.x behaviour known | if 0.12.x rejects, gate `metadata` on the capability |
| S5 | Same key twice concurrently → one creation; fresh key after a failed attempt → runs again; same key after a failed attempt → stored failure | all three as stated | pr-radar's key rule changes |
| S6 | Rejection text and `onEvent` phases when `worktree.create.after` throws | message starts `Plugin <id> worktree.create.after failed:`; worktree removed; phases seen | rendering in PR 9 keys on what is seen |
| S7 | Claim plugin (enforce mode) + reuse of a workspace whose first agent is archived | new agent gets a write identity | blocks reuse in the operator's setup until Q4 is settled |
| S8 | `action: "checkout"` with `refName` = an existing local branch, and with `refs/pr-radar/pull/N`, plus explicit `branchName` | worktree on that commit, no suffix for an existing branch | F-a dropped → F-b; issue-branch reuse dropped |
| S9 | Plugin RPC timeout vs the 120 s clone in `resolveBaseCheckout` | clone completes or the timeout is known | clone moves out of the RPC or gets a progress/poll shape |

Pass for Move 1 overall: S1, S4, S5 and S6 pass on both daemons. The fork path ships as F-a if S2, S3 and S8 pass,
else as F-b. Record a half-page of notes in PR 9's body.

### B.5 Refusals and failure UX

- Match the documented prefix `Plugin <id> <hook> failed:` and show "Blocked by the `<id>` plugin. PR Radar created
  nothing." with the raw detail below it.
- Keep the modal open with every field intact. The modal gets its own error, not the page-wide `openError` (R5).
- Pass `onEvent` and show the phase. Creation can take a fetch plus a 30 s hook.
- Never fall back silently to an unclaimed directory. "Open the base checkout instead" is an explicit choice.

### B.6 Deleted, kept, moved (`server/workspace-ops.ts`)

| Lines | Fate |
|---|---|
| `:578-591` (`prRef`, `fetchPrHead`) | **Keep** if F-a ships (fork fetch into the private ref); else delete |
| `:593-891` (transport guard, fast-forward, ignored-file and gitlink guards, issue start point, refresh) | delete |
| `:893-1153` (`prepareWorkspaceCwdHandler`) | replaced by `resolveBaseCheckout` (~60 lines) plus the per-target clone lock |
| `:438-444` (`resolveGitCommonDir`) | delete (dead with the handler; Rev §E4) |
| `:217-284` (dynamic filter neutralizing) | delete only if F-a is dropped; the fork fetch and the clone's `checkout --force` (`:305`) still run git. Say in the PR why the static `GIT_SAFETY_ARGS` suffice for a fresh clone |
| `:344-382` (worktree-name helpers) | `worktreeName` stays and is passed as `worktreeSlug` (Rev §E2 fact 5); keep validation |
| `:384-436`, `:446-576`, `:39-131`, `:189-210`, `:294-306` | keep |
| `:1155-1361` (routing) | moved by Move 2 to `server/routing-script.ts` |

### B.7 Version floor

Keep `"paseo": "^0.12.0 || ^0.13.0 || ^0.14.0 || ^0.15.0 || ^0.16.0"`. Add `^0.17.0` only after the plugin's checks pass
on a 0.17 daemon. Raise the floor only if a hook-based fallback is ever chosen; F-a and F-b need none.

### B.8 Stale head (behaviour change)

The daemon no longer refreshes a reused PR worktree (F53's fix goes away). "Start New Agent" on a PR with new commits
then starts on old code unless the agent fetches. Mitigation (PR 11): the row shows "worktree is N commits behind PR
head" (read-only `rev-list --count HEAD..refs/pr-radar/pull/N` after a fetch into the private ref, or a compare of
`HEAD` with `headRefOid` when no fetch is wanted). The launch prompt says the PR head SHA and tells the agent to fetch
if it differs. State this to the maintainer up front (issue draft).

## C. Census (F01–F90), final classification

Columns: final class at `9f1f053` (FIXED / LIVE / PARTIAL / TEST-GAP / DECLINED / OUTDATED / ADE) and disposition.
In-flight work: **#662-rw** (the #662 rework), **618-fu** (the #618 follow-ups PR), **F41-PR**, **ADE-fix**,
**M2** (Move 2). Planned PR numbers refer to section F.

### C.1 Table

| F | Topic | Final | Disposition / evidence |
|---|---|---|---|
| F01 | `ComboSelect` draft reset by older ack | LIVE | PR 6 |
| F02 | dropdown hides selected | FIXED | `client/agent-launch-options.tsx:237` |
| F03 | directory-observation helper tests | TEST-GAP | PR 8 |
| F04 | swallowed `setLabel` | FIXED | `client/radar.ts:846-852`; identity moves to metadata in PR 9 |
| F05 | no busy state during auto routing | PARTIAL | PR 5 (lock mirrored to state) |
| F06 | provider/profile routing drift; model kept across provider | LIVE (Val §V1.2-2) | PR 3 |
| F07 | routing warning lost | FIXED | `client/pr-radar.tsx:291-297` |
| F08 | auto guard released early | FIXED | `client/pr-radar.tsx:409-423` |
| F09 | action group width | LIVE | PR 7 |
| F10 | unanchored GitHub remote regex | LIVE | PR 7 |
| F11 | issue association by current branch | PARTIAL | PR 9 (metadata, then the new branch scheme) |
| F12 | persist issue id on workspace | PARTIAL | PR 9 |
| F13 | vet vs launch use different config | LIVE, low (adjacent TOCTOU; Val partial) | PR 5 (vet again at launch on fresh config) |
| F14 | failed-only issue row bucket | FIXED | `client/radar.ts:1027-1028` |
| F15 | "Agent failed" reason on issue rows | LIVE | PR 7 |
| F16 | rapid settings edits drop final value | LIVE | PR 6 |
| F17 | directories offered as routing scripts | FIXED | `client/settings-screen.tsx:47-56` |
| F18 | save chain resends old revision | LIVE | PR 6 |
| F19 | disabled routing leaves applied values | PARTIAL | PR 3 (replace, not merge) |
| F20 | routing invocation token | FIXED | `client/start-agent-modal.tsx:145`, `:192` |
| F21 | compare routed provider with current | FIXED | `:142`, `:213-219` |
| F22 | routing closes over empty `profiles` | LIVE | PR 4 |
| F23 | profile-default provider before compare | PARTIAL (first open) | PR 4 |
| F24 | profile closure; cross-provider model | LIVE (Val §V1.2-2) | PR 3 |
| F25 | = F22 | LIVE (dup) | PR 4 |
| F26 | fallback profile on first open | PARTIAL | PR 4 |
| F27 | model-only result after provider edit | PARTIAL | PR 3 |
| F28–F33 | modal routing/scroll/busy fixes | FIXED | `client-races.md` §1 |
| F34 | template dispatch values | FIXED | standalone `examples/pr-radar-route.py` (#618) |
| F35 | copy-only install cannot run router | FIXED | same; **do not re-sync** (Val §V4.3) |
| F36 | bundled router imports ADE modules | OUTDATED | `examples/route_agent.py` removed in #618; no ADE action |
| F37 | errored accounts count as measured | ADE, FIXED | `~/src/djbclark-ade/bin/route_agent.py:1120-1125` |
| F38 | sublimits/windows merge | ADE, FIXED | `route_agent.py:1240-1241` |
| F39 | stale `age_s` retained | ADE, LIVE | ADE-fix (in flight) |
| F40 | unknown window folded | ADE, FIXED | `route_agent.py:986-990`, `:1097` |
| F41 | failed issue search wipes cache | LIVE | F41-PR (in flight) |
| F42 | issue cap truncation flag | FIXED | review sample (Rev §E1) |
| F43 | completeness includes issue limits | FIXED | review sample |
| F44, F45 | git env / gh credential separation | FIXED | `server/workspace-ops.ts:80-115`, `:199-210` |
| F46 | `XDG_CONFIG_HOME` dropped | DECLINED | `:202-203`, maintainer |
| F47 | repo-config command execution | FIXED | `:140-187` |
| F48 | fast-forward guard | FIXED | `:655-669` (deleted by PR 12) |
| F49 | shared `FETCH_HEAD` | FIXED | `:578-591` |
| F50 | issue CWD-BASE dirty check | OUTDATED | `:972-982` |
| F51 | leading-dash repo names | DECLINED | maintainer 2026-11-17: rejected on purpose |
| F52 | CWD-BASE then CWD-WORKTREE on same item | LIVE (Val §V1.2-1) | 618-fu |
| F53 | reused PR worktree freshness | FIXED | `:842-869`; removed by Move 1 → B.8 hint (PR 11) |
| F54 | exact `github.com` host | FIXED | `:390-397` |
| F55 | existing issue branch kept | FIXED | `:999-1002`; lost under Move 1 unless S8 passes (B.3) |
| F56 | fetched PR identity kept | FIXED | `:578-591` |
| F57 | `child.stdin` errors | FIXED | `:1321` |
| F58, F59, F64 | ignored-file and gitlink guards | FIXED | `:696`, `:862-863`, `:992-993`, `:1000-1006` |
| F60 | no `origin/HEAD` → any branch is a start | LIVE | PR 9 (Paseo `branch-off` from the default) |
| F61 | transactional worktree creation | PARTIAL: issue FIXED, PR path LIVE (Rev §E1) | 618-fu (half-created PR worktree cleanup) |
| F62 | worktree root confinement | FIXED | `:941-950` |
| F63 | branch names lack repository | LIVE | PR 9 (`pr-radar/<owner>/<repo>/issue-<n>`; PR rows by `headRef` + head check) |
| F65 | issue start point from stale HEAD | PARTIAL (residue = F60) | PR 9 |
| F66 | CWD-BASE shared path, no reservation | LIVE | PR 10 per Q1 |
| F67 | concurrent prepares race | LIVE | 618-fu (per-issue lock, partial); PR 9 (no base index writes; per-target clone lock) |
| F68, F69 | issue worktree cleanup / reuse | FIXED | `:876-891`, `:1117-1124` |
| F70, F71 | script dir writability / owner | FIXED | `:1184-1208` |
| F72 | guard released before killed group exits | DECLINED | M2's settle-on-`close` covers it incidentally; no thread reopen |
| F73 | guard released while script runs | FIXED | `:1350-1354` |
| F74 | descendants outlive success | LIVE | M2 (kill group on `exit`, settle on `close`) |
| F75 | auto-launch modes coverage (2 of 4) | TEST-GAP | PR 8 |
| F76 | routing example empty input | TEST-GAP | PR 8 |
| F77 | no registration-level RPC test | TEST-GAP | PR 8 (`listLocalDirs`, `runRoutingScript`), PR 9 adds `resolveBaseCheckout` |
| F78 | routing cleanup between tests | FIXED | review sample |
| F79 | object-store env in fixtures | FIXED | `tests/workspace-ops-handlers.test.ts:50-51`, `:77-78` |
| F80 | checkout sequencing untested | TEST-GAP | superseded by PR 12 deletion; nothing before |
| F81 | gh credential source boundary | TEST-GAP | PR 8 (in a surviving or new test file, not one PR 12 guts) |
| F82–F84 | path-query / hostname / empty-name tests | FIXED | review sample |
| F85 | unchanged-gitlink test asserts too little | TEST-GAP | 618-fu (unchanged-gitlink test) |
| F86 | ignored-file fixture | FIXED | review sample |
| F87 | submodule test is a false positive | TEST-GAP | 618-fu (gitlink tests; confirm the fixture commits the `submodule add` first) |
| F88 | failure-reason variant coverage | TEST-GAP | PR 8 |
| F89 | fixture git env isolation | FIXED | `tests/workspace-ops.test.ts:280-286` |
| F90 | ambient env in safety test | FIXED | review sample |

Not in the inventory, still open: R1 (modal double click) → PR 5; R3 (settings captured while loading) → PR 4; R4
(re-run keeps old routed values) → PR 3; R5 (page-wide error in modal) → PR 5; C5, C7 → M2.

### C.2 Tally and thread replies

- Open rows: 36 (counting F25 as a duplicate, F39 for ADE, partials and test gaps). Fixed, outdated or declined: 54.
- Do **not** resolve any thread for F05, F11, F12, F13, F27, F41, F52, F61, F77, F85 or F87 before its PR lands
  (Rev §E9 Q8 plus Val's additions).
- Threads for code that PR 12 deletes (F60, F63, F65, F66, F67, F80) are resolved when PR 12 merges, not before.

## D. Move 2: routing trust (brief ready; dispatched after #662 lands)

One PR (v1's PRs 1+2):
- Move `:1155-1361` to `server/routing-script.ts` and its tests to `tests/routing-script.test.ts`. Export what it
  needs (`allowListedEnv`, `exists`, `expandUserPath`, `isAbsoluteOrHomePath`, limits) from a small shared module.
  This is not a pure move: the import paths in the tests change.
- **C4 / F74.** In `child.on("exit")`, call `signalGroup("SIGKILL")` so a descendant holding the pipes cannot hold up
  `close`. Keep `settle()` and `finishedGate.resolve()` in `close`, so buffered stdout is read in full. Keep a
  backstop `signalGroup("SIGKILL")` in `close` (Val §V3). Tests: a descendant with redirected stdio is gone after a
  successful run; a descendant that inherits the pipes does not turn a 0 exit into a timeout. `setsid` escapes are
  documented, not fixed.
- **C5.** Walk the configured path hop by hop with `lstat` + `readlink`. At each hop check the link's owner and its
  parent directory's owner and mode (sticky rule included), then run the existing real-path ancestor walk. Tests: a
  symlink in a group-writable directory is refused; a two-hop chain through such a directory is refused.
- **C7.** A bounded queue: at most 2 waiters, a third is rejected. Chain on `run.finished`, not `run.result` (or F73
  returns). Queue time counts against the 30 s, because Paseo's plugin request timeout is also 30 s
  (`runtime.ts:45`). Test: two concurrent calls both return, in order; a third is rejected.
- **#618 LOW item 3.** Document that POSIX ACLs are not read; run the script with a fixed minimal `PATH`.
- Proof: the new tests fail on `9f1f053` and pass on the branch (red/green recorded in the PR body); typecheck;
  biome.

## E. Move 3: client logic, not a component split

The #662 rework already moves launch into `client/launch.ts` (`launchNewAgent`, `applyAutoRouting`,
`defaultLaunchOverrides`). Everything below extends those functions; none adds a parallel path.

- **PR 3, `resolveRoutedLaunch(current, routed, profiles, edited)` returning a delta, plus `buildRoutingPayload`.**
  The modal and `applyAutoRouting` both call them. Rules:
  1. Drop a model-only result when the provider was edited (F27).
  2. Read `current` from refs at apply time, not render (F24 closure).
  3. **Replace** previously routed fields, never merge (F19 residue, R4).
  4. Clear an unedited `model` whenever the effective provider changes, including through a routed profile with no
     `routed.provider` (Val addition 1; F06/F24).
  5. Resolve `defaultProvider: ""` to the profile's provider on both paths (F06 drift).
  The routing stdin payload gets a golden fixture here; PR 12 uses it.
- **PR 4, modal readiness.** Await the config query inside the handler. Re-check the row and sequence guard after the
  new `await`. A failed fetch falls back instead of discarding the result. Start stays disabled until settings are
  loaded (R3).
- **PR 5, one launch entry.** `useStartAgent` (B.2 lock) wraps `launchNewAgent`. The profile is vetted again at launch
  on the fresh `paseo.config.get()` list (Val addition 2; F13). No idempotency key. The modal gets its own error
  (R5). Delete the unreachable `start`/`checkout` branches of `agentMutation` (`client/pr-radar.tsx:195-230`).
- **PR 6, settings.**
  - `savePatch` uses the `saving`-flag pattern from `agent-monitor/client/agent-monitor.tsx:150-160`; never wait for
    `settings.revision`, which a no-op or failed save never changes.
  - `ComboSelect` gets a `commitMode` prop: `"keystroke"` for the modal, which keeps `editedFieldsRef` and the F21/F27
    guards working, and `"commit"` for Settings, with a flush on unmount. It re-syncs from `value` only when the
    field is not dirty, so routing results, row reset and Settings "Reset" still reach it.
  - Error text says what failed, not always "changed concurrently".
- No component split now. A mechanical split of `PrRadar` for readability is optional and comes last.

## F. Plan after the in-flight work

In flight (not repeated here): **#662-rw** → merge; **618-fu**; **F41-PR**; **ADE-fix**; **M2** (after #662).
Every PR follows the repo's `deliver` skill. Checks go through `bgrun` after `load-watch ci decide`: `bun run
typecheck`, `bun run check`, and `bun run test -- <named files>`. Runtime PRs end on an isolated daemon left up with a
seeded scenario. "Proof" means a check that fails before the change and passes after.

| # | Scope | Files | Proof | Closes |
|---|---|---|---|---|
| 1 | **Design issue** to omercnet (draft: `upstream-issue-draft.md`). Gate for 9–12 | — | his answers recorded in the PR 9 body | Q1, Q2, Q6, Q7, semver |
| 2 | **Spike** B.4 (not merged); file the getpaseo/paseo fork issue if S2/S3 confirm | — | S1–S9 table filled in | decides F-a/F-b, S4, S8 |
| 3 | `resolveRoutedLaunch` + `buildRoutingPayload`; golden routing payload | `client/launch.ts`, `client/radar.ts`, `client/start-agent-modal.tsx`, `tests/routed-launch.test.ts` | table tests: model-only after provider edit; profile-only cross-provider clears model; replace-not-merge on rerun/disable; `defaultProvider: ""` both paths. Each case red on the #662 head | F06, F19, F24, F27, R4 |
| 4 | Modal readiness | `client/start-agent-modal.tsx`, test | extracted "route when ready" test with profiles arriving late and a failed config fetch; Start disabled while settings load | F22/F25, F23, F26, R3 |
| 5 | `useStartAgent`: lock in ref + state, vet at launch, modal-local error, dead branches deleted | `client/use-start-agent.ts` (new), `client/pr-radar.tsx`, `client/launch.ts`, tests | hook test: two synchronous calls → one `launchNewAgent`; busy state visible during routing; profile edited between vet and launch → refused. Isolated daemon: double-click in modal and auto → one agent, one navigation | F05, F13, R1, R5 |
| 6 | Settings save queue + `ComboSelect` `commitMode` | `client/agent-launch-options.tsx`, `client/settings-screen.tsx`, test | fake `save` that resolves late, returns `false`, and is a no-op: final value lands, no deadlock, no false "concurrently". Modal keystroke mode keeps `editedFieldsRef` behaviour (test) | F01, F16, F18 |
| 7 | One-liners: shared anchored remote parser in `shared/`; issue-row "Agent failed"; action width | `shared/`, `client/radar.ts`, `client/pr-radar.tsx` | `tests/radar.test.ts`: `https://evil.example/github.com/o/r` and `git@notgithub.com:o/r` rejected; failed-only issue reason | F09, F10, F15 |
| 8 | Tests: registration-level RPC test through `contribute(server)`; F03, F75 (all 4 modes), F76, F81, F88 | `tests/` (new or surviving files) | the tests themselves, plus one mutation each: break the code under test and see it fail | F03, F75, F76, F77 (partly), F81, F88 |
| 9 | **Native worktree for PR and issue rows** (v1's 6+7): `resolveBaseCheckout` (+ per-target clone lock, half-clone detection); fork rule (F-a or F-b); fresh key; `onEvent`; hook-failure rendering; metadata when `entityMetadata`; reuse lookup (B.3); post-create head check; issue branch `pr-radar/<o>/<r>/issue-<n>`; association: metadata → new branch → legacy; picker lists matching Paseo workspaces; `worktreeName` → `worktreeSlug`. **Opt-in** behind a `checkoutBackend: "legacy" \| "paseo"` setting, default `legacy` | `server/workspace-ops.ts`, `shared/viewer-scope.ts`, `index.server.ts`, `client/use-start-agent.ts`, `client/launch.ts`, `client/radar.ts`, `client/start-agent-modal.tsx`, README, tests | handler tests for `resolveBaseCheckout` (confinement, remote match, concurrent clone of one target); pure tests of the create options, the key rule and association (old/new branch, metadata, suffix). Isolated daemons at 0.12.x and 0.16.2, with `paseo` selected: PR row in a plain clone → HEAD = `headRefOid`; fork+upstream row → F-a result or F-b refusal; fork-own-PR-N → head-check warning; issue row twice → second launch reuses; throwing test hook → modal stays open with "Blocked by …" | F60, F63, F65, F67 (worktree mode), F11, F12 |
| 10 | CWD-BASE per Q1 | `client/use-start-agent.ts`, modal text, README, server if the mode is removed | isolated daemon: base launch leaves branch and `git status` unchanged; a second base launch while a live agent holds the directory is refused | F66 |
| 11 | "N commits behind" row hint + head SHA in the launch prompt | `server/workspace-ops.ts` (read-only RPC), `client/radar.ts`, `client/pr-radar.tsx` | test: worktree at an old commit with a fetched private ref → hint shows the count; isolated daemon: push to the PR, the row shows "1 behind" | mitigates B.8 |
| 12 | **Flip and delete** (`feat!`): default `paseo`, remove the setting, `prepareWorkspaceCwd`, the guards, and their tests | `server/workspace-ops.ts` (≈ −600), `shared/viewer-scope.ts`, tests (≈ −1,500), README, settings text | (a) `rg` over `pr-radar/` for every removed symbol and RPC name → zero hits outside CHANGELOG. (b) `tsc --noUnusedLocals --noUnusedParameters` one-off, or biome `noUnusedVariables`, → no dead private helper (e.g. `resolveGitCommonDir`). (c) **Routing stdin contract**: the PR 3 golden payload is unchanged except for documented fields, and `~/src/djbclark-ade/bin/pr-radar-route` plus `examples/pr-radar-route.py` both parse it (test runs the example; the ADE adapter check runs in ADE CI). (d) README and settings copy grep for "sibling", "CWD-BASE switches", `pr-radar/pr-`. (e) Old client: the server keeps a `prepareWorkspaceCwd` stub for one release that rejects with "update PR Radar"; test it. (f) Isolated daemon: the PR 9 scenario list again, plus an existing legacy `pr-radar/pr-N` worktree still showing on its row | F80 (superseded); closes the deletion-bound threads (C.2) |
| 13 | Optional: mechanical split of `client/pr-radar.tsx` | `client/` | typecheck; no test change | — |

Order and parallelism:
- 1 and 2 start now, alongside the in-flight work.
- 3 → 4 → 5 → 6 in that order. They all touch the modal or `ComboSelect`, and 5 depends on #662-rw's `launch.ts`.
- 7 and 8 are independent and can run beside the chain.
- 9 starts only after 1 is answered, 2 passes, 5 has merged and 618-fu has merged.
- 9 → 10 → 11 → 12. 12 waits one release after 9 ships opt-in.
- 618-fu repairs code that 12 deletes. It goes ahead anyway: the maintainer asked for it (merge comment, 2026-11-18),
  and Move 1 is not agreed.

## G. Operator decisions

| # | Question | Ruling or recommendation |
|---|---|---|
| Q1 | CWD-BASE switches the shared checkout's branch | **Open.** Recommend: keep the mode but stop switching. Open the checkout as is, and **refuse when a live agent already holds that directory** (closes F66; Rev §E9). Removing the mode is the cleaner alternative, but the maintainer asked for its warning and detached-HEAD guard in #618 (2026-11-17), so ask him in the issue |
| Q2 | Paseo places worktrees; `worktreeName` | **Open.** Accept placement; keep `worktreeName` and send it as `worktreeSlug`; the picker lists matching Paseo workspaces. Users who want siblings install a `worktree.create.before` placement plugin (Paseo ships an example) |
| Q3 | Keep clone inside pr-radar | **Open.** Keep it, with a per-target lock and half-clone detection; resolve S9's timeout question first |
| Q4 | Claim rebind when the owner agent is archived | **Open, blocker for the operator's setup** (Rev §E3). Recommend that the claim plugin rebind a claim whose owner agent is archived to the next agent opened in that workspace, and log it. Until it does, pr-radar does not reuse a workspace whose agents are all archived (B.3) |
| Q5 | Raise the floor to `^0.15` | **Ruled: no** (brief; Rev §E2, Val §V3). Add `^0.17.0` only after testing |
| Q6 | No daemon refresh of reused PR worktrees | **Open.** Accept only with the regression stated to the maintainer and PR 11's hint |
| Q7 | Drop pr-radar's git hardening for mutating commands | **Open, the maintainer's call.** Recommend accept. Under F-a the private-ref fetch and the clone stay, and so does the static `GIT_SAFETY_ARGS` (and the dynamic filter neutralizing if any checkout remains) |
| Q8 | The 90 findings | **Ruled: redo with corrections** (Rev §E9) → section C. Reply per C.2 |
| Q9 | Re-sync `examples/pr-radar-route.py` | **Ruled: no.** Fix the ADE docstring instead (brief; Val §V4.3; ADE-fix in flight) |
| R1 | #662 first | **Ruled:** rework in flight; merge before Move 2 and Move 3 |
| R2 | Design issue before Move 1 | **Ruled** (this brief). The draft is ready. Send it before PR 9, not before PR 5 |
| R3 | Semver | **Open.** Recommend: PR 9 ships the native path **opt-in** (minor). PR 12 flips the default and deletes in one `feat!` (major) |
| R4 | Existing `pr-radar/pr-N` / `pr-radar/issue-N` worktrees | **Open.** Recommend: never touched or deleted. Association keeps the legacy patterns for one major. A legacy worktree is "open existing" only, never refreshed. README says how to remove them (`git worktree remove`) |
| R5 | Hosts without `entityMetadata` | **Open.** Recommend: create always (no legacy path), omit `metadata`, associate by branch (B.3) |
| R6 | Paseo 0.17 | **Open.** Run the spike on 0.17.0-beta.1 too if cheap. The four checkout/creation files read here behave the same at `v0.17.0-beta.1` |
| R7 | Idempotency key | **Ruled:** fresh per click, transport recovery only (brief; Val §V2c) |
| R8 | UI lock | **Ruled:** kept; one instance, ref + state (brief; Rev §E5) |
| R9 | Fork topology | **Open.** Recommend F-a if S8 passes, else F-b. File the getpaseo/paseo issue either way |
| R10 | Post-create head check | **Open.** Recommend yes. It is cheap, read-only, and the only guard against the fork's own PR N and against a third plugin rewriting the request |
| R11 | F41 own PR; label namespace | **Ruled; in flight** (F41-PR, #662-rw) |

## H. What could not be verified

- Which repository `gh repo view` resolves in a fork+upstream checkout without `gh repo set-default` (S2).
- Whether a 0.12–0.15 daemon rejects or ignores `metadata` on create (S4).
- Whether `action: "checkout"` with an arbitrary `refName` (an existing local branch, or `refs/pr-radar/pull/N`) and an
  explicit `branchName` works without a suffix (S8). F-a and issue-branch reuse depend on it.
- Whether the plugin RPC 30 s timeout applies to client-to-plugin RPCs, including a 120 s clone (S9).
- Whether existing `pr-radar/pr-N` worktrees link to their PR rows today (Rev §E9-4).
- `v0.16.2` / `v0.17.0-beta.1`: only the four files named in the header were read. Hooks, `worktree.ts` and the
  protocol schema were not.
- The #662 rework's `client/launch.ts` is uncommitted; its API may change before merge.
- No tests, builds or daemons were run.
