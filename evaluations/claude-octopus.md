# Evaluation: claude-octopus

**Repo:** [nyldn/claude-octopus](https://github.com/nyldn/claude-octopus)
**Stars:** 3,926 | **Last updated:** 2026-08-05 | **License:** MIT
**Last verified:** 2026-10-16
**Last triaged:** 2026-10-16  <!-- triaged: bulk -->
**Dev loop stage:** Plan + Review (primary) — also spans Implement/Verify/Ship via lifecycle commands
**Layer:** Tooling

---

## What it does

Catalog one-liner: "Run up to 8 AI models in parallel on research, design, or code to surface blindspots before shipping." The repo as of v9.45 actually advertises *nine* providers — Codex (GPT-5.4), Gemini, Antigravity CLI (`agy`), GitHub Copilot, Qwen, Ollama, Perplexity, OpenRouter, and OpenCode — orchestrated alongside Claude, which is the only required one. The pitch is blindspot coverage: every model has gaps, so put several on the same task and gate on agreement before shipping.

Mechanically it is a native Claude Code **plugin** installed from the `nyldn-plugins` marketplace (`claude plugin marketplace add` → `claude plugin install octo@nyldn-plugins` → `/octo:setup`). It ships 49 slash commands (`/octo:*`), 32 personas (agent definitions), and 54 skills, plus lifecycle hooks (session start/end, prompt submit, tool use, compaction, plan mode, worktrees, etc.) declared in `.claude-plugin/hooks.json`. The differentiated multi-model surface is not the four-phase lifecycle commands (`/octo:embrace` = Discover→Define→Develop→Deliver) but the *cross-checking* commands: `/octo:debate` (structured provider debate with consensus), `/octo:council` (3/5/7-persona deliberation with goal modes advice/decision/plan/implement/review, adversarial/red-team styles, quorum + critical-veto gates, budget caps), `/octo:review` (multi-LLM code review with inline PR comments), and `/octo:research` (attributed multi-provider fan-out).

The consensus is not theater-by-prompt. `council.md` mandates that the command shell out to `"$CLAUDE_PLUGIN_ROOT/scripts/orchestrate.sh" council $ARGUMENTS` — the real runner that dispatches to actual provider CLIs/APIs — and explicitly **prohibits** the model from role-playing multiple personas inside one Claude context unless the user passes `--simulate`/`--single-model` (in which case the output must be labeled "single-model simulation"). A documented **75% consensus gate** flags disagreement before work proceeds; an "agent status ledger" (`octopus agent-summary`) reports which providers actually contributed, ran degraded, or failed. Providers authenticate via existing subscriptions/OAuth (Codex via ChatGPT sub, Gemini via Google account, Copilot via GH sub) or per-token API keys; Ollama is local/free.

## How we tested it

**Evidence:** RUN

**Hands-on** — cloned `nyldn/claude-octopus` at `04b24c6` (plugin manifest v11.14.0, 2026-10-15) into a scratch directory in a cloud sandbox with `HOME` redirected, then ran the plugin's own smoke and unit suites, `claude plugin validate`, and the provider-detection and council entry points. The sandbox holds Claude Code only (no Codex, Gemini, Copilot, Ollama or other provider CLI, and no model credentials), so every number below describes the Claude-only path.

```bash
git clone --depth 1 https://github.com/nyldn/claude-octopus && cd claude-octopus
claude plugin validate .                    # Validation passed with warnings (56 lines: 42 unquoted ${CLAUDE_PLUGIN_ROOT} hook commands, 14 other)
./tests/run-all.sh smoke                    # Total 16 / Passed 16 / Failed 0, 1m27s
./tests/run-all.sh unit                     # stopped after ~13 min: 50 PASS, 2 FAIL of 408 suites reached
HOME=$scratch ./scripts/orchestrate.sh detect-providers   # 5.9s; CLAUDE_STATUS=ok, all 22 other providers not-installed (CLAUDE_SDK/ANTHROPIC_API unauthenticated)
HOME=$scratch ./scripts/orchestrate.sh council "Should we use a global mutable cache in a request handler?"   # killed by `timeout 120` and again by `timeout 100`, output ends at "Project policy: no project policy source found"
HOME=$scratch ./scripts/orchestrate.sh --dry-run debate "tabs vs spaces"   # ERROR: AI Debate Hub not found, run git submodule update --init --recursive
find commands -name '*.md' | wc -l          # 54
find skills -name SKILL.md | wc -l          # 63
ls agents/personas | wc -l                  # 31
```

## Test design

- **Task/corpus:** the repository's own test suites at `04b24c6` (16 smoke, 408 unit) plus four entry-point invocations on a machine with Claude as the only provider. The planned cross-model comparison (council or review on a planted-defect file against a single-model baseline) needs at least one non-Claude provider with credentials, which this sandbox lacks, so no defect-catch rate is reported.
- **Baseline:** none. This is a single-arm run of the plugin's own checks and entry points; it measures whether the install is sound, not whether consensus beats one model.
- **Metric:** suites passed/failed, wall-clock seconds, exit behaviour of each command.
- **Reproduce:** the commands in the block above; `apt-get install rsync gawk` was done in the sandbox before the council retry (mawk, the default awk, printed `REcompile() - panic` on a `{0,3}` regex in the first council run).

### Test design — skills (required when Type is skill or plugin)

- **Triggering:** not measured. Skill triggering needs a live model session, and the sandbox has none. The 63 skills and 54 commands register (smoke `test-packaging-integrity.sh` and `test-help-commands.sh` pass), which shows they load, not that they fire on the right prompts.
- **Output A/B:** not measured, for the same reason; the with/without comparison needs a model on both arms.

## What worked

- **Structurally sound package.** `claude plugin validate .` passes, and all 16 smoke suites pass (packaging integrity, syntax of every script, safety hooks, dry-run of all commands, council and costs command contracts) in 87 seconds.
- **Provider detection is fast and honest.** `detect-providers` finished in 5.9 s and reported 23 providers with a status each; on this machine Claude was `ok` and the other 22 `not-installed`, with no false positives and the SDK and API routes correctly marked `unauthenticated`.
- **Large hermetic test investment.** 408 unit suites plus smoke, integration and live tiers; the live tier is separated so `make test` never spends money. Of the 52 unit suites that completed before the run was stopped, 50 passed.
- **Surface matches the earlier review.** 54 commands, 63 skills and 31 personas on disk, consistent with the manifest and the catalog description; the plugin is now at v11.14.0, well past the v9.45 the earlier review read.

## What didn't work or surprised us

- **The headline value was not exercised.** No second provider and no credentials were available, so the 75% consensus gate, the council vote and cross-model review were never run against real model output. Whether cross-model disagreement catches defects a single strong reviewer misses remains unmeasured here and, as far as the repo's docs show, unmeasured upstream.
- **Council hangs with Claude as the only provider.** `orchestrate.sh council` produced no result in 120 s or in a second 100 s attempt (after installing gawk) and was killed by `timeout`; the last line was the project-policy notice. The cause was not isolated: the dispatch may be waiting on the unauthenticated Claude route, so this is a Claude-only, no-credentials observation and not proof of a hang with a configured account.
- **Two unit suites fail in this sandbox.** `test-agy-provider.sh` (a stalled catalog lookup returned in 1159 ms but left the process alive) and `test-bare-auth-probe-timeout.sh` (a grandchild process survived the probe's kill, and an interrupted probe returned 143 with the process still running). Both concern process-tree cleanup, which may be a property of this container; they are reported as observed failures, not diagnosed.
- **Run time.** The unit tier covered 52 of 408 suites in about 13 minutes, so a full local pass is a multi-hour job in this environment.
- **Packaging warnings.** `claude plugin validate` raised 42 warnings that hook commands use `${CLAUDE_PLUGIN_ROOT}` unquoted, which breaks if the install path has a space.
- **`debate` needs a submodule.** `--dry-run debate` stops with "AI Debate Hub not found" and asks for `git submodule update --init --recursive`, so the command is not usable from a plain marketplace install in this form.
- **Footprint.** 54 commands, 63 skills, 31 personas and a broad hook set, from what is effectively one maintainer's fast release cadence; the earlier review's maintenance and cost concerns stand.

## Quality signals affected

| Signal | Impact | Evidence |
|--------|--------|----------|
| Correctness | neutral (unmeasured) | Cross-model catch rate was not measurable without a second provider; the plumbing passes 16/16 smoke suites and 50 of 52 completed unit suites |
| Speed | - | Council produced nothing in 100-120 s on the Claude-only path, and the full unit tier is a multi-hour run; provider detection itself is fast (5.9 s) |
| Maintainability | +/- | Heavy hermetic test coverage, but 54 commands, 63 skills and 31 personas is a large surface with overlapping entry points |
| Safety | +/- | Safety-hook smoke tests pass, but the plugin installs a broad hook set and routes prompts to external providers; 42 unquoted-path warnings |
| Cost Efficiency | - | Design fans one task out to several paid providers; the cost controls (`--max-cost`, preflight) were read in the earlier review and not exercised here |
| Verifiability | +/- | The agent-status ledger and test tiers make provider participation checkable in principle; none of it was observable here with one provider, and the consensus output is itself model-generated |

## Verdict

**discovery-log — tentative read** — the package installs cleanly and passes its own smoke tier (16/16, 50 of 52 completed unit suites), but the cross-model consensus value that justifies it needs a second provider and credentials, and the Claude-only council path produced no result in the sandbox; a real read needs a machine with at least two configured providers and a planted-defect review comparison against a single-model baseline.

## Triage note

Left at `discovery-log`, not SKIPped. It was banded against
[`code-review`](https://github.com/anthropics/claude-plugins-official) (STACK), but the two answer
different questions: `code-review` runs four review agents inside one model's head, while octopus
dispatches *genuinely independent providers* (Codex/Gemini/Copilot/Qwen/Ollama/…) and gates on a
75% consensus. Cross-model disagreement is information a single-model reviewer cannot produce by
construction, so this is not the redundancy this band eliminates.

The open question is whether that information is worth its price — multiple paid subscriptions, a
large fast-moving install footprint, and an essentially single maintainer at ★3.8K. That is a
with/without read on a disclosed set of review tasks (does the council catch defects the four-agent
review misses?), which is P0 work, not a bulk call.

_Triaged 2026-08-04 by the P2 challenger band ([#267](https://github.com/mattbutlerengineering/ai-tooling/issues/267))._

## Catalog entry

| Name | Type | One-liner | Problem it solves | Overlaps with |
|------|------|-----------|-------------------|---------------|
| [claude-octopus](https://github.com/nyldn/claude-octopus) | plugin | Run up to 8 AI models in parallel on research, design, or code to surface blindspots before shipping (3.6K stars) | Single-model review misses blindspots; want cross-model consensus before merge | PR-Agent, code-review-graph |
