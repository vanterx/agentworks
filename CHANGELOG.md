# Changelog

All notable changes to this project are documented here. The format is
based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and
this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [1.1.0] - 2026-07-04

### Security
- **Template injection fixed**: `render_template()` now renders in two
  phases via per-run random sentinels, so a value (e.g. a malicious
  issue body) containing `{{placeholder}}` text can never be expanded,
  regardless of substitution order.
- **Review-file forgery fixed**: the reviewer's output file moved from a
  fixed path inside the PR-head worktree to a randomized `mktemp` path
  outside it. A PR that committed its own `.aw-review.md` with
  `VERDICT: PASS` can no longer be mistaken for reviewer output; crash
  detection is now "file still empty" instead of "file missing".
- **Strict verdict parsing**: the verdict must be the last non-empty
  line of the agent's output (fallback: of the review file); quoted
  examples mid-text can no longer false-match. Still fails closed.
- Review prompt's governance guard widened to `prompts/`, `aw.conf`, and
  `.github/trusted-reviewers.json`; `validate.sh`/`validate.yml` carry
  an explicit never-execute-overlaid-content invariant;
  `AW_CLAUDE_PERMISSION_MODE` typos now warn at preflight.

### Added
- `tests/run.sh` — zero-dependency bash test harness (24 assertions over
  rendering, skills extraction, verdict parsing, config loading, label
  sweep), wired into CI as a `tests` job.
- `scripts/render_prompt.sh` — prints the exact prompt a loop would send
  for a given issue/PR (prompt builders now shared in `common.sh`).
- `scripts/metrics.sh` — audit-trail aggregation (rework rate,
  release-without-PR rate, claim→PR cycle time) plus live queue depths.
- Heartbeat files (`.aw/heartbeat-<script>`) touched every loop
  iteration for external liveness monitoring.
- `AW_AGENT_OUTPUT_LIMIT` (default 10 MB) caps captured agent output;
  hitting the cap is a tooling failure, not a work verdict.
- `doctor.sh`: prompt-template validation (existence, placeholder set,
  balance), AGENTS.md adopter-placeholder warning, runtime detection.
- Docs: `docs/MIGRATION.md`; OPERATIONS troubleshooting matrix, cost
  estimation methodology, heartbeat monitoring; Windows/Git Bash/WSL
  guidance in GETTING_STARTED; example validators in `validate.sh`.

### Changed
- Review queue is processed oldest-first (was random) — the review-claim
  lock already prevents reviewer stampedes.
- `fetch_open_issues()` and `reap.sh` warn when they hit the 100-item
  query caps instead of silently truncating.
- `make_worktree()` re-fetches and retries once when the ref vanished
  between fetch and add (force-push race).

### Rejected (by design, from the same review)
- Issue batching (breaks the one-issue-one-PR invariant), per-run
  correlation IDs (the issue/PR number already correlates), GraphQL
  pagination (warn instead; a 100-deep queue is a triage problem), and
  bats (zero-dependency harness instead).

## [1.0.1] - 2026-07-03

### Added
- External prompt templates: `prompts/work.md`, `prompts/rework.md`,
  `prompts/review.md` with `{{var}}` placeholders, rendered by a
  dependency-free bash `render_template()` (`AW_PROMPTS_DIR` to
  relocate). Prompts are now customized by editing files, not scripts.
- Per-issue skill injection: a `## Skills` section in an issue body is
  passed to the worker agent as a framed, advisory tooling request
  (untrusted input, G0-gated, never executed).
- Multi-reviewer quorum: when `required_approvals` in
  `.github/trusted-reviewers.json` (or `AW_REVIEW_QUORUM`) is above 1,
  the review loop keeps the merge gate `pending` ("Quorum: N/M trusted
  approvals") until enough distinct trusted reviewers approve; the
  reviewer completing the quorum merges. Counting matches
  `merge_ready.sh` exactly.
- TDD enforcement: `AW_ENFORCE_TDD=1` injects tests-first requirements
  into work/rework prompts and a corresponding NEEDS_WORK criterion into
  the review prompt.

## [1.0.0] - 2026-07-03

### Added
- Structured logging with levels, optional JSON output, and optional log
  file (`AW_LOG_LEVEL`, `AW_LOG_FORMAT`, `AW_LOG_FILE`).
- Append-only JSONL audit trail of every state transition, review
  verdict, merge-gate write, and merge (`.aw/audit.jsonl`, configurable
  via `AW_AUDIT_LOG`).
- Config file layer: committed `aw.conf` for team defaults, gitignored
  `aw.conf.local` for operator-local values and secrets; environment
  always wins.
- `gh_retry` exponential-backoff wrapper on the queue fetch and every
  merge-gate commit-status write.
- `scripts/doctor.sh` — read-only deployment health check (binaries,
  auth, labels, trust config, branch protection, observability).
- Full dry-run coverage: `AW_DRY_RUN=1` (or `--dry-run`) makes every
  loop report intended actions without invoking an agent or touching
  GitHub state.
- Optional single-instance lock per script per repo
  (`AW_SINGLE_INSTANCE=1`).
- CI workflow (`ci.yml`): shellcheck, bash syntax, YAML parse, trust
  config schema check, content validator.
- Governance files: CODEOWNERS, SECURITY.md (threat model), 
  CONTRIBUTING.md, PR template.
- `docs/OPERATIONS.md` — production runbook (deployment topologies,
  monitoring, incident response, token rotation, cost controls) with a
  complete environment-variable reference.

### Changed
- Trust config loading now warns and fails safe (empty whitelist) on
  malformed JSON.
- `preflight` warns on `gh` versions older than 2.20.0 and logs the
  resolved identity/agent at debug level.

## [0.1.0] - 2026-07-03

### Added
- Initial template: issue/label state machine, worker loop, adversarial
  review loop with strict and solo identity modes, stale-claim reaper,
  whitelist trust-model merge automation, worktree isolation,
  multi-agent CLI dispatch (claude/codex/hermes), GitHub Actions label
  bookkeeping, onboarding and triage skills, adopter documentation.
