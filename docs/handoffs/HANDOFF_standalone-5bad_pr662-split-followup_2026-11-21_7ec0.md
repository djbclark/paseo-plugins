---
schema_version: 1
handoff_id: 7ec0
parent_handoff_ids: []
lineage: none
chain: [standalone-5bad]
repo: omercnet/paseo-plugins (fork checkout ~/src/paseo-plugins-omercnet)
workspace: paseo-plugins-omercnet
branch: feat/pr-radar-launch-and-dprs
head_sha: a06d92e
created_at: 2026-11-21T12:10:08-05:00
writer: claude-code
---
# Handoff — omercnet PR #662 split, #672 rebase, #676 fallbacks, #675 follow-ups

## The Goal
Land omercnet's scope change: #662 is the routing-label feature alone; fallback rules and the settings save queue move to their own PR, where a fallback never drops a trust-downgrade label (e.g. `ade.kind=external-review`); #672 (routing-script runner) rebases on the trimmed #662. Plus a follow-up PR for omercnet's notes on merged #669/#670.

## Where We Are
All four PRs open, MERGEABLE, CI green (pr-radar job, CI sentinel, Baz) as of 2026-11-21 ~17:05Z, waiting on omercnet:
- [#662](https://github.com/omercnet/paseo-plugins/pull/662) head 66d41ca (fork-omercnet `feat/pr-radar-routing-labels`). Appended commits only (no rewrite): e1bdd0c removal of fallbacks/save queue; 2aa3a6c prefix separator; fafdc2a bad-prefix error; 14c3704 rejection-notice count; e95858a preview 128; af4791d collision reason; 6410915 settings v2 migrate drops retired `routingLabelFallbacks` (omercnet 14:13Z blocker); 66d41ca prefix warning cleared on reset (Baz thread). Reply posted: issuecomment-7065876262.
- [#672](https://github.com/omercnet/paseo-plugins/pull/672) head 50f326e (`feat/pr-radar-routing-trust`), rebased onto 66d41ca, force-pushed with lease. Notes fixed: 7d349c5 first-call timeout trim, c0d0d77 + 50f326e kill script on plugin stop, 77106d0 hung-preflight test, f181964 1 MiB stdin cap (refuse). 486 tests. Reply posted: issuecomment-7065761772.
- [#676](https://github.com/omercnet/paseo-plugins/pull/676) head 8bd9dcf (`feat/pr-radar-label-fallbacks`), stacked on #662. Fallback blocked if any returned key is admitted by an allowed prefix. Settings v3 (one-way: going back to #662/main needs a settings Reset; documented). 514 tests.
- [#675](https://github.com/omercnet/paseo-plugins/pull/675) head cb8947f (`fix/pr-radar-669-670-followups`, off main): running promise on second click, unborn-HEAD hint, `--` in worktree remove, all matched roles kept. 329 tests (120 s timeout under load).
Worktrees (all clean, pushed): ~/src/paseo-plugins-followup-wt (#676), ~/src/paseo-plugins-move2-wt (#672), ~/src/paseo-plugins-669fu-wt (#675). Main checkout dirty `.gitignore`, `.ignore`, `.local/` predate this session: not ours.
Demo daemon (isolated) running http://100.113.53.87:54904/ on the #676 branch; routing script moved to ~/.local/state/pr-radar-demo/route.py and `plugin-settings/pr-radar.json` repointed (whether the running daemon re-reads it is NOT verified). Seeded acme/api #41 (fallback applies) and #42 (refused: cannot drop ade.kind).

## What We Tried
- First rebase target e95858a, then af4791d, then 66d41ca: #662 kept moving as fixes landed; W2 held its push (MSG 3) until the final head.
- W2 pushed 59e7897 early: brief said "commit with git-commit-own" (pushes by default) and "do not push". Fast-forward, harmless.
- Workers were closed by a session restart mid-slice; resumed via paseo send (not relaunched).

## Key Decisions
Trim #662 by appending a removal commit rather than rewriting history, so reviewers keep their anchors. The downgrade rule blocks rather than keeping a partial label set, so labels stay all or nothing. The stdin cap refuses oversize payloads rather than truncating, because a truncated record looks complete to the script. The stop path uses the plugin cleanup hook, which Paseo awaits on stop and reload.

## Decisions
- [decision] #662 trimmed by appended removal commit; rejected: history rewrite; because review anchors and stacked #672 stay valid.
- [decision] fallback blocked when any key is prefix-admitted; rejected: keep only the downgrade label; because it keeps labels all or nothing; in [PR 676](https://github.com/omercnet/paseo-plugins/pull/676).
- [decision] stdin over 1 MiB refused; rejected: truncate; because a cut record looks complete; in [PR 672](https://github.com/omercnet/paseo-plugins/pull/672).
- [decision] settings v2 migrate drops retired keys; rejected: loosen strict schema; because strict parse still guards input; in [PR 662](https://github.com/omercnet/paseo-plugins/pull/662).
- [ruling] "Keep, move script" for the demo daemon; "Send correction"; "Add to fleet-watch"; "Handoff and continue" (djbclark, 2026-11-21)

## Relations
- decided_in [[HANDOFF_standalone-5bad_pr662-split-followup_2026-11-21_7ec0]]

## Evidence & Data
- Briefs and worker reports: /private/tmp/claude-502/-Users-djbclark-src-paseo-plugins-omercnet/4d85bfd2-9676-4410-a5f9-8607aa5a5462/scratchpad/w/ (w1/w2/w3-brief.md, -report.txt).
- Adversary reports: ~/.local/state/agent-reports/19e17f44-32fa-4afd-a521-c5b7811482aa/ (662 trim, 662 migration fix: diff sha256 5f1f4962... matches `git diff af4791d 66d41ca`; fallback PR; merge delta).
- Astra profile unavailable: Opus 5.5 high stood in for #675/#676 (said in PR bodies).

## Operator Feedback
Picked all recommended options in /loose. Wants reviews followed by script, not model polling.

## Where We're Going
1. Extend ~/ops/site-private-djbclark/bin/fleet-watch so it watches omercnet/paseo-plugins #662 #672 #675 #676 for new comments, reviews, CI failures and merges, and hermes-pings on each event (script, not a recurring model prompt). Ask which repo only if fleet-watch's own repo is unclear.
2. On a new omercnet round: re-task a worker using the matching brief in the scratchpad above (route_agent.py plan; `git-commit-own --no-push` when pushes are gated; cite /Users/djbclark/src/paseo-plugins-omercnet/pr-radar/AGENTS.md by absolute path).
3. Merge order once approved: #662, then rebase #672 and #676 on main; #675 independent. Never `--admin`.
4. Stop the demo daemon only when djbclark says he is done: ~/src/paseo-plugins-followup-wt/.agents/skills/deliver/scripts/dev-daemon.sh down pr-radar

## Detached jobs
none (all workers archived; demo daemon is a deliberate long-lived process, not a job record).

## Quick Start
```sh
for n in 662 672 675 676; do gh pr view $n -R omercnet/paseo-plugins --json state,mergeable,reviewDecision; done
~/src/paseo-plugins-followup-wt/.agents/skills/deliver/scripts/dev-daemon.sh cli pr-radar plugin ls
```
