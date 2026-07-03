# Contributing

Thanks for improving this template. Two kinds of contribution flow through
this repo, with different bars:

## 1. Ordinary changes (docs, examples, skills)

Follow the standard loop in [AGENTS.md](AGENTS.md): one issue per PR,
claim before starting, commit ending `(Closes #n)`, expect adversarial
review.

## 2. Engine changes (`scripts/`, `.github/workflows/`)

These alter how work is claimed, reviewed, and merged for everyone —
they are governance changes:

- Open an issue describing the problem **before** writing code, and frame
  the PR explicitly as an orchestration/governance proposal.
- Expect the adversarial reviewer to return NEEDS_WORK on unframed engine
  changes by design.
- CODEOWNERS routes these paths to human maintainers; automation cannot
  self-approve them.

## Development standards

- Bash: `set -uo pipefail`, no unguarded `set -e` in shared code paths;
  every mutating `gh` call must be either idempotent, retried
  (`gh_retry`), or explicitly best-effort (`|| true`) with a logged
  outcome.
- Every state transition must go through `set_status_label()` /
  `audit_event()` — never edit labels ad hoc in a new code path.
- All scripts must pass `bash -n` and `shellcheck --severity=warning`
  (CI enforces both).
- Workflow YAML must parse and keep least-privilege `permissions:` blocks.
- New env vars use the `AW_` prefix, get a default in
  `scripts/lib/common.sh`, and get a row in the reference table in
  [docs/OPERATIONS.md](docs/OPERATIONS.md).

## Commit messages

Conventional commits (`feat:`, `fix:`, `docs:`, `refactor:`, `ci:`,
`chore:`). Update `CHANGELOG.md` under `[Unreleased]` in the same PR when
the change is user-visible.

## Testing your change

1. `bash -n` every touched script.
2. Run `./scripts/doctor.sh` against a scratch repo.
3. Walk the relevant part of [docs/SETUP.md](docs/SETUP.md) end-to-end
   with `AW_DRY_RUN=1` first, then live in the scratch repo.
