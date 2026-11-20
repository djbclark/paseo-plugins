STATUS: done

# PR Radar re-architecture — design record

Architect: Fable 5.2, 2026-11-20. Design only; nothing in any repo was changed.
Code read: `/Users/djbclark/src/paseo-plugins-followup-wt` at `9f1f053` (`feat/pr-radar-routing-labels`).
Paseo API read: installed `@getpaseo/plugin`, `@getpaseo/client`, `@getpaseo/protocol` **0.16.0** type
declarations in that worktree's `node_modules`, plus `getpaseo/paseo` source and docs at tag `v0.16.0-beta.2`
(`docs/plugins.md`, `public-docs/plugins/hooks.md`, `packages/protocol/src/messages.ts`,
`packages/server/src/server/creation/index.ts`, `packages/server/src/utils/worktree.ts`).
ade-claim read: `~/src/djbclark-ade/plugins/ade-claim/` at `77645420` (README, `index.server.ts`, `server/hook.mjs`,
`claim-lib.mjs` `planWorktreeClaimCreate`), `docs/git-guard.md`.

Paths below are relative to `pr-radar/` unless they start with `/` or `~`. `Fnn` numbers index the 90 findings
sorted by path then line; the table that maps `Fnn` to GitHub comment ids is in the appendix.

Prior art / docs checked: Paseo plugin docs and hook reference at `v0.16.0-beta.2` (local git, read before designing);
found native change-request worktree creation, `worktree.create.before/after` hooks and creation idempotency keys,
which change Move 1 (below).

## Headline

1. **Move 1 is right in direction and wrong in mechanism.** `ade-claim` cannot be asked for a worktree and cannot be
   imported. It is a hook listener: it claims whatever worktree **Paseo** creates. The seam that works is
   *pr-radar asks Paseo to create the workspace; ade-claim, when installed, claims it through Paseo's hooks*.
   Paseo 0.16 already does PR checkout into a worktree natively, with fork handling, hooks and an idempotency key.
2. **The "~30 findings" number for `workspace-ops.ts` is stale.** 31 findings point at that file; at the current
   head I count 5 still live (F60, F63, F66, F67, F74) and one declined (F72). The rest were fixed in place before
   merge, which is why the file grew to 1,361 lines. The case for Move 1 is deleting about 575 lines of
   security-critical git code and about 1,500 lines of tests that pin it, and removing three defects that cannot be
   fixed locally (F63, F66, F67).
3. **Move 2 is nearly done already.** Ownership and parent-directory checks exist. Four small defects remain
   (one of them not in the inventory).
4. **Move 3 should not be a component split.** 7 client defects are live. They come from one stale closure, one
   save queue and one duplicated code path. Extracting two pure functions and one launch entry fixes them; splitting
   components would not.
5. **Of the 90 findings, about 60 are already closed at the head.** 8 disappear with deletion, about 14 are worth
   cherry-picking, 5 are test additions, 2 belong to djbclark-ade (section F, Q8).

## A. Verifying the characterization

### A(a) `workspace-ops.ts` hand-rolls git — confirmed, with corrections

Confirmed. The file runs `git` through `execFile` for every step:

| Lines | What it does |
|---|---|
| `server/workspace-ops.ts:39-115` | environment allow-lists for scripts, `gh` and `git` |
| `:137-291` | `GIT_SAFETY_ARGS`, per-call enumeration of `filter.*`/`hook.*` config, `runGit` |
| `:298-306` | `cloneGitHubRepo` (`clone --no-checkout`, then `checkout --force`) |
| `:384-481` | remote parsing, remote matching, `worktree list` parsing, clone-path choice |
| `:483-576` | `listLocalDirsHandler` (read-only directory and worktree discovery) |
| `:578-636` | `fetchPrHead` into `refs/pr-radar/pull/N`, transport-config guard |
| `:638-891` | branch fast-forward guard, ignored-file guard, gitlink guard, issue start point, worktree refresh |
| `:893-1153` | `prepareWorkspaceCwdHandler`: clone, CWD-BASE branch switch, worktree reuse, worktree create, cleanup |
| `:1155-1361` | routing script: trust check, capped spawn, single-run guard |

Status of the 31 findings on this file at `9f1f053`:

| Status | Findings | Evidence |
|---|---|---|
| Fixed in place | F44, F45 | env allow-list, tokens kept from git (`:80-115`, `:199-210`) |
| | F47 | `GIT_SAFETY_ARGS` and filter/hook neutralizing (`:140-187`, `:217-270`) |
| | F48 | `assertBranchFastForwards` (`:655-669`, called `:861`, `:986`, `:1127`) |
| | F49, F56 | dedicated ref instead of `FETCH_HEAD` (`:578-591`) |
| | F52 | reuse excludes the base checkout (`:1028-1030`) |
| | F53 | `refreshPrWorktree` fetches and verifies (`:842-869`) |
| | F54 | exact `github.com` host for both remote forms (`:390-397`) |
| | F55 | existing issue branch checked out without `-B` (`:999-1002`) |
| | F57 | stdin error handler (`:1321`) |
| | F58, F59, F64 | ignored-file and gitlink guards on every base checkout (`:696`, `:1000-1006`) |
| | F61, F68, F69 | failed checkout removes the worktree; reuse re-materializes (`:876-891`, `:1117-1124`) |
| | F62 | worktree root confined to the source directory (`:941-950`) |
| | F70, F71 | owner and ancestor checks (`:1184-1208`) |
| | F73 | guard released from `run.finished`, not `finally` (`:1350-1354`) |
| Outdated / declined | F46, F50, F51 | maintainer replies; code agrees |
| Declined, small residue | F72 | `finishedGate` resolves right after sending SIGKILL (`:1272-1275`) |
| Partly fixed | F65 | start point is the remote default when recorded (`:802-815`); F60 is its residue |
| **Live** | F60 | without `origin/HEAD`, any published branch counts as a safe start (`:822-839`) |
| **Live** | F63 | branch names carry no repository identity (`:960-963`); two remotes in one checkout collide |
| **Live, structural** | F66 | CWD-BASE returns a shared path with no reservation (`:1015-1022`) |
| **Live, structural** | F67 | no serialization of `prepareWorkspaceCwdHandler`; concurrent runs race on `index.lock` (`:1094-1124`) |
| **Live** | F74 | descendants survive a successful script exit (`:1309` kills the group only when `terminating`) |

Where the proposal is wrong here:

- §3 lists shared `FETCH_HEAD`, stale-head reuse, `--no-checkout` sequencing, submodule/gitlink, dead-worktree
  cleanup, `XDG_CONFIG_HOME` and remote-host matching as things delegation would kill. All are already fixed or
  declined at the head. Delegation makes them irrelevant; it does not fix anything that is still broken there.
- The live set is F60, F63, F66, F67 (checkout) and F74 (routing). Only F66 and F67 are structural.
- The cost that matters is not open findings. It is 13 guard functions that each exist because one review round
  found one more way for `git checkout` in a shared directory to lose data. That list has no natural end.

### A(b) Routing-trust surface — mostly already closed

- The script path comes from registered plugin settings, never RPC input (`index.server.ts:23-30`,
  `server/workspace-ops.ts:1342-1343`).
- `resolveRoutingScript` (`:1164-1215`) requires an absolute or `~/` path, a regular file after `realpath`, owner
  = daemon user or root, no group/other write bit, every ancestor of the real path owned by the user or root and
  not group/other-writable unless sticky, and the execute bit. F70 and F71 are fixed.
- `startScriptCapped` (`:1224-1325`) runs it with no arguments, an allow-listed environment (`:1236`), its own
  process group, a 30 s timeout, a 1 MiB output cap and a SIGTERM-then-SIGKILL ladder.
- One script at a time: a module-level boolean (`:1327`, `:1344-1347`), released when the process group is gone or
  has been sent SIGKILL (`:1351-1354`).
- The proposal's "~10–15 findings" is high. Seven findings touch this surface (F57, F70, F71, F72, F73, F74 server;
  F17 client). Four are fixed, one declined, two live. Section C lists what is left.
- Threat model, stated once: these checks defend against **other local users**. They do not defend against a Paseo
  client, and need not: any authenticated client can already start a yolo agent or a terminal on the host.
- The adapter is **not** byte-identical today. `cmp ~/src/djbclark-ade/bin/pr-radar-route examples/pr-radar-route.py`
  fails in both checkouts I compared (`paseo-plugins-followup-wt`: about 250 diff lines;
  `paseo-plugins-omercnet` branch `feat/pr-radar-launch-and-dprs`: differs from line 26). The ADE head moved after
  the last sync. That is a sync-process gap, not a design problem; it is listed in F.

### A(c) Client race class

The class is real but smaller than the proposal says, and its cause is not the monolith.

- 31 client findings checked at `9f1f053`: 22 fixed (4 only partly), 1 not a defect, 8 rows live that are 7
  distinct defects: F01, F09, F10, F15, F16, F18, F22 (= F25).
- Of the proposal's five named races: "auto-launch guard released early" is fixed (`client/pr-radar.tsx:409-423`);
  "routed model after provider edit" is fixed (`client/start-agent-modal.tsx:212-219`); "stale routing after
  disable" is fixed for the pending result (`:271-275`) but already-applied values are not unwound; "rapid
  `ComboSelect` edits" is still live (`client/settings-screen.tsx:113-146`, `client/agent-launch-options.tsx:96-101`);
  "routed profile dropped on uncached config" is still live (`client/start-agent-modal.tsx:195`, `:286`).
- Root causes: a stale closure over `profiles` in one effect; a save that resends a render-captured revision while
  the input overwrites the user's draft; and routing-then-launch logic written twice with separate guards
  (`client/start-agent-modal.tsx:151-246` and `client/pr-radar.tsx:338-398`).
- The proposal says the races come from "one 700-line modal". The modal's own guards (sequence counter, row id,
  edited-field set) are sound. The duplication is between the modal and `PrRadar`, not inside the modal.

Detail and the per-finding table: section D and `/Users/djbclark/src/.paseo-fanout/pp-rearch/client-races.md`.

## B. Move 1 — delegate checkout and ownership

### B.1 Why "ask ade-claim for a worktree" cannot be built

- `ade-claim` exposes no request API. Its entry registers four hooks and returns a disposer
  (`~/src/djbclark-ade/plugins/ade-claim/index.server.ts`, `server/hook.mjs:462-475`).
  `registerWorktreeClaimHook` reacts to `worktree.create.after` (`hook.mjs:222-267`). It never creates a worktree.
- Paseo has no plugin-to-plugin dependency or import. `docs/plugins.md` ("SDK import boundaries") says the compiler
  rejects a relative import to any code file outside the plugin's `client/`, `server/` and `shared/` directories, and
  nothing in the document describes one plugin depending on another. "Importable as a plugin dependency" in the
  brief does not match the SDK.
- pr-radar is `omercnet/paseo-plugins`, a public upstream plugin. It must not require a private toolchain.
- `planWorktreeClaimCreate` (`claim-lib.mjs:293-311`) only builds the argv for `ade_claims.py new`. Calling it from
  pr-radar would mean pr-radar creates the worktree itself and then claims it, which is the fork the lead ruled out.

### B.2 The seam that exists: Paseo creates, ade-claim claims

Paseo 0.16 creates change-request worktrees itself:

- `WorkspaceCreateRequest.source` accepts
  `{ kind: "worktree", cwd | projectId, action: "checkout" | "branch-off", refName, branchName, checkoutSource: { kind: "change_request", forge?, number, projectPath? }, worktreeSlug }`
  (`packages/protocol/src/messages.ts:2921-2965`, `:1323-1328`).
- The plugin client can call it: `PluginPaseoApi extends PaseoApi`, and `PaseoApi.workspaces.create(options)` takes
  `PaseoWorkspaceCreateOptions = Omit<WorkspaceCreateRequest, "type" | "requestId" | "agent" | "subscribe"> & { agent?, onEvent? }`
  (`@getpaseo/plugin/dist/client/reference-api.d.ts`, `@getpaseo/client/dist/index.d.ts:92-96`, `:165`).
  `idempotencyKey`, `metadata` and `title` pass through.
- The same request can carry the first agent (`agent: { config, labels, metadata, env, … }`), so workspace and agent
  are created by one daemon operation.
- Creation order is fixed: `worktree.create.before` (waits, may change path and branch) → `git worktree add` →
  `worktree.create.after` (waits; a throw aborts creation and Paseo removes the new worktree) → workspace
  registration → agent (`public-docs/plugins/hooks.md`, "worktree.create.before", "worktree.create.after").
- For a change request Paseo resolves the head through its forge service, fetches `refs/pull/N/head`, picks a unique
  local branch, and sets a push remote for forks (`packages/server/src/utils/worktree.ts:1576-1637`,
  `packages/server/src/server/resolve-worktree-creation-intent.ts`).
- Creation is idempotent per key: same key and same request returns the existing creation; same key and a different
  request fails with `workspace_request_key_conflict` (`packages/server/src/server/creation/index.ts:64-72`).

So the design is:

```text
pr-radar client ── rpc resolveBaseCheckout ──▶ pr-radar server   (read-only git + optional clone)
pr-radar client ── paseo.workspaces.create ──▶ Paseo daemon      (fetch, worktree add, branch, agent)
                                                  │ worktree.create.after / agent.open.before / …
                                                  ▼
                                              ade-claim           (claim, git-guard install, owner bind)
```

pr-radar never names ade-claim. With ade-claim absent, everything works unclaimed. With it present, every pr-radar
worktree is claimed because it is a Paseo worktree.

### B.3 What pr-radar asks for and what it receives

**Step 1, server RPC `resolveBaseCheckout`** (replaces `prepareWorkspaceCwd`):

- Input: `{ repository, targetDir?, allowClone }`. Output: `{ baseRoot, remoteName, cloned }`.
- It keeps today's validation and nothing else: slug check (`:370-375`), path confinement to `sourceCodeDir`
  (`:329-337`, `:904-917`), directory and git checks (`:930-950`), remote match (`:408-432`), optional clone
  (`:298-306`, `:920-928`).
- It runs no fetch, no checkout of a PR, no `worktree add`, no branch switch.

**Step 2, client call** (PR, worktree mode):

```ts
paseo.workspaces.create({
  idempotencyKey,                                  // one per launch intent, see below
  source: {
    kind: "worktree",
    cwd: baseRoot,
    action: "checkout",
    checkoutSource: { kind: "change_request", forge: "github", number: row.number, projectPath: row.repository },
  },
  metadata: { "pr-radar.item": `github:${row.repository}#pr:${row.number}` },
  agent: { config, labels, metadata },             // built by the existing buildAgentConfig
});
```

Issue, worktree mode: `source: { kind: "worktree", cwd: baseRoot, action: "branch-off", branchName }` with no
`refName`, so Paseo uses the project's default base. `branchName` includes the repository
(`pr-radar/<owner>-<repo>-issue-<n>`), which closes F63; Paseo makes it unique if it exists.

**Receives:** a `PaseoWorkspaceHandle` (`id`, `directory`) on success, or a rejected promise whose message names the
failing stage. A hook failure names the plugin and the hook.

**Idempotency key:** `pr-radar:<hostId>:<repository>#<kind>:<number>:<launchNonce>`. `launchNonce` is minted once per
launch intent (modal open, or auto-launch button press) and reused for every retry of that intent. A double press
sends the same key and the same request, so the daemon returns one creation. This closes the double-launch class
(F05, F08) in the daemon instead of with UI guards.

### B.4 Refusal and expiry

| Event | What happens | pr-radar behaviour |
|---|---|---|
| ade-claim in observe mode (today's default; no mode file on this machine) | hook logs the planned argv, never throws (`hook.mjs:250-253`) | nothing visible |
| Claim refused in enforce mode (`ade_claims.py new` exit 4, path held by another owner) | hook throws (`hook.mjs:254-258`, `:261-264`); Paseo aborts and removes the worktree | show the error text as the launch failure. No retry, no fallback to an unclaimed directory, no cleanup: nothing was left behind |
| `worktree.create.before` placement hook rejects or times out (30 s) | creation fails before `git worktree add` | same |
| Forge lookup or fetch fails | creation fails | same; the message is Paseo's |
| Claim expires (TTL 72 h, read-time, `docs/git-guard.md:162-165`) | the workspace reads as unclaimed; another owner may claim the path | pr-radar does not look. It keys reuse on the Paseo workspace record, not on the claim |
| Workspace archived | Paseo removes its own worktree; ade-claim releases the claim (`hook.mjs:276-325`) | nothing; pr-radar never deletes a worktree |

pr-radar treats every refusal the same way: creation failed, show why. It never inspects claims.

### B.5 Reuse and cleanup when another component owns them

- **Reuse is a record lookup, not a git query.** Before creating, the client looks for a non-archived workspace
  for the item: first `metadata["pr-radar.item"]`, then the association the radar already computes for the row
  (`row.workspaceIds`). If one exists, launch the agent there with `paseo.workspaces.ref(id).agents.create(...)`.
  `git worktree list` matching on `pr-radar/pr-N` (`:1027-1052`, `:1063-1092`) is deleted.
- **A reused worktree is not refreshed by the daemon.** `refreshPrWorktree` (`:842-869`) rewrites a directory an
  agent may own; it is the source of F48, F53, F58 and F64 and it contradicts one-writer ownership. The launch
  prompt tells the agent the PR number; the agent fetches. This is a behaviour change (listed in F).
- **Second agent in a claimed workspace.** With enforcement on, `agent.open.before` exports `ADE_CLAIM_ID` only when
  the claim is unbound or owned by that agent (`hook.mjs:361-371`), so a second agent gets no claim identity. That
  is correct one-writer behaviour and it is ade-claim's rule to own. pr-radar's part: when the row already has a
  live agent, the primary action is the existing "Open Existing Agent"/ask-agent path, not "Start New Agent".
  Whether a claim whose owner agent is archived should rebind to a new agent is an ade-claim question (F, Q4).
- **Cleanup.** None in pr-radar. Failed creation is rolled back by Paseo. Normal teardown is workspace archive.
- **Placement.** Today's sibling placement (`dirname(baseRoot)/<repo>-pr-N-slug`, `:1054-1061`) and the
  `worktreeName` input go away. Paseo places worktrees; a user who wants siblings installs a
  `worktree.create.before` placement plugin (Paseo ships an example). This is a behaviour change (F, Q2).
- **Existing `pr-radar/pr-N` worktrees** made by the old code stay as ordinary directory workspaces. They keep
  showing on their rows and can be reused as existing workspaces. Paseo does not own them and will not delete them.
  No migration code.

### B.6 What is deleted and what is kept in `workspace-ops.ts`

| Lines | Fate |
|---|---|
| `:578-591` `prRef`, `fetchPrHead` | delete |
| `:593-636` `effectiveConfigKeys`, `assertNoUnsafeTransportConfig` | delete |
| `:638-669` `branchExists`, `assertBranchFastForwards` | delete |
| `:671-790` ignored-file guard, `runGitLossless`, gitlink guard | delete |
| `:792-891` `currentBranch`, `resolveIssueStartPoint`, both `refresh*Worktree` | delete |
| `:893-1153` `prepareWorkspaceCwdHandler` | replace with `resolveBaseCheckout` (about 60 lines: `:897-958` minus mode handling) |
| `:217-270` `configuredFilterSafetyArgs` and its use in `runGit` (`:273-284`) | delete once only `remote`, `rev-parse`, `worktree list` and `clone --no-checkout` remain; keep the static `GIT_SAFETY_ARGS` |
| `:344-363`, `:377-382` `slugifyBranch`, `defaultWorktreeName`, `isValidWorktreeName` | delete with the `worktreeName` input |
| `:384-436` `parseGitHubRepoFromRemote`, `findMatchingRemote` | **keep** |
| `:298-306`, `:473-481` clone and clone-path choice | keep (see F, Q3) |
| `:483-576` `listLocalDirsHandler` | keep unchanged in this move |
| `:39-131`, `:189-210`, `:294-296` env allow-lists, `ghEnv`, path helpers | keep (`ghEnv` is used by `server/viewer-scope.ts`) |
| `:1155-1361` routing script | move to `server/routing-script.ts` (Move 2) |

Net: about 620 of 1,361 lines deleted, about 210 moved, about 530 left, of which about 60 are new.
Tests: most of `tests/workspace-ops-handlers.test.ts` (1,664 lines) and part of `tests/workspace-ops.test.ts` (413)
go with the code they pin.

**Is repo matching still needed?** Yes. Paseo checks out "change request N of the project at `cwd`". pr-radar must
still prove that `cwd` is a checkout of `row.repository` before asking, or PR 7 of another repository is checked
out (the F54 class). `projectPath: row.repository` in `checkoutSource` is the second pin; its effect on fork
checkouts must be confirmed in the spike (E, PR 0).

**Where the host boundary falls:**

- pr-radar server: reads git (`remote`, `rev-parse`, `worktree list`), may clone into `sourceCodeDir`, runs the
  routing script, queries `gh`. It never changes a branch, index or working tree of an existing checkout.
- Paseo daemon: every fetch, `worktree add`, branch creation, push-remote setup, worktree removal.
- ade-claim: ownership, through hooks only.

### B.7 Does `@getpaseo/plugin` expose enough?

Yes for 0.16, with three things to confirm before coding (E, PR 0):

1. **Version floor.** pr-radar declares `^0.12.0 || … || ^0.16.0` (`paseo-plugin.json`). `checkoutSource` and
   `idempotencyKey` are in the protocol schema at `v0.12.0`; the worktree hooks first appear at `v0.15.0`; workspace
   `metadata` at 0.16. Whether `paseo.workspaces.create` is on the plugin client API across 0.12–0.14 I did not
   verify. Recommendation: raise the floor to `^0.15.0 || ^0.16.0` in the same PR and treat metadata as optional
   (fall back to row association).
2. **Fork checkouts.** pr-radar prefers an `upstream` remote over `origin` (`:417-421`). Paseo defaults to `origin`
   unless its forge service returns `checkoutRefs`. Confirm PR N resolves against `row.repository` when `origin` is
   a fork.
3. **Error surface.** Confirm the rejection text when a `worktree.create.after` hook throws is readable enough to
   show as is.

**Fallback if any of the three fails:** keep `prepareWorkspaceCwd` for that case only, behind the same client
function, and still delete the CWD-BASE branch switch and the daemon-side refresh. If fork resolution is the
failing one, a second option is a pr-radar `worktree.create.before` hook that sets `checkoutRefs.remoteName` for
requests whose `sourceCheckout` it just resolved; that is hook code, not git code.

### B.8 Migration order

1. Spike on an isolated daemon (PR 0, not merged).
2. Add the native path for PR + CWD-WORKTREE behind the existing launch function; old path stays for everything else.
3. Add the native path for issue + CWD-WORKTREE.
4. Decide CWD-BASE (F, Q1) and change it.
5. Delete the old handler, guards, RPC contract and tests in one pure-deletion PR.

## C. Move 2 — minimal routing-trust fix

The synced adapter `examples/pr-radar-route.py` and `route_agent.py` are not touched by anything here.

| # | Defect | Where | Smallest correct fix |
|---|---|---|---|
| C1 | Ownership never verified (F71) | `:1184-1189` | **Already fixed.** No change. Keep the `uid` parameter test |
| C2 | Parent-directory writability (F70) | `:1195-1208` | **Already fixed** for the real path. See C5 for the gap |
| C3 | Guard released while the script still runs (F73) | `:1350-1354` | **Already fixed.** No change |
| C4 | Descendants outlive a successful run (F74) | `:1307-1311` | In the `close` handler call `signalGroup("SIGKILL")` unconditionally (drop `if (terminating)`). One line. A descendant that kept the pipes open already holds `close` until the timeout path kills the group |
| C5 | Configured path's own chain is not checked (**not in the inventory**) | `:1179`, `:1195` | Only ancestors of `realpath(expanded)` are checked. If the configured path contains a symlink in a directory another user can write, that user can repoint it at any executable owned by the daemon user or root, which then passes every check and runs. Fix: run the same owner/write-bit loop over the lexical ancestors of `expanded` too, using `lstat`, before `realpath`. About 12 lines, reusing `trustedOwner` |
| C6 | Guard released before a killed group has exited (F72, declined upstream) | `:1272-1275` | Optional. After sending SIGKILL, resolve `finishedGate` on `close` or after one more `killGraceMs`, whichever is first. Three lines. I would not reopen it with the maintainer on its own; fold into C4's PR only if welcome |
| C7 | Second request is rejected, not queued | `:1344-1346` | The single-run guard throws "already running". Two rows launched close together make the second fall back to defaults. Fix: replace the boolean with a promise chain (`routingTail = routingTail.then(run, run)`) so calls serialize; keep the existing 30 s timeout per run as the bound. About 8 lines. Functional, not security |
| C8 | Settings offers directories as routing scripts (F17, client) | `client/settings-screen.tsx` | Covered in D |

Not worth fixing:

- **Check-then-exec by path** (`:1210-1214` then `:1234`). After C5, only the daemon user or root can swap the file
  between check and exec. Executing a validated file descriptor is not portable for `#!` scripts on macOS.
- **POSIX ACLs** are not read. Document it in the README line that already describes the check.

The proposal's alternative, "restrict the plugin to the synced preset", should be dropped. Paseo 0.16 does not tell
a plugin its install directory (`PluginServerContext` has no source path; ade-claim works around the same gap,
`hook.mjs:30-60`), so the bundled copy cannot be located reliably. It would also break pointing the setting at the
live `~/src/djbclark-ade/bin/pr-radar-route`, which is the configuration that keeps the router current.

Move 2 ships first: it is independent of Moves 1 and 3 and it is the only security-relevant live item.

## D. Move 3 — client split

Evidence: `/Users/djbclark/src/.paseo-fanout/pp-rearch/client-races.md` (per-finding verdicts, state inventory and
duplication map for the client at `9f1f053`, produced by a separate read-only verification pass; I spot-checked it
against the findings and the server code but did not re-read every client line myself).

### D.1 Recommendation: do not split components to fix races

Of the 31 client findings checked, 22 are fixed, 1 is not a defect, and 7 distinct defects are live. The live ones
sit in three pockets. None is caused by the viewer, routing, launch and settings code sharing a component. A
four-way component split would move the same closures and guards into new files and would not remove one race.

What does remove them is moving **logic**, not components: one pure function for "apply a routing result", one
launch entry, and a draft-owning text input.

### D.2 The split by responsibility, and what each part removes

| Part | Change | Removes |
|---|---|---|
| Routing | New pure functions (in `client/radar.ts` beside `routedProviderModel`, or `client/routed-launch.ts`): `buildRoutingPayload(row, settings, target)` and `resolveRoutedLaunch(current, routed, profiles, edited)` returning a delta. Both call sites use them: `client/start-agent-modal.tsx:151-246` and `client/pr-radar.tsx:338-398`, which today hold two copies | F06 drift (auto path compares against `""`, `client/pr-radar.tsx:386`); F19 residue and R4 (a delta can be undone; setters cannot); F24 residue (the function takes the current profile as an argument instead of closing over it) |
| Launch | One launch entry (a `useStartAgent` hook) holding the synchronous lock and the idempotency key from B.3. The modal's Start and the auto path both call it. Today the modal calls `mutate` directly (`client/pr-radar.tsx:1301`) and skips `autoLaunchingRef` (`:310`) | R1 (modal double-click); the F05/F08 class for good, because the daemon dedupes on the key; F13's adjacent gap (one config read instead of `:268` plus `:381`); F04 residue (identity moves into creation `metadata`) |
| Modal readiness | The routing effect waits for profiles: await the config query inside `handleRunRouter` before vetting, instead of closing over `profiles` (`client/start-agent-modal.tsx:140-257`, effect deps `:286`) | F22 = F25; the first-open cases of F23 and F26; R3 if the reset effect also waits for loaded settings (`:104-125`) |
| Settings | `ComboSelect` owns its draft and stops resetting it from an older acknowledged value (`client/agent-launch-options.tsx:96-101`); free-text fields save on commit (blur or Enter), not per keystroke; `savePatch` sends the next save only after `settings.revision` has changed (`client/settings-screen.tsx:113-146`) | F01, F16, F18 |
| Viewer | No change. No race was found in `client/radar.ts` row building or `client/use-radar.ts` | F10, F15 and F09 are one-line fixes, not races |

### D.3 Cheaper than a split?

Yes. The table above is five small changes, three of them under 100 lines, all testable as pure functions in the
existing `tests/routed-launch.test.ts` pattern. A component split of `PrRadar` (`client/pr-radar.tsx:98-1314`, one
function of about 1,200 lines) is worth doing later for readability: styles, the row renderer and the dead
`start`/`checkout` branches of `agentMutation` (`:195-230`) can move or go mechanically. It should be its own PR
after the logic moves, and it should be sold as readability, not as a race fix.

No state-machine library. The modal's sequence counter and edited-field set already do that job
(`client/start-agent-modal.tsx:96-98`, `:138`, `:192`); the bugs are in what the handler closes over.

## E. Ordered implementation plan

Checks run in `pr-radar/` through `~/ops/site-private-djbclark/bin/bgrun` after `load-watch ci decide`:
`bun run typecheck`, `bun run check` (biome), and `bun run test -- <the named test file>`. PRs that change runtime
behaviour also get the repo's isolated-daemon run (`deliver` skill, `dev-daemon.sh up`), never the default daemon.
"Closes" lists findings that are live today.

| # | Scope | Files | Proof | Closes |
|---|---|---|---|---|
| 0 | **Spike, not merged.** On an isolated daemon: call `paseo.workspaces.create` from the plugin client for a PR; confirm the three items in B.7 and that a second call with the same key returns the same workspace; with ade-claim installed in observe mode, confirm one `worktree.create.after` line in its JSONL | none | half a page of notes pasted into PR 5's body | decides B.7 |
| 1 | **Pure move.** Routing script code to `server/routing-script.ts`; its tests to `tests/routing-script.test.ts` | `server/workspace-ops.ts:1155-1361`, `index.server.ts`, tests | typecheck; moved tests pass unchanged | none |
| 2 | Routing trust: C4, C5, C7 (C6 if the maintainer wants it) | `server/routing-script.ts`, its test | new tests: a backgrounded descendant is gone after a successful run; a configured path through a symlink in a group-writable directory is refused; two concurrent calls both return, in order | F74 (F72 optional) |
| 3 | Extract `buildRoutingPayload` and `resolveRoutedLaunch`; both call sites use them | `client/radar.ts` or new `client/routed-launch.ts`, `client/start-agent-modal.tsx`, `client/pr-radar.tsx`, `tests/routed-launch.test.ts` | new table tests, including provider-only result with `defaultProvider: ""` on both paths | F06 drift, F19 residue, R4 |
| 4 | Modal waits for profiles and loaded settings before routing | `client/start-agent-modal.tsx` | test of the extracted "route when ready" function with profiles arriving late | F22, F25, F23/F26 first-open, F24 residue, R3 |
| 5 | Single launch entry with lock and idempotency key. **Delete** the unreachable `start`/`checkout` branches of `agentMutation` | `client/pr-radar.tsx`, new `client/use-start-agent.ts` | unit test of the key builder; isolated daemon: double-click Start yields one agent | R1, F13 adjacent |
| 6 | Native worktree for PR rows: `resolveBaseCheckout` RPC; `workspaces.create` with `checkoutSource`; reuse by workspace record; `pr-radar.item` metadata; raise the Paseo floor | `shared/viewer-scope.ts`, `server/workspace-ops.ts`, `index.server.ts`, `client/use-start-agent.ts`, `client/radar.ts`, `paseo-plugin.json` | handler tests for `resolveBaseCheckout` (reusing today's fixtures for confinement and remote match); pure test of `buildWorkspaceCreateOptions`; isolated daemon: PR row launches into a Paseo worktree, second launch reuses it | F67; F66 for worktree mode |
| 7 | Native worktree for issue rows: `branch-off`, branch name with repository; row association reads metadata first, then old and new branch patterns | `client/use-start-agent.ts`, `client/radar.ts:385-414`, `tests/radar.test.ts` | association tests for old branch, new branch and metadata | F60, F63, F65 residue |
| 8 | CWD-BASE per decision Q1 (recommended: open the checkout as is, no branch switch) | `client/use-start-agent.ts`, `client/start-agent-modal.tsx` (warning text), README | isolated daemon: base launch leaves `git status` and the branch unchanged | F66 fully |
| 9 | **Pure deletion.** `prepareWorkspaceCwd` contract and handler, all checkout guards, dynamic filter enumeration, `worktreeName`, and their tests | `server/workspace-ops.ts` (about −620), `shared/viewer-scope.ts:100-120`, `tests/workspace-ops-handlers.test.ts` and `tests/workspace-ops.test.ts` (about −1,500), modal field | typecheck proves no caller remains; remaining tests pass | supersedes F77, F79, F80, F89 |
| 10 | Settings draft and commit-save; honest error text for a failed save | `client/agent-launch-options.tsx`, `client/settings-screen.tsx` | test of the save queue with a fake `save` that bumps the revision late | F01, F16, F18 |
| 11 | One-liners: use one shared remote parser on the client (move `parseGitHubRepoFromRemote` to `shared/`); "Agent failed" reason on issue rows; bound the action group width | `shared/`, `client/radar.ts:187-190`, `:1064`, `client/pr-radar.tsx:701-709` | `tests/radar.test.ts` cases for a look-alike host and a failed-only issue row | F10, F15, F09 |
| 12 | Test cherry-picks | `tests/` | the tests themselves | F03, F75, F76, F81, F88 |
| 13 | Optional. Mechanical split of `client/pr-radar.tsx` (styles, row renderer) | `client/` | typecheck, no test change | none |

Ordering rules:

- 1 → 2 first. They are independent of everything else and carry the only live security item.
- 3 → 4 → 5 → 6 → 7 → 8 → 9 in that order. 3, 4, 5 and 9 all edit `client/start-agent-modal.tsx` or the launch
  mutation, so they must not run as parallel slices.
- 10, 11, 12 are independent of the chain and of each other. They can run beside it.
- 0 must finish before 6 starts. If the spike fails on fork checkouts, 6 uses the fallback in B.7 and 9 shrinks.
- Each PR follows the repo's `deliver` skill (issue first, review gates, conventional title for release-please).

## F. Risks and open questions

### Decisions for djbclark

| # | Question | Recommendation |
|---|---|---|
| Q1 | CWD-BASE today switches the base checkout to `pr-radar/pr-N` after a dirty check (`server/workspace-ops.ts:965-1023`). Keep that? | Stop switching. CWD-BASE opens the checkout as it is and the prompt names the PR. Branch switching in a shared checkout is the source of F66 and of four guard functions, and it contradicts ADE standing order 9 (work in a claimed workspace, never a shared checkout). Alternative: remove the mode |
| Q2 | Worktrees move from "sibling of the base checkout inside `sourceCodeDir`" to wherever Paseo puts them; the `worktreeName` field goes | Accept. Anyone who wants siblings uses a `worktree.create.before` placement plugin |
| Q3 | Keep "clone the repository when no checkout exists" inside pr-radar? | Keep. It is about 10 lines plus the existing path confinement, and Paseo's plugin API has no clone call that I found |
| Q4 | For the ade lead: when a pr-radar launch puts a new agent into a workspace whose claim is owned by an archived agent, should ade-claim rebind? Also `paseo-plugin.json` of ade-claim says `>=0.14.0`, but I find the worktree hooks only from tag `v0.15.0` | ade-claim's to decide; pr-radar needs no change either way |
| Q5 | Raise pr-radar's Paseo floor from `^0.12` to `^0.15`? This is the maintainer's call | Yes, in PR 6 |
| Q6 | The daemon stops refreshing a reused PR worktree to the latest head | Accept. The agent fetches. Refresh by the daemon rewrites a directory another writer owns |
| Q7 | Dropping pr-radar's git hardening (`GIT_SAFETY_ARGS`, filter and transport guards) for mutating commands, because pr-radar no longer runs them. PR checkout then runs under Paseo with the user's normal git config, exactly like Paseo's own New Workspace PR checkout | Accept, and say so in the PR. The maintainer reviewed that hardening in #618 and must agree to its removal |
| Q8 | The 90 findings: close as superseded, or cherry-pick? | Neither wholesale. About 60 are already fixed, outdated or declined at the head: reply with the commit and resolve. 8 are superseded by deletion (F60, F63, F66, F67, F77, F79, F80, F89): resolve when PR 9 merges, not before. 1–2 go to PR 2 (F74, optionally F72). 12 are cherry-picked in PRs 3, 4, 10, 11 (F01, F06, F09, F10, F15, F16, F18, F19, F22, F24, F25, F26). 5 are test cherry-picks in PR 12. F36 and F38 are about `route_agent.py` and belong to djbclark-ade, not pr-radar |
| Q9 | The adapter copy has drifted from `~/src/djbclark-ade/bin/pr-radar-route` in both checkouts I compared | Re-sync as its own chore PR. Point the live `routingScriptPath` setting at the ADE script so the daemon never runs the copy. Not part of this re-architecture |

### What could make this fail

- **Upstream does not want it.** This is omercnet's repository. PRs 6–9 remove behaviour (sibling placement,
  CWD-BASE switching, daemon refresh, hardening) that was just reviewed and merged. Open one design issue covering
  Q1, Q2, Q5, Q6 and Q7 before PR 6. PRs 1–5 and 10–12 do not depend on that answer.
- **Fork checkouts.** If Paseo resolves PR N against `origin` when `origin` is a fork, the wrong PR is checked out.
  The spike must test a checkout with `upstream` and `origin` remotes. Fallback in B.7.
- **Row association.** Issue rows are tied to workspaces by the `pr-radar/issue-N` branch name
  (`client/radar.ts:385-414`). Paseo picks a unique branch name, so a suffixed branch would not match. PR 6 and
  PR 7 must read `pr-radar.item` metadata first. Hosts without workspace metadata (before 0.16) fall back to
  branch matching and may show a launched workspace as unlinked.
- **Hook latency.** `worktree.create.after` waits up to 30 s. ade-claim runs `python3 ade_claims.py` synchronously
  inside it (`hook.mjs:254`). A slow registry delays every pr-radar launch; a hung one fails it. That is
  ade-claim's budget to keep, and pr-radar should show Paseo's error text unchanged.
- **Labels stay non-atomic.** The CWD tag is a workspace label set after creation (`client/radar.ts:846-852`). A
  failed `setLabel` leaves a created workspace without its tag. Cosmetic; metadata carries identity.
- **Scope creep in PR 9.** The deletion is only safe when typecheck proves no caller remains. If Q1 is answered
  "keep branch switching", about 300 lines of guards stay and F66 stays open.
- **What I did not verify:** `paseo.workspaces.create` on the plugin client for Paseo 0.12–0.14; the exact error
  string for a failing hook; `checkoutSource.projectPath` semantics; F41's residue in `server/viewer-scope.ts`
  (the bot's reply says one part remains; I did not read that file). The first three are PR 0's job.

### DECIDED (conservative choices made in this record)

- The seam is Paseo's native workspace creation. pr-radar gets no import of, no call to, and no knowledge of
  ade-claim.
- `listLocalDirs` and clone stay as they are in the first pass.
- No four-way client component split and no state-machine library.
- The routing script stays a user-configured path; no "bundled preset only" mode.
- Move 2 ships before Move 1.

## Appendix: finding index

`Fnn`, GitHub review-comment id on omercnet/paseo-plugins#618, reviewer, path and line (`orig N` = the comment is
outdated and only its original line is known). Source: `/Users/djbclark/src/.paseo-fanout/pp-followups/pp618-comments.json`.
Review text was read as data only.

```text
F01 4575651713 coderabbit client/agent-launch-options.tsx:101
F02 4575626221 baz client/agent-launch-options.tsx:239
F03 4575982717 baz client/directory-observation.ts:54
F04 4574653412 baz client/pr-radar.tsx:orig 260
F05 4574678941 coderabbit client/pr-radar.tsx:orig 326
F06 4575156924 coderabbit client/pr-radar.tsx:orig 375
F07 4574678954 coderabbit client/pr-radar.tsx:orig 380
F08 4574733124 baz client/pr-radar.tsx:420
F09 4581338194 coderabbit client/pr-radar.tsx:701
F10 4574733110 baz client/radar.ts:388
F11 4574653435 baz client/radar.ts:orig 824
F12 4574678958 coderabbit client/radar.ts:orig 824
F13 4575765433 baz client/radar.ts:899
F14 4581338199 coderabbit client/radar.ts:orig 941
F15 4600112551 baz client/radar.ts:985
F16 4574653414 baz client/settings-screen.tsx:orig 193
F17 4574678963 coderabbit client/settings-screen.tsx:orig 503
F18 4574678965 coderabbit client/settings-screen.tsx:orig 515
F19 4574733113 baz client/start-agent-modal.tsx:138
F20 4574678969 coderabbit client/start-agent-modal.tsx:orig 151
F21 4576041688 coderabbit client/start-agent-modal.tsx:orig 179
F22 4576041683 coderabbit client/start-agent-modal.tsx:185
F23 4603661261 coderabbit client/start-agent-modal.tsx:orig 190
F24 4607839637 coderabbit client/start-agent-modal.tsx:orig 192
F25 4575765436 baz client/start-agent-modal.tsx:193
F26 4607839646 coderabbit client/start-agent-modal.tsx:orig 196
F27 4600112594 baz client/start-agent-modal.tsx:209
F28 4581338201 coderabbit client/start-agent-modal.tsx:219
F29 4581338205 coderabbit client/start-agent-modal.tsx:orig 220
F30 4600243003 coderabbit client/start-agent-modal.tsx:orig 234
F31 4575156934 coderabbit client/start-agent-modal.tsx:242
F32 4574678972 coderabbit client/start-agent-modal.tsx:305
F33 4574678977 coderabbit client/start-agent-modal.tsx:orig 560
F34 4574678983 coderabbit examples/pr-radar-route.py:orig 95
F35 4574653423 baz examples/pr-radar-route.py:orig 122
F36 4574678991 coderabbit examples/route_agent.py:orig 137
F37 4575430018 coderabbit examples/route_agent.py:orig 991
F38 4574678999 coderabbit examples/route_agent.py:orig 1068
F39 4575156936 coderabbit examples/route_agent.py:orig 1082
F40 4575430029 coderabbit examples/route_agent.py:orig 1106
F41 4574679006 coderabbit server/viewer-scope.ts:orig 205
F42 4574653420 baz server/viewer-scope.ts:502
F43 4574679010 coderabbit server/viewer-scope.ts:orig 587
F44 4574679016 coderabbit server/workspace-ops.ts:orig 44
F45 4581338208 coderabbit server/workspace-ops.ts:orig 83
F46 4600876032 baz server/workspace-ops.ts:99
F47 4581338211 coderabbit server/workspace-ops.ts:orig 121
F48 4575156941 coderabbit server/workspace-ops.ts:orig 319
F49 4574733116 baz server/workspace-ops.ts:orig 320
F50 4574653430 baz server/workspace-ops.ts:orig 347
F51 4603661267 coderabbit server/workspace-ops.ts:374
F52 4574679023 coderabbit server/workspace-ops.ts:orig 375
F53 4574653432 baz server/workspace-ops.ts:orig 379
F54 4574653425 baz server/workspace-ops.ts:401
F55 4575156946 coderabbit server/workspace-ops.ts:orig 401
F56 4574679030 coderabbit server/workspace-ops.ts:orig 430
F57 4574679039 coderabbit server/workspace-ops.ts:orig 489
F58 4576041692 coderabbit server/workspace-ops.ts:orig 493
F59 4576041703 coderabbit server/workspace-ops.ts:orig 662
F60 4576041699 coderabbit server/workspace-ops.ts:830
F61 4601446460 baz server/workspace-ops.ts:orig 856
F62 4581338216 coderabbit server/workspace-ops.ts:936
F63 4581338220 coderabbit server/workspace-ops.ts:963
F64 4603661280 coderabbit server/workspace-ops.ts:1002
F65 4574733120 baz server/workspace-ops.ts:1014
F66 4581338221 coderabbit server/workspace-ops.ts:1016
F67 4607915138 baz server/workspace-ops.ts:1043
F68 4603661285 coderabbit server/workspace-ops.ts:orig 1089
F69 4607768688 baz server/workspace-ops.ts:orig 1097
F70 4575718719 baz server/workspace-ops.ts:1181
F71 4575718725 baz server/workspace-ops.ts:1192
F72 4575765428 baz server/workspace-ops.ts:1274
F73 4575718730 baz server/workspace-ops.ts:1281
F74 4576041705 coderabbit server/workspace-ops.ts:1309
F75 4601171052 baz tests/routed-launch.test.ts:93
F76 4575649266 baz tests/routing-example.test.ts:35
F77 4575231978 baz tests/workspace-ops-handlers.test.ts:25
F78 4581338225 coderabbit tests/workspace-ops-handlers.test.ts:orig 54
F79 4575430035 coderabbit tests/workspace-ops-handlers.test.ts:72
F80 4601306663 baz tests/workspace-ops-handlers.test.ts:298
F81 4601171063 baz tests/workspace-ops-handlers.test.ts:358
F82 4575231981 baz tests/workspace-ops-handlers.test.ts:orig 370
F83 4607768683 baz tests/workspace-ops-handlers.test.ts:orig 431
F84 4601964650 baz tests/workspace-ops-handlers.test.ts:648
F85 4603174751 baz tests/workspace-ops-handlers.test.ts:785
F86 4608008336 baz tests/workspace-ops-handlers.test.ts:orig 896
F87 4602268208 baz tests/workspace-ops-handlers.test.ts:931
F88 4600112557 baz tests/workspace-ops.test.ts:193
F89 4574679052 coderabbit tests/workspace-ops.test.ts:orig 194
F90 4600112580 baz tests/workspace-ops.test.ts:336
```

Closing search: run last, 2026-11-20. (1) `https://paseo.sh/docs/plugins/hooks` through the shared
`mcp__firecrawl__firecrawl_scrape`: the live hook reference lists the same 14 hooks as the 0.16.0 SDK types; both
worktree hooks wait; a throwing `worktree.create.before` aborts with `Plugin <id> <hook> failed: …`; a throwing
`worktree.create.after` aborts and removes the worktree; no plugin-to-plugin dependency or import API exists; the page
does not document `checkoutSource` or `idempotencyKey` (those are confirmed from the protocol source and the installed
0.16.0 types only, which is why PR 0 is a spike). (2) ade-claim docs relied on: `plugins/ade-claim/README.md` and
`docs/git-guard.md` at djbclark-ade `77645420`; ade-claim is in observe mode on this machine (no mode file). (3) Already
landed upstream? No: `gh pr list -R omercnet/paseo-plugins --search "pr-radar worktree"` shows only #618 (merged) and
release PRs; the client at `9f1f053` still calls `prepareWorkspaceCwd` then `paseo.workspaces.open`. Not re-checked:
whether a Paseo release newer than 0.16.0 changes `WorkspaceCreateRequest`.
DONE
