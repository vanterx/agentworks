# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **template** for orchestrating autonomous AI coding agents against a GitHub repo. It is deliberately bash-only with zero dependencies beyond `git`, `gh`, and `jq` — no Python, no Node runtime deps, no frameworks. That constraint is the product's niche; do not introduce new runtime dependencies (this is why the test harness is pure bash, not bats, and templating is bash substitution, not Jinja2).

There is no build step. The "app" is a set of bash scripts plus markdown/YAML configuration.

## Commands

```bash
./tests/run.sh                      # run the full test harness (pure bash, no jq/gh/network needed)
bash -n scripts/*.sh scripts/lib/common.sh   # syntax check (CI also runs shellcheck --severity=warning)
./scripts/doctor.sh                 # read-only deployment health check
AW_DRY_RUN=1 ./scripts/start_work.sh         # any loop, without touching GitHub or invoking agents
./scripts/render_prompt.sh work|rework|review <n>   # preview the exact prompt a loop would send
./scripts/metrics.sh                # audit-trail stats + live queue depths
```

There is no per-test selection — `tests/run.sh` is one fast script; run it whole. CI (`.github/workflows/ci.yml`) runs shellcheck, `bash -n`, the harness, YAML parsing, and a trust-config schema check. shellcheck is typically not installed on the local Windows dev machine (nor is `jq` under Git Bash) — the harness is written to run without jq; rely on CI for shellcheck.

## Architecture

The system is four layers, and the separation between them is the core design invariant:

```
GitHub Issues + Labels   →  scripts/*.sh          →  agent CLI            →  git worktree
(the state machine)         (the ONLY thing that     (does the work,         (throwaway sandbox;
                             changes state)           never touches state)    user's clone never dirtied)
```

**The scripts own every label change and merge decision; agents only produce code/reviews.** Every prompt explicitly forbids the agent from editing labels/assignees. If you add a code path that changes issue/PR state, it must go through `set_status_label()` and emit an `audit_event()` — never ad-hoc `gh issue edit --add-label`.

- `scripts/lib/common.sh` — everything shared: config layering, logging, audit trail, `run_agent()` dispatcher (claude/codex/hermes), worktree isolation, the label state machine, claim-race resolution, PR↔issue linkage, prompt builders, trust model. Every entry-point script sources it first.
- `scripts/start_work.sh` — worker loop. Queue priority: my `changes-requested` rework → TTL-freed rework → fresh `available`. Claims via optimistic assign + jittered settle + alphabetically-smallest-login tiebreak (no lock service).
- `scripts/review_work.sh` — adversarial review loop. Reviewer identity must differ from PR author (`REVIEW_GITHUB_TOKEN` swaps `GH_TOKEN`); solo self-review is an explicit opt-in (`AW_ALLOW_SOLO_REVIEW=1`) that stamps every artifact. Posts a plain commit status (`aw/merge-gate`) as the real merge gate. Supports N-reviewer quorum (gate stays `pending` until quorum).
- `scripts/merge_ready.sh` — merges based on `.github/trusted-reviewers.json`, NOT GitHub's native review count (which only counts write-access reviewers). Its latest-review-per-login counting rule is mirrored by `count_trusted_approvals()` in common.sh — the two must never diverge.
- `scripts/reap.sh` — cron GC for stale claims/reworks; no model calls; runs in CI on the ambient `GITHUB_TOKEN`.
- `scripts/triage_work.sh` / `scripts/plan_work.sh` — the optional autonomy loops (agent triage of new issues; backlog generation from `GOALS.md` when the queue is dry). Both are gated by `.github/autonomy.json` — the owner-controlled autonomy switchboard (everything off by default, fail-safe on missing/malformed file via `autonomy_setting()`). `docs/AUTONOMY.md` is the reference.
- `prompts/*.md` — all agent prompts, as `{{var}}` template files. Behavior tuning happens here, not in scripts. Rendering (`render_template()`) is two-phase via random sentinels so untrusted values containing `{{...}}` can never be expanded — keep it that way.
- `.aw/` (gitignored) — runtime state: append-only `audit.jsonl`, heartbeat files.

### Key mechanics that span files (easy to break if you don't know them)

- **Closed-set label sweep**: `set_status_label()` always removes every other status in `ALL_STATUSES`, guaranteeing exactly one `status:` label by construction. `issue-status.yml` sources common.sh and reuses it — one implementation, two call sites.
- **Verdict contract**: the review agent must print `VERDICT: PASS|NEEDS_WORK` as the *last non-empty line*; `last_verdict_line()` enforces this strictly and fails closed (missing/ambiguous = NEEDS_WORK). The review file must stay a randomized `mktemp` path *outside* the worktree — the worktree is an untrusted PR checkout that could commit a forged review file.
- **Non-defect stop conditions**: SIGINT/SIGTERM stops the whole runner; rate-limit signals (`was_usage_limited`) and output-cap hits (`output_was_truncated`) release the item quietly and are *not* recorded as work failures. Preserve this three-way distinction when touching the loops.
- **Config precedence**: env > `aw.conf.local` (gitignored, may hold tokens) > `aw.conf` (committed, never secrets) > defaults in common.sh. The loader allow-lists `AW_*` and `REVIEW_GITHUB_TOKEN` keys only and never evals values.
- **Untrusted input boundaries**: issue bodies (incl. the `## Skills` section) and PR titles/bodies are attacker input. G0 (a human applying `status: available`) is the injection filter; review prompts frame PR text as untrusted. See SECURITY.md before touching anything on this boundary.

## Conventions for changes

- `scripts/`, `.github/workflows/`, `prompts/`, `aw.conf`, `.github/trusted-reviewers.json`, `.github/autonomy.json`, and `GOALS.md` are **governance surfaces** — the review prompt routes unframed changes to them to NEEDS_WORK, and CONTRIBUTING.md requires framing such PRs as proposals.
- New env vars: `AW_` prefix, default in `scripts/lib/common.sh`, a row in the table in `docs/OPERATIONS.md`, and a commented line in `aw.conf`.
- New template placeholders: also add them to `TEMPLATE_VARS` in `scripts/doctor.sh` (it warns on unknown placeholders).
- Mutating `gh` calls must be idempotent, `gh_retry`-wrapped, or explicitly best-effort (`|| true`).
- Shell scripts are LF-only (enforced by `.gitattributes`); avoid bash-4+-only syntax like `${var^^}` — macOS ships bash 3.2. Note the bash 5.2 gotcha already handled in `render_template()`: replacement strings in `${var//pat/rep}` must be quoted or `&` expands.
- Behavioral changes need a `CHANGELOG.md` entry ([Keep a Changelog](https://keepachangelog.com)); breaking ones also go in `docs/MIGRATION.md`. Version lives in `VERSION` (semver).
- Commit style: conventional commits (`feat:`, `fix:`, `docs:`, ...).

## Doc map

`AGENTS.md` = the contract given to agents working *in* an adopting repo. `docs/AUTOMATION.md` = status lifecycle + why merge_ready.sh exists. `docs/OPERATIONS.md` = runbook + full env-var reference. `docs/GETTING_STARTED.md` / `docs/ADOPTION_PROMPT.md` = adopter setup (manual / agent-orchestrated). `docs/SETUP.md` = end-to-end walkthrough against `example/`. `example/` is a throwaway validation target adopters delete.
