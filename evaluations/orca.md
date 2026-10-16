# Evaluation: Orca

**Repo:** [stablyai/orca](https://github.com/stablyai/orca)
**Stars:** 37,590 | **Last updated:** 2026-08-05 (pushed) | **License:** MIT | **Platforms:** desktop + iOS/Android
**Last verified:** 2026-10-16
**Last triaged:** 2026-10-16  <!-- triaged: bulk -->
**Dev loop stage:** Agent Orchestration (an "ADE" — agent development environment)
**Layer:** Harness / platform (GUI app that hosts many coding agents)

---

## What it does

Orca is an **ADE (Agent Development Environment) for working with a fleet of parallel agents** — "run any coding agent with your own subscription," on desktop and mobile. It hosts **Codex, Claude Code, OpenCode, or Pi side-by-side**, each in its own git worktree, tracked in one place. Standout features:

- **Parallel worktrees** — fan one prompt across five agents, each in an isolated git worktree, then compare results and merge the winner.
- **Mobile companion** — monitor and steer agents from your phone; get notified when one finishes and send follow-ups from anywhere.
- **Terminal splits** — Ghostty-class WebGL terminals with infinite splits and scrollback that survives restarts.
- **Design Mode** — click any element in a real Chromium window to send its HTML/CSS + a cropped screenshot straight into the agent's prompt.
- **Native GitHub & Linear** — browse PRs, issues, and boards in-app and open a worktree from any task.

## How we tested it

**Evidence:** RUN

**Hands-on** — cloned `stablyai/orca` v1.4.214 (`--depth 1`), installed with `pnpm install --ignore-scripts` under `/tmp/eval-orca/`, built the CLI and the Electron bundle (`pnpm run build:cli`, `pnpm run build:electron-vite`), rebuilt the `node-pty` native module via the repo's own `ensure-native-runtime.mjs --runtime=electron`, then started the headless runtime (`orca serve`, Electron under `xvfb-run`) and drove it from a second shell with `orca --pairing-code <orca://pair…>`. Fixture: a 3-file git repo plus a planted defect (`calc.py: add() returns a - b`, 2 unittest cases, 1 failing). Three scripted "candidate" patches stand in for agents (correct fix, wrong fix, no-op); the oracle is `python3 -m unittest`.

```text
$ orca status            # no server
runtimeState: not_running
$ orca --pairing-code … status
runtimeState: ready   runtimeReachable: true   graphState: ready
$ orca repo add --path /tmp/eval-orca/fx/repo      -> ok
# treatment: 5x orca worktree create --repo path:… --name orca-N
orca_total_ms=2772      # all rc=0; git worktree list shows 5 new worktrees on branches t/orca-N
# baseline: 5x git worktree add -b raw-N
git_total_ms=61
# fan-out: worktree create fan-1..3, terminal create (one per worktree), terminal send, terminal read
fan-1 (correct fix): terminal tail "OK"                    oracle=OK
fan-2 (wrong fix):   terminal tail "FAILED (failures=1)"   oracle=FAILED (failures=1)
fan-3 (no-op):       terminal tail "FAILED (failures=1)"   oracle=FAILED (failures=1)
# merge the winner outside Orca: git merge t/fan-1 -> master unittest: OK
# master's calc.py stayed `a - b` until the merge (isolation held)
$ orca worktree rm --worktree name:fan-2 --force   -> {"removed": true}
```

Reproduce: build as above, start `orca serve --port N`, then run the scripted commands in `/tmp/eval-orca/run.sh` and `fan.sh` (not committed; the commands above are the whole procedure).

## Test design

- **Task/corpus:** 5 worktree creations (timing) + a 3-way fan-out over one planted-defect repo, each branch given a different scripted patch.
- **Baseline:** raw `git worktree add -b` for the same count of worktrees; for fan-out, the oracle run by hand in each worktree.
- **Metric:** wall-clock for 5 creations; oracle pass/fail per worktree; whether the main checkout stayed unmodified; whether Orca's terminal read-back matches the oracle I ran directly.
- **Reproduce:** commands above; one run (n=1 per arm), so timing is indicative rather than a distribution.

## What worked

- **Headless runtime is real.** A built checkout starts `orca serve`, and a separate CLI process pairs by `orca://pair` code and drives it (repo add/list, worktree create/rm, terminal create/send/read) with `--json` output.
- **Worktree fan-out is correct and isolated.** 3 worktrees, each ran its own patch; oracle results (1 pass, 2 fail) matched what Orca's terminal read-back showed and what I got running the tests directly; the main checkout was untouched until the explicit merge.
- **State persists across a runtime restart.** After killing and restarting the server, previously created worktrees were still listed.
- **Everything is scriptable.** The CLI exposes worktrees, terminals, automations, skills, accounts and references, so an agent can orchestrate it (it also ships bundled skills: orca-cli, orchestration, orca-linear, orca-jira, computer-use).

## What didn't work or surprised us

- **Not tested: launching a real coding agent** (`worktree create --agent codex|claude`), the GUI, Design Mode, the mobile companion, and Linear/GitHub/Jira integrations. They need an installed agent with credentials or a display session; none were available. Every claim in the README about them stays unverified.
- **Setup is heavy for a server-only use.** `pnpm install --frozen-lockfile` failed with HTTP 403 on a GitHub-tarball dev dependency (`oxlint-plugin-anti-slop`) behind this sandbox's proxy; the run proceeded only after removing that dev dependency from the scratch copy's `package.json`/workspace patch list. `--ignore-scripts` also left `node-pty` unbuilt, so the first terminal call failed with `Cannot find module './prebuilds/linux-x64//pty.node'` until the native rebuild ran. The Electron binary was needed even for `serve`, which first printed `this Orca install carries no orcad template` and then fell back to a serve host.
- **Worktree creation is ~45x slower than raw git** (2772 ms vs 61 ms for 5; ~554 ms vs ~12 ms each, n=1). Irrelevant next to agent run times, but it is overhead for scripts.
- **Status is per-host.** A plain `orca status` reported `not_running` while the paired server was `ready`, so the local CLI only sees a server it is explicitly paired with.
- **Worktrees land outside the repo** (`/root/orca/workspaces/<repo>/<name>`, branches `t/<name>`), and `git worktree list` shows them, so cleanup must go through `orca worktree rm` or git.
- My scripted `EXIT_$?` marker reported `tail`'s exit code, not the test's; the oracle result is the line above it.

## Quality signals affected

| Signal | Impact | Evidence |
|--------|--------|----------|
| Correctness | + (narrow) | Isolated worktrees plus an oracle picked the one passing candidate of three, and the merge produced a passing tree. Agent output quality itself was not measured. |
| Speed | neutral | ~554 ms per worktree vs ~12 ms raw git; the value is parallel agent runs, which were not exercised. |
| Maintainability | neutral | Branches/worktrees outside the repo tree and a 1.4.x fast-moving app; adopting it is a workflow commitment. |
| Safety | neutral / − | Large Electron app, pairing server on `ws://0.0.0.0:<port>` by default (bound to all interfaces in this run), mobile and cloud surfaces widen trust. Isolation by worktree itself held. |
| Cost Efficiency | neutral / − | MIT and bring-your-own subscription, but fanning one prompt across N agents multiplies token spend. |
| Verifiability | + | JSON CLI plus terminal read-back lets a human or script confirm each worktree's result against an independent oracle. |

## Verdict

**discovery-log — tentative read** — The headless runtime, worktree fan-out, terminal control and merge-the-winner flow all work as described on a disclosed fixture (RUN, no real agents), but the agent-launch, GUI, Design Mode and mobile features that are Orca's actual pitch were not exercised, and setup required workarounds. Nothing here supports a verdict either way; a human with an agent subscription and a display should pilot `worktree create --agent` on a real multi-agent task before this moves.

Compared to neighbors: **claude-squad** (TUI, in STACK) covers parallel sessions with a lighter footprint; **worktrunk** covers worktree management alone. Orca's differentiator remains the all-in-one ADE plus scriptable CLI.

## Triage note

Re-examined 2026-10-16 by the P0 measure band (bulk lane): promoted from REVIEW to RUN evidence; eliminate-only authority means the verdict stays at `discovery-log`. What remains for a human: the agent-launch path and the GUI/mobile features listed above.

## Catalog entry

| Name | Type | One-liner | Problem it solves | Overlaps with |
|------|------|-----------|-------------------|---------------|
| [orca](https://github.com/stablyai/orca) | platform | Agent development environment (MIT) for a fleet of parallel agents — run Codex/Claude Code/OpenCode/Pi side-by-side, each in its own git worktree; fan one prompt across agents and merge the winner; WebGL terminals, click-to-context Design Mode, native GitHub/Linear, desktop + mobile companion | Running many coding agents in parallel is fragmented — no single place to launch, isolate, compare, steer (incl. from phone), and merge their work | claude-squad, worktrunk, dmux, claude-fleet, happy |
