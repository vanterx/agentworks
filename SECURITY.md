# Security Policy

## Reporting a vulnerability

Report suspected vulnerabilities privately via GitHub Security Advisories
("Report a vulnerability" on the repo's Security tab), not as public
issues. Include reproduction steps and impact. You should receive an
acknowledgement within 72 hours.

## Threat model — read this before deploying

This workflow runs AI coding agents **unattended** with permissive flags
(`--permission-mode bypassPermissions`, `--dangerously-bypass-approvals-and-sandbox`,
`--yolo`). That is the operating model, not an accident — the safety
properties come from the surrounding controls, all of which you are
responsible for keeping intact:

1. **Issue bodies are attacker input.** Any issue labeled
   `status: available` becomes a prompt to an agent with shell access on
   the runner machine. Gate G0 (a human applying that label) is your
   injection filter — never auto-apply `status: available` to issues from
   untrusted reporters, and never point a runner at a repo whose issue
   queue you don't control. This includes the optional `## Skills`
   section: it lets the issue author put tooling requests into the work
   prompt. It is passed as clearly-framed advisory text and never
   executed or turned into configuration, but G0 review should still read
   it — don't approve an issue whose Skills section you wouldn't say to
   the agent yourself.
2. **PR titles/bodies are attacker input to the reviewer.** The review
   prompt explicitly marks them untrusted; do not weaken that language.
3. **The merge gate is a commit status** (`aw/merge-gate`) plus branch
   protection. If branch protection doesn't require that check, nothing
   stops a compromised or confused agent from merging. Run
   `./scripts/doctor.sh` to verify.
4. **Identity separation** (`REVIEW_GITHUB_TOKEN`) is what makes the
   adversarial review meaningful. Solo mode (`AW_ALLOW_SOLO_REVIEW=1`)
   deliberately weakens this and marks every artifact it touches; treat it
   as a development convenience, never a production posture.
5. **`scripts/`, `.github/workflows/`, `prompts/`, `aw.conf`,
   `.github/trusted-reviewers.json`, `.github/autonomy.json`, and
   `GOALS.md` are governance surfaces.** The review prompt routes
   changes to them to NEEDS_WORK by default, CI validates them, and
   CODEOWNERS should force human review. Keep all three layers.

## Auto-triage

Enabling `auto_triage` in `.github/autonomy.json` weakens gate G0: issue
text reaches agents without a human reading it first. The compensating
controls are the `trusted_authors` allowlist (the zero-token CI tier
only fires for logins you chose), the `max_auto_available_per_day`
budget cap, and — for the agent tier — a fail-closed triage verdict that
treats prompt-injection attempts as grounds for REJECT and never
auto-accepts issues targeting governance surfaces. Even so: **never
enable `agent_triage` on a public repo with open issue creation** unless
you accept that any GitHub user can put text in front of your triage
model. The adversarial review gate still stands between any triaged
issue and a merge. See docs/AUTONOMY.md for the full trade-off ladder.

## Secrets handling

- `REVIEW_GITHUB_TOKEN` is a locally-held credential — never commit it,
  never store it as plaintext in `aw.conf` (use `aw.conf.local`, which is
  gitignored, or your secret manager / environment).
- CI workflows run on the ambient `GITHUB_TOKEN` only; no model API keys
  or agent credentials belong in repository secrets.
- Rotate `REVIEW_GITHUB_TOKEN` on any suspected exposure and audit recent
  merges approved by that identity (`.aw/audit.jsonl` records every
  merge-gate write with a timestamp).

## Supported versions

Only the latest release on the default branch is supported with security
fixes.
