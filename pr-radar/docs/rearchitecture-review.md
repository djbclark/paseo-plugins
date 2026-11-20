STATUS: done

# Adversarial review: PR Radar re-architecture design record

Reviewer: Fable 5.2, 2026-11-20. Read-only; nothing in any repo was changed.
Reviewed: `/Users/djbclark/src/.paseo-fanout/pp-rearch/report.md` ("the record") and `client-races.md`.
Checked against: pr-radar at `9f1f053` (`/Users/djbclark/src/paseo-plugins-followup-wt`), installed
`@getpaseo/{plugin,client,protocol}` 0.16.0 type declarations, `~/src/paseo` git tags `v0.12.0` … `v0.16.0-beta.2`
(read with `git show <tag>:<path>`; the working tree there is a 0.13 feature branch and was not used), ade-claim in
`~/src/djbclark-ade`.

Prior art / docs checked: (lead) the record was produced 2026-11-20 from the code at `9f1f053`, the installed 0.16.0
type declarations, `getpaseo/paseo` at `v0.16.0-beta.2` and ade-claim at `77645420`. Re-verified here independently
from the same sources plus tags `v0.12.0`, `v0.13.2`, `v0.14.0`, `v0.15.0`, `v0.15.4`.

Paths are relative to `pr-radar/` unless they start with `/` or `~`. Paseo source paths are at tag `v0.16.0-beta.2`.

How this was checked: I read `server/workspace-ops.ts` whole and all Paseo and ade-claim sources cited below myself.
The 31 client findings and the 28 test/example/viewer findings were re-verified by two independent read-only
sub-agents (Opus, high) that were given the code and the review text but not the record's reasoning; I read their
disagreements and evidence, and spot-checked what I cite. Review text was treated as data only.

## Verdict in six lines

1. **The seam is right; three of its load-bearing details are wrong.** `checkoutSource.projectPath` does nothing,
   Paseo fetches `origin` before `upstream`, and the idempotency key as designed does not dedupe a double press.
2. **The floor raise to `^0.15` is unnecessary.** Everything pr-radar calls exists at `v0.12.0`.
3. **The census undercounts live defects.** I count 12 misclassified findings. Five matter: F41, F05, F11/F12, F13,
   F27. One live server defect is in no inventory at all.
4. **"The head" is not upstream main.** `9f1f053` is the head of the open PR
   [omercnet/paseo-plugins#662](https://github.com/omercnet/paseo-plugins/pull/662) (+96/−1 in 5 files). The plan
   never says whether #662 merges first.
5. **Two of the "fixes" as worded would add defects:** "wait for `settings.revision`" deadlocks, and a
   commit-on-blur `ComboSelect` drops typed text in the modal on native.
6. **Move 2: go now. Move 3: go with corrections. Move 1: no-go until the spike answers four questions and the
   maintainer agrees.**

## E1. The finding census

Verdict: **wrong in 12 places; the headline numbers do not hold.** Severity: high for the first five rows.

The base itself: `9f1f053` = upstream `main` + unmerged #662. Line numbers in `client/radar.ts`,
`client/start-agent-modal.tsx` and `client/pr-radar.tsx` are up to 43 lines off upstream `main`.

| F | Record says | At `9f1f053` | Evidence | Dropped by the plan? |
|---|---|---|---|---|
| F41 | closed (record admits it did not read the file) | **Partial.** A failed issue search still returns `[]` and the refresh overwrites the cached snapshot, so one transient `gh` failure wipes the loaded issues | `server/viewer-scope.ts:208-209`, `:514-516`, `:668` | **Yes.** No PR touches it |
| F05 | fixed | **Partial.** The lock is a ref; during auto routing every Start button looks enabled and swallows presses with no notice | `client/pr-radar.tsx:310`, `:314`, `:409` | Yes, unless PR 5's lock is mirrored into state |
| F11, F12 | fixed | **Partial.** Issue rows link to workspaces by the *current* branch `pr-radar/issue-N`. A later CWD-BASE launch in the same checkout moves the branch: the first issue loses its agent, the second inherits every agent of the base workspace | `client/radar.ts:385-415`, `server/workspace-ops.ts:960-1014` | Partly (PR 7 only if issue rows read metadata first) |
| F13 | not a defect | **Live, low.** The modal vets the routed profile against a 30 s cached list; the launch re-reads config and never vets again. A routed name can resolve to an unvetted profile. It defeats a security check | `client/agent-launch-options.tsx:326-334`, `client/pr-radar.tsx:268-276` | Yes ("F13 adjacent" is named but no PR vets at launch) |
| F27 | fixed | **Partial.** A model-only routed result that arrives after the user changed the provider is applied to the new provider | `client/radar.ts:793`, `client/start-agent-modal.tsx:228` | Yes, unless `resolveRoutedLaunch` drops it |
| (none) | not in the inventory | **Live.** The PR path creates a worktree with `--no-checkout` and has no cleanup when `checkout --force` fails; the issue path has it. The leftover has an empty index, so the next launch's dirty check should refuse it for good. F61/F68/F69 are "fixed" for issue rows only | `server/workspace-ops.ts:1133-1143` vs `:1117-1124`; reuse at `:1028-1041`, `:849-859`. By reading; not executed | Superseded only if PR 9 lands |
| F85, F87 | closed (implied) | **Live test gaps.** F85's test asserts only the folder and branch; F87's rejection is caused by the parent's own staged `submodule add` | `tests/workspace-ops-handlers.test.ts:783-785`, `:914-931` | Superseded only if PR 9 lands |
| F77 | superseded by deletion | **Not fully.** No test goes through `server.handle` registration for `listLocalDirs` or `runRoutingScript`, and both stay | `index.server.ts:34-36`, `tests/workspace-ops-handlers.test.ts:17-26` | Yes |
| F79, F89 | live, superseded | Fixed at head. Harmless overcount | sub-agent table | No |
| F36–F40 | F36, F38 go to djbclark-ade | `examples/route_agent.py` is not in the tree at `9f1f053` (`examples/` holds only `pr-radar-route.py`). F39 is also still live upstream, so three go to ade, not two | `~/src/djbclark-ade/bin/route_agent.py:1242` (sub-agent; not re-read by me) | F39 dropped |

Corrected tally: live client defects are 7 plus 5 partials (F05, F11, F12, F13, F27), not 7; live in
`workspace-ops.ts` is 6, not 5; one live viewer defect (F41) is missing. "About 60 closed" is nearer 52.

Sample that agreed with the record (server, by my own read): F44/F45 (`:80-115`), F46 declined (`:202-203`),
F47 (`:140-187`), F48 (`:861`, `:986`, `:1127`), F49/F56 (`:578-591`), F50 outdated (`:972-982`), F51 declined
(`:374`), F52 (`:1028-1030`), F53 (`:842-869`), F54 (`:390-397`), F55 (`:999-1002`), F57 (`:1321`), F58/F59/F64
(`:862-863`, `:992-993`, `:1000-1006`), F60 live (`:822-839`), F62 (`:941-950`), F63 live (`:960-963`), F65 partial
(`:802-815`), F66 live (`:1015-1022`), F67 live, F70/F71 (`:1184-1208`), F72 declined (`:1272-1275`), F73
(`:1350-1354`), F74 live (`:1309`). Client and test rows that agreed: F01, F04, F06–F10, F14–F26 (except F13),
F28–F33, F02, F03, F34, F35, F42, F43, F75, F76, F78, F80–F84, F86, F88, F90.

## E2. API claims in B.2 / B.7

| Claim | Verdict | Evidence |
|---|---|---|
| (a) `paseo.workspaces.create` reachable from client and server halves | **True, and for longer than the record says** | Client: `@getpaseo/client/dist/index.d.ts:165`, options type `:92-96`. Server: `PluginHandlerContext.paseo: PaseoApi` (`@getpaseo/plugin/dist/server/contracts.d.ts:8-10`), so an RPC handler can call it too. `create(options: PaseoWorkspaceCreateOptions)` is present in `packages/client/src/index.ts` at every tag from `v0.12.0` |
| (a) `idempotencyKey`, `source.kind:"worktree"`, `checkoutSource`, co-created `agent` | True | `packages/protocol/src/messages.ts:2921-2965`, `:1323-1328`. All four fields are in `WorkspaceCreateRequestSchema` at `v0.12.0` |
| (a) `checkoutSource.projectPath` pins the repository | **False** | `resolveInputChangeRequest` returns `projectPath` (`packages/server/src/server/resolve-worktree-creation-intent.ts:135-150`) and the caller drops it: `resolvePrCheckoutIntent({ refName, changeRequestNumber, repoRoot, deps })` (`:82-89`, `:104-111`). No other server code reads it. It is a no-op. The record calls it "the second pin" (B.6) |
| (b) idempotent per key, double press included | True **only for a byte-identical request** | Daemon: `packages/server/src/server/creation/index.ts:64-76` (persisted record, request fingerprint, running creation returned). Client: `packages/client/src/creation/index.ts:110-131` returns the same promise for same key + same fingerprint. Same key + any changed field rejects with `workspace_request_key_conflict` on both sides. The fingerprint is the whole intent: source, title, metadata, agent config, labels, initial prompt |
| (c) `worktree.create.after` throw aborts and removes the worktree | True per docs | `public-docs/plugins/hooks.md` "worktree.create.after"; call site `packages/server/src/server/paseo-worktree-service.ts:85`. I read the doc and the call site, not the rollback body |
| (d) 30 s hook budget | True | `packages/server/src/server/plugins/runtime.ts:45` `REQUEST_TIMEOUT_MS = 30_000`; hooks.md "Hook timeout" row |
| (e) workspace `metadata` at 0.16, queryable | Half true | In the create schema only at `v0.16.0-beta.x` (absent at `v0.15.4`). Readable on `PaseoWorkspaceHandle.metadata` and on list entries. There is **no server-side filter**: `PaseoWorkspaceListOptions` is the plain fetch request (`@getpaseo/client/dist/index.d.ts:78-80`), results are paged (`pageInfo`), and `setMetadata` is documented "Needs a host with `entityMetadata`" (`:126`). "Find the prior workspace for an item" is a client-side scan of every page |

### Version floor: the record's recommendation is wrong

- The record says it did not verify `workspaces.create` on 0.12–0.14 and recommends raising the floor to `^0.15`.
  I verified it: the method, `checkoutSource`, `idempotencyKey`, `worktreeSlug` and the co-created `agent` are all
  present at `v0.12.0`, `v0.13.2` and `v0.14.0`.
- The worktree hooks (`worktree.create.before/after`, `workspace.create.before`) first appear at `v0.15.0`. **pr-radar
  does not use them.** Only ade-claim does. The floor that matters for hooks is ade-claim's own manifest, not
  pr-radar's.
- So Move 1 needs **no floor raise** for pr-radar. Raising it would drop 0.12–0.14 users of a public plugin for the
  benefit of a private one. The only thing that would force `^0.15` is the B.7 fallback (a pr-radar
  `worktree.create.before` hook to steer the fetch remote). Decide that after the spike, not before.
- Not verified: that the 0.12–0.14 **daemon** performs the change-request checkout the same way as 0.16 (I checked
  the schema and the client method at those tags, not `worktree.ts`). The spike must run against the lowest
  supported daemon, not only 0.16.

### Facts the record missed that change Move 1

1. **Paseo fetches `origin` before `upstream`.** GitHub checkout refs are
   `[origin refs/pull/N/head, upstream refs/pull/N/head]`, first success wins
   (`packages/server/src/services/github-service.ts:3522-3525`, `packages/server/src/utils/worktree.ts:1686-1711`).
   pr-radar prefers `upstream` (`server/workspace-ops.ts:417-421`). In a fork checkout whose fork has its own PR
   #N, Paseo checks out the fork's PR, silently. The record flags fork risk but relies on `projectPath`, which does
   nothing.
2. **The local branch is the PR's head ref name, not `pr-radar/pr-N`.**
   `localBranchCandidate = source.localBranchName ?? source.headRef`, made unique
   (`packages/server/src/utils/worktree.ts:1577-1580`). Any pr-radar code that recognizes its worktrees by branch
   name stops matching.
3. **`workspace.create` "never deduplicates by directory: it always produces a fresh workspace"**
   (`packages/protocol/src/messages.ts:2918-2920`). Reuse is entirely pr-radar's lookup. If the lookup misses (no
   metadata on a pre-0.16 host, branch-name association broken by fact 2), every launch makes a new worktree.
4. **A third plugin can rewrite the request.** `workspace.create.before` receives the whole create request and may
   replace it (hooks.md "workspace.create.before"). Not a defect, but the record's seam diagram omits it.
5. **`worktreeSlug` exists on the request.** The record lists it in B.2 and then says the `worktreeName` field must
   go (B.5, Q2). It can be kept and passed as `worktreeSlug`.
6. **Branch-off never reuses an existing branch.** `resolveUniqueLocalBranchName` suffixes on collision
   (`packages/server/src/utils/worktree.ts:1554`, `:1932-1957`). Today an existing `pr-radar/issue-N` branch is
   reused with its commits (`server/workspace-ops.ts:1095-1103`, the F55 fix). After Move 1 a second launch for the
   same issue, once the first workspace is archived, starts from the default branch on `…-issue-N-1`. The record
   presents the suffix as a feature and does not list the lost reuse.

## E3. The seam's coupling risk

Verdict: **independence holds; the failure UX does not.** Severity: medium.

- No dependency: confirmed. ade-claim only registers hooks (`~/src/djbclark-ade/plugins/ade-claim/server/hook.mjs:222-267`);
  pr-radar would call only Paseo. In observe mode the hook never throws (`:250-253`, `:262-264`).
- In enforce mode **every** failure inside the hook throws, not only "path held by another owner": a missing
  `ade_claims.py`, a failed `git rev-parse`, a slow registry past 30 s. The user then sees, inside PR Radar's modal,
  `Plugin ade-claim worktree.create.after failed: ade_claims.py new failed (exit 4): …`.
- "Show Paseo's error text" is not enough. The text names tools a public user has never heard of, inside a
  product that did not cause the failure, with no next step. What pr-radar should do instead:
  1. Match the documented prefix `Plugin <id> <hook> failed:` and render it as
     "Blocked by the `<id>` plugin. PR Radar created nothing." with the raw detail below.
  2. Keep the modal open with every field intact. Today any failure lands in the page-wide `openError`, which is
     also reused as the modal's error (`client/pr-radar.tsx:1299`, residual R5).
  3. Pass `onEvent` to `workspaces.create` and show the phase. Creation can now take a 120 s fetch plus a 30 s
     hook; today's only progress signal is `isPending`.
  4. Never fall back to an unclaimed directory on its own. Offering "open the base checkout instead" as an
     explicit choice is acceptable.
- Risk the record files under "ade-claim's question" (Q4) that is really a blocker for the operator's own setup:
  with enforcement on, reuse puts a new agent into a workspace whose claim belongs to the previous agent, so it
  gets no `ADE_CLAIM_ID` (`hook.mjs:361-371`) and git-guard stops its commits. pr-radar's reuse path then produces
  agents that cannot write. Settle the rebind rule before PR 6 turns reuse on, or do not reuse a workspace whose
  only agents are archived.
- ade-claim's manifest says `>=0.14.0`; the hooks exist only from `v0.15.0`. The record is right about that.

## E4. The deletion table (B.6)

Verdict: **line ranges are exact; four gaps.** Severity: low to medium.

- Every range in the table matches the file at `9f1f053`. `ghEnv` is used by `server/viewer-scope.ts:14,77`.
  `slugifyBranch`, `defaultWorktreeName` and `isValidWorktreeName` have no caller outside this file and its tests.
- **Dead but unlisted:** `resolveGitCommonDir` (`:438-444`) is used only at `:1065-1066` and dies with the handler.
  `listWorktrees` (`:446-471`) stays (used at `:518`).
- **"Keep" row that is incomplete:** the clone keeps `checkout --force` (`:305`), a mutating checkout. The stated
  reason for deleting `configuredFilterSafetyArgs` ("only `remote`, `rev-parse`, `worktree list` and
  `clone --no-checkout` remain") omits it. It is safe (a fresh clone has no local filter config and global config
  is off, `:202-203`), but the PR must say so because the maintainer reviewed that guard.
- **"Keep unchanged" that will not stay correct:** `listLocalDirsHandler` shows only worktrees inside
  `sourceCodeDir` (`:521`). Paseo places worktrees elsewhere, so every worktree Move 1 creates disappears from the
  modal's directory picker. Either list by workspace record or accept that the picker shows base checkouts only.
- **PR 1 is not a pure move.** The routing code uses `allowListedEnv` (`:108`, not exported), `exists` (`:124`, not
  exported), `expandUserPath`, `isAbsoluteOrHomePath`, `COMMAND_TIMEOUT_MS` and `RoutingScriptLimits` (`:27-37`).
  It needs exports or a third module, and the tests' import paths change. Small, but "moved tests pass unchanged"
  is not literally true.
- Arithmetic: the listed ranges sum to about 660 lines; 207 move; about 60 are new. The file ends near 540.
  "620 / 210 / 530" holds to within 40 lines if "620" is net of the new code.
- Not checked: `shared/viewer-scope.ts:100-120`, and how much of the 1,664-line handler test file is routing
  tests that must move rather than die.

## E5. Live defects and their dispositions

| Defect | Proposed fix | Does it close? |
|---|---|---|
| F60 | Paseo `branch-off` from the project default | **Yes, with different semantics.** Paseo resolves the default branch or fails (`resolve-worktree-creation-intent.ts:70-76`, `:264-273`). Whether that is the local or the remote default, and whether it is fetched first, I did not verify |
| F63 | `pr-radar/<owner>-<repo>-issue-<n>`; metadata carries the repository | **Issue rows: yes**, with a nit: `a-b/c` and `a/b-c` both give `a-b-c`; use `pr-radar/<owner>/<repo>/issue-<n>`. **PR rows: no.** The local branch is the PR head ref name. With `origin`=fork and `upstream`=parent, Paseo fetches `origin refs/pull/N/head` first. The reuse fallback (`row.workspaceIds`) is keyed by the host's head-ref lookup for PRs and by current branch for issues, not by repository |
| F66 | Stop switching the base checkout (Q1) | **No. It moves the collision into the agents.** Two CWD-BASE launches still share one directory; each agent is told a PR number and checks it out itself, now without the dirty, ignored-file and gitlink guards. Closing F66 needs a refusal when a live agent already holds that directory, or removal of the mode |
| F67 | Delegation + idempotency key | **Partly.** Nothing touches the base index any more. Two launches for one issue with different keys now give either "branch exists" or two worktrees (`…-1`). And `resolveBaseCheckout` keeps the clone with no lock: two rows of one uncloned repository race on the same target directory, and the second can see a half-cloned directory as an existing checkout (`:920-958`) |
| F74 | Drop `if (terminating)` in the `close` handler | **Half.** It kills descendants that detached their stdio. A descendant that still holds the pipes keeps `close` from firing, so a script that exits 0 hangs for 30 s and is then reported as a **timeout**, and the launch falls back to defaults. Kill the group on `exit`, not `close`; `close` then follows with the full output. A descendant that calls `setsid` escapes either way |
| F01, F16, F18 | Draft-owning `ComboSelect`, commit-save, wait for `settings.revision` | **Not as worded.** See E7 |
| F22 = F25 | Await the config query inside the handler | Yes, if the row/sequence guard is re-checked after the new `await` and a failed fetch falls back instead of discarding the routing result |
| F09, F10, F15 | One-liners | Yes |

Does `idempotencyKey` close the double-launch class? **Not with a random nonce.**

- The key dedupes only an identical request under the same key. A nonce minted per click differs between the two
  presses; one minted per render changes on every clock tick (`client/pr-radar.tsx:168-171`).
- The auto path has no retry UI, so each press is a new intent. The key adds nothing there; the synchronous lock
  does all the work. The record's "closes F05/F08 in the daemon instead of with UI guards" is wrong.
- The modal path breaks on a normal retry: a failed launch, one edited field, Start again → same key, different
  fingerprint → `workspace_request_key_conflict` shown raw.
- After a daemon restart mid-create the key returns `outcomeUnknown` for good (`creation/index.ts:76-86`).
- Two devices pressing Start on one row get two nonces and two worktrees.
- Daemon dedupe covers `workspaces.create` only. `resolveBaseCheckout` and the three `setLabel` calls run before
  or after it, once per caller.

What the key is good for: a dropped socket during a long create. Keep it for that. Build it as
`pr-radar:<repository>#<kind>:<number>:<hash of the request body>:<attempt>`, bump `attempt` on
`outcomeUnknown`, and keep one shared synchronous lock (mirrored into state, which also fixes F05).

Dropping the daemon-side refresh: the regression the record glosses over is that **"Start New Agent" on a PR with
new commits now starts on stale code unless the agent decides to fetch.** F53 was a merged, reviewed fix for
exactly that. The agent's prompt is the only remaining control, and a reused worktree may be dirty with another
agent's work. Say this in the design issue; do not bury it under Q6.

## E6. Routing trust (C)

- **C4:** real; the fix is half right (E5, F74 row). The PR 2 test must cover both a descendant with redirected
  stdio and one that inherits the pipes. Severity low.
- **C5: real, narrower than stated, and the proposed fix is incomplete.** Verified: only ancestors of
  `realpath(expanded)` are checked (`:1179`, `:1195-1208`). The attack is concrete: if the configured path is a
  symlink in a directory another user can write, that user repoints it at `/bin/sh` (root-owned, clean ancestors,
  executable). The script is run with no arguments and the JSON payload on stdin (`:1234-1238`, `:1322`); `sh`
  reads stdin as a script, and `JSON.stringify` does not escape `$(…)` or backticks, so a PR title supplies the
  command. Precondition: the operator pointed the setting through such a symlink, on a multi-user host. Severity
  low to medium. The "12 lines over the lexical ancestors of `expanded`" misses (a) multi-hop chains
  (`/a/l1 → /b/l2 → /c/file`: `/b` is on neither chain) and (b) the symlink's own owner in a sticky directory.
  Walk each hop with `lstat` + `readlink`, checking owner and parent at every hop.
- **C7: the direction is right, the snippet is not.** An unbounded promise chain lets one slow script (30 s + 2 s
  grace) delay every queued launch, and Paseo's plugin request timeout is also 30 s
  (`packages/server/src/server/plugins/runtime.ts:45`, `:778`), so a queued call can fail at the client while the
  plugin still runs it later and holds the queue. Bound it: at most 2–3 waiters, reject beyond that, and count
  queue time against the 30 s. The chain must wait on `run.finished`, not `run.result`, or F73 comes back. I did
  not trace that the 30 s applies to client RPCs specifically.
- **"Not worth fixing" calls:** check-then-exec by path: agreed after C5. ACLs: agreed.
- **Dropping "restrict to the synced preset": justified, on the second reason only.** The first reason (a plugin
  cannot find its directory) is weaker than stated: ade-claim does find it through `~/.paseo/config.json`
  (`hook.mjs:30-44`). The second (the operator points the setting at the live ADE script) is sufficient alone.

## E7. The client recommendation (D)

- **"Do not split components": sound.** Both verifiers agree no live defect comes from co-location.
- **Five extracted changes are not sufficient.** Uncovered: F05 busy state, F11/F12, F13, F27, F41, R2, R4, R5.
  `resolveRoutedLaunch` closes F27, F24 and the F19 residue only if it (a) drops a model-only result when the
  provider was edited, (b) reads `current` from refs at apply time, and (c) **replaces** the previous routed
  fields instead of merging. None of the three is in the record.
- **"`savePatch` sends the next save only after `settings.revision` has changed" deadlocks.** The revision is a
  hash of the stored values, so a successful no-op save never changes it (Enter after per-keystroke saves, a blur
  with no change, a trailing space, Reset on defaults); a `false` result never changes it either. Use the
  `saving`-flag pattern already in this repo (`agent-monitor/client/agent-monitor.tsx:150-160`).
- **A commit-on-blur `ComboSelect` breaks the modal.** `ComboSelect` is shared. The modal depends on per-keystroke
  `onSelect` for its state and for `editedFieldsRef`, which is what stops a routing result overwriting a field in
  mid-edit (the F27 and F21 fixes). On native, tapping Start does not blur first, so a half-typed value is not
  sent. And "stop resetting the draft from `value`" breaks routing results, the per-row reset and Settings
  "Reset", which reach the text only through that effect. Make the commit mode a prop: keystroke for the modal,
  commit for Settings, with a flush on unmount; re-sync from `value` only when the field is not dirty.
- **Does daemon dedupe hide a bug from the user?** Yes, in one case: if both callers run the launch function, both
  call `navigation.openAgent` (it is inside `mutationFn`, `client/pr-radar.tsx:287-289`), which can push the agent
  screen twice. One shared lock in a single `useStartAgent` instance owned by `PrRadar` prevents it.
- R3 is not fixed by (c). Simplest: keep Start disabled until settings are loaded.

## E8. The plan (E)

- **PRs 1–2 first: correct, with one precondition.** #662 is open on the same branch. Decide merge-or-close before
  anything else; the proposal says "frozen", the record builds on its code. It is 96 lines: merge it.
- **PR 9's proof is not a proof.** Typecheck shows no TypeScript caller of the removed exports. It does not show:
  dead private helpers (`resolveGitCommonDir`, unless `noUnusedLocals` or biome flags them); the **routing-script
  stdin contract** (`workspace` fields the ADE adapter reads; removing `worktreeName` or changing `cwdMode` values
  breaks `~/src/djbclark-ade/bin/pr-radar-route` silently); README and settings text; an old cached client bundle
  calling a removed RPC.
- **PR 5 before PR 6 has a hidden seam.** In PR 5 the launch is still `workspaces.open` + `agents.create`, so the
  key can only go on `agents.create`, and at 0.16 a keyed agent create with an initial prompt may be rejected
  (`session.ts` near `:4752`, reported by the verifier; not re-read by me). Ship the lock in PR 5 and the key in
  PR 6.
- **PR 10 is not independent of 3–4.** It changes `ComboSelect`, which the modal's guards rely on (E7).
- **PR 12 partly feeds PR 9.** F81 and F88 add tests to the two files PR 9 guts. Do them after 9, or in the new
  `routing-script` and `radar` test files.
- **PR 8 does not close F66** (E5). Either add the "directory already has a live agent" refusal or retitle.
- **Missing PRs:** F41 (viewer cache wipe); vet-at-launch for F13; the design issue for the maintainer as a
  numbered step *before* PR 5, not a note under "risks"; a registration-level test for the surviving RPCs (F77).
- **Reorder:** (0a) merge or close #662 → (0b) design issue → 0 spike, against the lowest supported daemon and a
  fork+upstream checkout → 1+2 → F41 fix + 11 → 3+4 → 5 (lock, state-mirrored) → 10 (prop-gated) → 6 → 7 → 8 → 9
  → 12.
- **13 PRs is about right for review size, wrong in two joins.** Collapse 1 into 2 (a move plus its three fixes
  is one reviewable unit) and 6 with 7 (same function, same lookup). Split nothing else. 13 stays out.

## E9. Decisions for the operator (F)

| Q | Record | Ruling |
|---|---|---|
| Q1 | Stop switching; open the checkout as is | **Remove the mode instead**, or add a refusal when a live agent holds the directory. "Open as is" leaves F66 open and hands unguarded branch switching to the agent |
| Q2 | Accept Paseo placement; drop `worktreeName` | Accept placement. **Keep the field** and pass it as `worktreeSlug`. Also decide what the directory picker shows (E4) |
| Q3 | Keep clone | Keep, **with a per-target lock** and a check that a directory mid-clone is not treated as a checkout |
| Q4 | ade-claim's to decide | **Blocker for reuse in the operator's setup**, not a footnote (E3) |
| Q5 | Raise the floor to `^0.15` | **No.** Not needed (E2). Separately add `^0.17.0` only after testing; `^0.16.0` already covers 0.16.2 |
| Q6 | Accept no daemon refresh | Accept only with the stale-head regression stated to the maintainer and a visible "worktree is N commits behind the PR" hint on the row |
| Q7 | Accept dropping the hardening | Accept, but it is the maintainer's call and the largest security-posture change in the plan: PR content is then checked out under the user's global git config and filters |
| Q8 | ~60 resolve, 8 superseded, 12 cherry-pick, 5 tests, 2 to ade | **Redo with the E1 corrections** before replying on any thread. Do not resolve F41, F05, F11, F12, F13, F27, F85, F87, F77. Three go to ade (add F39) |
| Q9 | Re-sync the adapter as a chore | Agree |

Decisions the record did not surface:

1. **#662: merge first or close.** Everything is numbered against it.
2. **The maintainer.** PRs 6–9 remove four behaviours merged a few days ago. One design issue, before PR 5.
3. **Semver.** Removing CWD-BASE switching, sibling placement, branch names `pr-radar/pr-N` and daemon refresh
   is breaking for a plugin under release-please. Pick `feat!` and a major, or a setting that keeps the old path
   for one release.
4. **Existing `pr-radar/pr-N` worktrees.** The record says they "keep showing on their rows". Unproven: PR rows
   link through the host's head-ref lookup, and those branches have no upstream config, so they may not link
   today either. After Move 1 a launch on such a PR makes a second worktree beside the old one.
5. **Existing `pr-radar/issue-N` branches.** They are no longer reused (E2, fact 6). Rows linked by the old
   pattern must keep matching; PR 7 says so, but the launch will still branch afresh.
6. **Hosts without `entityMetadata`** (all of 0.12–0.15). Reuse falls back to an association that Paseo's branch
   naming breaks for issues. Decide: always create, or keep the old path on those hosts.
7. **Paseo 0.17.** `v0.17.0-beta.1` shipped 2026-11-19 and 0.16.2 on 2026-11-18. The record read 0.16.0.

## Top 5 changes before any implementation PR

1. **Fix the fork story.** Delete "`projectPath` is the second pin". State that Paseo tries `origin` then
   `upstream`. Make the spike test fork+upstream with a same-numbered PR on the fork, and decide the fallback
   (refuse when the matching remote is not `origin`, or a `worktree.create.before` hook, which is what would force
   `^0.15`).
2. **Redesign the idempotency key and keep the UI lock** (E5). Drop the claim that the daemon replaces UI guards.
3. **Correct the census** (E1): add F41, F05, F11/F12, F13, F27 and the PR-path half-created worktree; give F41
   and F13 a PR each.
4. **Drop the floor raise; state the base.** No `^0.15`. Say `9f1f053` is #662's head and settle #662.
5. **Rewrite Move 3's settings and `ComboSelect` fixes** (E7): `saving`-flag queue, commit mode as a prop, and
   the three rules `resolveRoutedLaunch` must obey.

## Go / no-go per move

| Move | Call | Smallest safe first step |
|---|---|---|
| Move 1 (delegate checkout) | **No-go as written.** Go after the spike and the maintainer's answer | The spike only: one isolated daemon at the lowest supported Paseo and one at 0.16.2; `workspaces.create` for a PR in (a) a plain clone, (b) fork+upstream; record branch name, fetched commit, error text for a throwing `worktree.create.after`, and whether an existing row links to the new workspace. No code merged |
| Move 2 (routing trust) | **Go now** | PRs 1+2 as one: move, kill the group on `exit`, per-hop symlink check, bounded queue on `run.finished` |
| Move 3 (client logic) | **Go, with E7's corrections** | PR 3 alone: `buildRoutingPayload` and `resolveRoutedLaunch` as pure functions with table tests that include the model-only-after-provider-edit case and replace-not-merge |

## What I could not verify

- Which repository `gh repo view` resolves in a fork+upstream checkout (it decides whose PR metadata Paseo reads).
- That 0.12–0.14 daemons perform change-request checkout like 0.16 (schema and client method checked; daemon
  code not).
- Whether 0.16.1, 0.16.2 or 0.17.0-beta.1 changed `WorkspaceCreateRequest` or the hooks (local tags stop at
  `v0.16.0-beta.2`; the installed types are 0.16.0).
- The body of the rollback after a throwing `worktree.create.after` (doc and call site read only).
- Whether Paseo renames a branch-off branch after the first prompt
  (`maybeMarkFirstAgentBranchAutoNameEligible`, `paseo-worktree-service.ts:92`); if it does, issue association by
  branch breaks again.
- Whether `createWorktreeCore` can return an existing worktree (`created: false`, same file `:83-84`), which
  would change the reuse story.
- The half-created PR worktree's exact failure (inferred from git's empty-index behaviour; not run).
- That the 30 s plugin request timeout applies to client-to-plugin RPCs, including the 120 s clone.
- F39 upstream, F79/F89 fixed, and the keyed-agent-create-with-prompt rejection: reported by the sub-agents with
  file:line; I did not re-read those lines.
- No tests, builds or daemons were run. This is a read-only review.

Closing search: run last, 2026-11-20. `gh release list -R getpaseo/paseo`: latest is v0.16.2 (2026-11-18) and
v0.17.0-beta.1 is out (2026-11-19); the record and this review read 0.16.0 types and `v0.16.0-beta.2` source, so
the spike must re-check the create request on the current release. `gh pr list -R omercnet/paseo-plugins --state
open` and `gh pr view 662`: #662 is open, mergeable, head `9f1f053`, +96/−1 in 5 files, which is the tree the
record calls "the head"; no other open PR touches pr-radar.
DONE
