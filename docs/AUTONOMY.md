# The Autonomy Ladder

How much of this workflow runs without a human is an **owner decision**,
made declaratively in one committed file: `.github/autonomy.json`. Every
rung below is off by default; you climb by flipping switches, and
`./scripts/doctor.sh` always tells you which rung you're standing on.

```json
{
  "auto_triage": { "enabled": false, "trusted_authors": [], "agent_triage": false, "max_auto_available_per_day": 20 },
  "auto_resume": { "ci_failures": false, "merge_conflicts": false },
  "dependency_gating": true,
  "planner": { "enabled": false, "goals_file": "GOALS.md", "max_issues_per_run": 3 }
}
```

The file is a governance surface: the adversarial reviewer flags PRs
that modify it, CODEOWNERS should route it to maintainers, and a
missing/malformed file fails safe to everything-off.

## The rungs

| Level | What runs alone | What humans still do | Enable via |
|---|---|---|---|
| **L0 — manual** | Nothing; scripts run ad hoc | Everything | (don't deploy loops) |
| **L1 — loops** | Claim → work → PR → adversarial review → merge → rework → GC | Write issues, label each one `status: available` (G0), keep runners alive (or use `deploy/`) | Run `start_work.sh` + `review_work.sh`; `reap.yml` is already cron |
| **L2 — auto-triage** | + new issues enter the queue by themselves | Write issues; review DEFERred ones | `auto_triage.enabled=true` + `trusted_authors` (zero-token CI tier); optionally `agent_triage=true` + `triage_work.sh` for everyone else |
| **L3 — auto-resume** | + CI failures and merge conflicts route themselves back to rework; issues wait for their `Depends-on:` dependencies | Write issues | `auto_resume.ci_failures=true`, `auto_resume.merge_conflicts=true` (`dependency_gating` is already on) |
| **L4 — self-driving backlog** | + when the queue runs dry, the planner proposes new issues from `GOALS.md` | Maintain `GOALS.md`, trust config, and incident response — that's it | `planner.enabled=true` + a real `GOALS.md` + `plan_work.sh` on a timer |

## What each rung costs you (read before climbing)

**L2 weakens gate G0.** G0 (a human labeling each issue) is the
injection filter — the moment labeling is automatic, issue text flows to
agents with shell access without a human reading it first. Compensating
controls: `trusted_authors` (deterministic tier only fires for logins
you chose), the daily cap (`max_auto_available_per_day` — the budget
valve G0 used to provide), the agent tier's fail-closed verdict and its
refusal to auto-accept issues touching governance surfaces, and the
adversarial review still gating every merge. Do not enable
`agent_triage` on a public repo with open issue creation unless you've
read SECURITY.md#auto-triage and accept it.

**L3 spends without asking.** Every auto-resumed rework cycle is a real
agent run. The dedup markers (`aw-ci-fail:<sha>` / `aw-conflict:<sha>`)
guarantee at most one bounce per head SHA, so a permanently broken build
converges to one rework attempt per push, not an infinite loop.

**L4 means the system decides what to build next.** The planner only
runs when the queue is truly dry, only proposes from `GOALS.md`
(refusing to run on the template stub), files at most
`max_issues_per_run` per pass, and cannot propose governance changes.
Its issues still pass through triage and adversarial review — proposing
work and admitting work stay separate decisions. Your leverage is the
goals file: keep objectives tight and non-goals explicit, because at L4
that document IS your steering wheel.

## Keeping the loops alive (all rungs)

- **Self-hosted** (recommended): `deploy/systemd/` units with
  `Restart=on-failure`, or `deploy/docker/compose.yml` with
  `restart: unless-stopped`. Model keys stay on your machines.
- **Cloud mode** (zero infrastructure, opt-in):
  `.github/workflows/agent-runner.yml.disabled` — rename to enable.
  Bounded batches on a 30-minute cron with `ANTHROPIC_API_KEY` as a repo
  secret. Read the header comment: this breaks the "model keys never in
  CI" posture and you accept that trade explicitly by renaming the file.
- Monitor either with heartbeat files + `metrics.sh` per
  [OPERATIONS.md](OPERATIONS.md).

## What can never be automated away

Regardless of rung: the adversarial review gate (a different identity, or
explicit solo mode, on every merge), the `do-not-automate` and
`review: human-only` parking brakes, branch protection on
`aw/merge-gate`, the trust whitelist for merges, and the audit trail
recording every transition. The ladder changes who *feeds* the machine,
never who *checks* it.
