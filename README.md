# AgentWorks

A production-grade, drop-in orchestration layer for running autonomous
AI coding agents against your GitHub repo — with owners deciding exactly
how much runs on its own, from human-triaged queues to a fully
self-driving backlog.

> Agents grind. Humans steer.

This is not a framework or a message queue. It's four ideas, each
implemented as something boring, auditable, and inspectable:

- **GitHub Issues + Labels are the state machine.** An issue's `status:`
  label tells every script exactly what state it's in — nothing else
  needs to track that.
- **Deterministic bash scripts are the state-transition engine.** They own
  every label change, claim, and merge — never the agent.
- **CLI coding agents are interchangeable worker nodes.** `AW_AGENT=claude
  |codex|hermes` — any CLI that accepts a prompt and can use `git`/`gh` is
  a drop-in worker.
- **Git worktrees are the sandbox.** Every task runs in a fresh, throwaway
  worktree. Your real clone is never touched.

Every merge is gated by an adversarial review — by default from a
different GitHub identity than the PR's author — so no single agent run
can merge its own work unchecked.

```
 GitHub Issues + Labels  →  scripts/*.sh  →  agent CLI  →  worktree  →  adversarial review  →  merge
      (state)                (engine)        (worker)      (sandbox)         (gate)
```

## Production features

- **Audit trail** — every claim, status change, review verdict,
  merge-gate write, and merge is recorded as append-only JSONL
  (`.aw/audit.jsonl`), queryable with `jq`.
- **Structured logging** — leveled, timestamped, optional JSON format and
  log file for centralized log shipping (`AW_LOG_LEVEL`, `AW_LOG_FORMAT`,
  `AW_LOG_FILE`).
- **Declarative config** — committed `aw.conf` for team defaults,
  gitignored `aw.conf.local` for operator-local values and secrets;
  environment always wins.
- **Resilience** — exponential-backoff retries on the queue fetch and
  every merge-gate write; rate-limit signals are detected and backed off
  rather than treated as work failures; ambiguous review verdicts fail
  closed.
- **Health checks** — `./scripts/doctor.sh` verifies binaries, auth,
  labels, trust config, and branch protection before you run anything.
- **Customizable prompts** — every agent prompt is a `{{var}}` template
  file in `prompts/`; teams tune agent behavior without touching scripts
  (zero-dependency bash rendering, no Jinja/Python).
- **Per-issue skills** — a `## Skills` section in an issue body passes
  advisory tooling requests to the worker agent (G0-gated, never
  executed).
- **Multi-reviewer quorum** — set `required_approvals` > 1 and run one
  review loop per identity; the merge gate stays pending until N
  distinct trusted reviewers approve.
- **TDD enforcement** — `AW_ENFORCE_TDD=1` requires tests-first in the
  work prompt and makes missing tests a NEEDS_WORK criterion in review.
- **Owner-configurable autonomy** — one committed file
  (`.github/autonomy.json`) decides how much runs alone: auto-triage of
  new issues, auto-resume on CI failures and merge conflicts,
  `Depends-on: #N` gating, and an optional planner that generates the
  backlog from `GOALS.md` when the queue runs dry. All off by default;
  see [docs/AUTONOMY.md](docs/AUTONOMY.md).
- **Always-on deployments** — systemd units and Docker Compose in
  `deploy/`, plus an opt-in GitHub Actions cloud mode.
- **Full dry-run** — `AW_DRY_RUN=1` (or `--dry-run`) on any script
  reports intended actions without invoking an agent or touching GitHub.
- **CI for the engine itself** — shellcheck, syntax, YAML, and trust-config
  schema checks on every PR (`.github/workflows/ci.yml`).
- **Governance** — CODEOWNERS routes engine changes to humans, a
  documented threat model ([SECURITY.md](SECURITY.md)), and an operations
  runbook ([docs/OPERATIONS.md](docs/OPERATIONS.md)).

## Quickstart

**Requirements:** `git`, [`gh`](https://cli.github.com/) (authenticated),
[`jq`](https://jqlang.org/), and at least one agent CLI (`claude`,
`codex`, or `hermes`). Nothing else — no Python, no Node runtime deps.

This is a template. The fastest way to adopt it: paste
[`docs/ADOPTION_PROMPT.md`](docs/ADOPTION_PROMPT.md) into your AI coding
agent inside your own repo and let it orchestrate the whole setup —
files, labels, branch protection, verification. Manual path:
[`docs/GETTING_STARTED.md`](docs/GETTING_STARTED.md). Then use
[`docs/SETUP.md`](docs/SETUP.md) for a concrete dry run using the
`example/` content before you point it at real work.

```bash
gh auth login
./scripts/doctor.sh              # verify the deployment first
./scripts/start_work.sh          # worker loop
./scripts/review_work.sh         # adversarial review loop
./scripts/merge_ready.sh         # trust-model merge evaluation
./scripts/reap.sh                # stale-claim garbage collection
```

## The scripts

| Script | Role |
|---|---|
| `scripts/start_work.sh` | Worker loop: claims `status: available` issues, does the work in a worktree, opens a PR |
| `scripts/review_work.sh` | Adversarial reviewer loop: reviews open PRs, sets the `aw/merge-gate` status check |
| `scripts/reap.sh` | Garbage collector: frees stale claims and reworks (cron-friendly, no model calls) |
| `scripts/merge_ready.sh` | Evaluates a trust-model whitelist against recorded reviews and merges READY PRs |
| `scripts/triage_work.sh` | Agent-triage loop for issues from non-trusted authors (autonomy L2) |
| `scripts/plan_work.sh` | Backlog planner: files new issues from GOALS.md when the queue runs dry (autonomy L4) |
| `scripts/doctor.sh` | Read-only deployment health check (labels, auth, branch protection, prompt templates) |
| `scripts/render_prompt.sh` | Preview the exact prompt a loop would send for an issue/PR |
| `scripts/metrics.sh` | Audit-trail stats (rework rate, cycle time) + live queue depths |
| `scripts/lib/common.sh` | Shared library every script above sources |
| `tests/run.sh` | Zero-dependency test harness (also runs in CI) |

## Docs

- [`AGENTS.md`](AGENTS.md) — the operating contract every agent (or human)
  working in this repo should follow.
- [`docs/AUTOMATION.md`](docs/AUTOMATION.md) — the full status lifecycle,
  the reasoning behind `merge_ready.sh`, and solo-mode review.
- [`docs/AUTONOMY.md`](docs/AUTONOMY.md) — the autonomy ladder: how much
  runs without humans, what each level costs, and the owner-controlled
  toggles.
- [`docs/ADOPTION_PROMPT.md`](docs/ADOPTION_PROMPT.md) — paste-ready
  prompt that has your AI agent orchestrate the setup in your repo.
- [`docs/GETTING_STARTED.md`](docs/GETTING_STARTED.md) — adopting this
  into your own project manually.
- [`docs/OPERATIONS.md`](docs/OPERATIONS.md) — production runbook:
  topologies, monitoring, incident response, cost controls, and the full
  environment-variable reference.
- [`docs/SETUP.md`](docs/SETUP.md) — an end-to-end dry run.
- [`SECURITY.md`](SECURITY.md) — threat model and secrets handling.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — how changes to the engine itself
  are governed.
- [`.claude/skills/`](.claude/skills/) — onboarding and triage skills for
  Claude Code.
