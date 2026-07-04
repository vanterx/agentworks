# Adoption Prompt

Setting up this workflow in your own repo? Copy everything below the
horizontal rule into your AI coding agent (Claude Code, Codex, etc.)
**running inside your target repo**, fill in the target-repo placeholder
at the top, and let it orchestrate the setup. It will ask you the decisions that
are yours to make (review identity, which agent CLIs, content dirs) and
verify the result with `doctor.sh` before calling anything done.

Prefer doing it by hand? The same steps live in
[GETTING_STARTED.md](GETTING_STARTED.md).

---

You are setting up the **AgentWorks** orchestration system in this
repository. It lets autonomous AI coding agents (Claude Code / Codex /
Hermes) claim GitHub issues, do work in isolated git worktrees, open PRs,
and pass adversarial review before anything merges — with deterministic
bash scripts (never the agents) owning every label change and merge
decision.

**Template source:** https://github.com/vanterx/agentworks
**Target repo:** `<OWNER/NAME>`  (this repository)

## Ground rules for this setup

1. Work on a new branch (`setup/agent-workflow`). Never commit directly
   to the default branch.
2. Before overwriting ANY existing file (e.g. an existing `AGENTS.md`,
   `.gitignore`, `SECURITY.md`, workflows), show me the conflict and ask.
   Merge additively where possible (append `.gitignore` entries rather
   than replacing the file).
3. Never write a token or secret into any committed file. If I give you
   a token, it goes in `aw.conf.local` (gitignored) or I set it in my
   environment myself.
4. Anything that changes GitHub repo settings (labels, branch
   protection) — list exactly what you're about to do and get my
   confirmation first.
5. If a step fails, stop and report; don't improvise around the
   orchestration design.

## Phase 0 — Prerequisites, then ask me these questions

First verify the runtime requirements on this machine: `git`, `gh`
(authenticated — check `gh auth status`), and `jq` must be on PATH, plus
at least one agent CLI (`claude`, `codex`, or `hermes`). If anything is
missing, tell me what and how to install it (e.g. `winget install
jqlang.jq`, `apt install jq`, `brew install jq`) and stop until it's
resolved — every orchestration script hard-fails without these.

Then, before touching anything, ask me (one round of questions, don't
drip):

1. **Review identity mode** — do I have (or want to create) a second
   GitHub identity for adversarial review (`REVIEW_GITHUB_TOKEN`), or do
   I start in solo mode (`AW_ALLOW_SOLO_REVIEW=1`, self-review, marked on
   every artifact)?
2. **Which agent CLIs** will run here (claude / codex / hermes), and
   which should be the default `AW_AGENT`?
3. **Trusted reviewers** — which GitHub logins belong in
   `.github/trusted-reviewers.json`, and how many approvals should
   `merge_ready.sh` require?
4. **Content validation** — which directories (if any) should
   `validate.yml` check on PRs? (Or should I drop `validate.yml` +
   `scripts/validate.sh` entirely?)
5. **Keep the walkthrough example?** The template ships `example/NOTES.md`
   + `docs/SETUP.md` for an end-to-end trial run — keep them for now, or
   skip straight to real work?

## Phase 1 — Bring in the template files

From https://github.com/vanterx/agentworks (clone it or fetch the files
raw), copy into this repo (respecting ground rule 2 on conflicts):

- `scripts/` — `lib/common.sh`, `start_work.sh`, `review_work.sh`,
  `reap.sh`, `merge_ready.sh`, `doctor.sh`, `validate.sh` (all must end
  up executable, LF line endings)
- `prompts/` — `work.md`, `rework.md`, `review.md` (the scripts fail
  without them; customize their wording for this repo in Phase 2)
- `.github/` — `labels.yml`, `trusted-reviewers.json`,
  `pull_request_template.md`, `ISSUE_TEMPLATE/task.yml`,
  `workflows/issue-status.yml`, `workflows/reap.yml`,
  `workflows/ci.yml`, and (per my Phase 0 answer) `workflows/validate.yml`
- `AGENTS.md`, `aw.conf`
- `docs/AUTOMATION.md`, `docs/OPERATIONS.md` (adjust internal links if my
  docs live elsewhere)
- `.claude/skills/onboard-contributor/`, `.claude/skills/triage-task/`
- Append the template's `.gitignore` entries (`.aw/`, `.aw-review.md`,
  `aw.conf.local`) to my existing `.gitignore`
- Do NOT copy the template's `README.md`, `LICENSE`, `CHANGELOG.md`,
  `VERSION`, `CONTRIBUTING.md`, or `SECURITY.md` wholesale — my repo has
  its own. Instead, tell me at the end which sections from the template's
  `SECURITY.md` threat model and `CONTRIBUTING.md` engine-change policy I
  should fold into mine.

## Phase 2 — Adapt to this repo

- `.github/trusted-reviewers.json`: my Phase 0 logins + approval count.
- `.github/CODEOWNERS`: if I have one, add `scripts/`,
  `.github/workflows/`, and `trusted-reviewers.json` routes to my
  maintainers; if not, create one and ask me for the handles.
- `aw.conf`: set my default `AW_AGENT` and any team defaults from
  Phase 0. Leave secrets out.
- `AGENTS.md`: replace generic project references with a one-paragraph
  description of what "doing the work" means in THIS repo (its language,
  test command, and conventions) so agents get real context.
- `prompts/*.md`: adjust the work/review instructions to this repo's
  conventions (build/test commands, review criteria). Ask me whether to
  enable `AW_ENFORCE_TDD=1` (tests-first enforced in both work and
  review prompts).
- `scripts/validate.sh` + `validate.yml` `CONTENT_DIRS`: point at my
  Phase 0 directories, or delete both if I opted out.
- Confirm the scripts' default-branch assumption matches this repo
  (`start_work.sh` creates worktrees from `origin/main` — adjust if my
  default branch differs).

## Phase 3 — GitHub configuration (confirm before each)

1. **Labels** — create every label in `.github/labels.yml` via
   `gh label create ... --force` (10 labels: six `status:*`,
   `review: claimed`, `review: human-only`, `do-not-automate`,
   `priority: high`).
2. **Branch protection** on the default branch — require the
   `aw/merge-gate` status check; do NOT enable GitHub's native
   "require approving reviews" count (docs/AUTOMATION.md explains why
   `merge_ready.sh` replaces it). If I lack admin rights, print the
   exact settings for whoever has them.
3. **Review identity** — per Phase 0: remind me to put
   `REVIEW_GITHUB_TOKEN` in my environment or `aw.conf.local`, or set
   `AW_ALLOW_SOLO_REVIEW=1` there. Never store it yourself.

## Phase 4 — Verify

1. Run `bash -n` on every copied script.
2. Run `./scripts/doctor.sh` and show me the full output. Fix every
   FAIL; explain every WARN and whether it's acceptable for my setup.
3. Run `AW_DRY_RUN=1 ./scripts/start_work.sh` and
   `AW_DRY_RUN=1 AW_POLL_SECONDS=0 ./scripts/reap.sh` — confirm they
   resolve the repo, fetch the (empty) queue, and exit cleanly without
   changing anything.
4. If I kept the example (Phase 0 Q5): offer to walk `docs/SETUP.md`
   end-to-end — create the sample issues, run one real worker cycle, one
   review cycle, and the rework path — before touching real backlog.

## Phase 5 — Hand off

Commit everything on the setup branch with a clear message, open a PR,
and give me a summary containing:

- What was installed and what was adapted (with any conflicts you
  resolved and how)
- The exact commands each runner machine needs (worker loop, review
  loop, and their env vars)
- What's still manual for me: labeling the first issue
  `status: available` (gate G0 — nothing runs until then), branch
  protection if you couldn't set it, token placement, and the
  SECURITY/CONTRIBUTING sections to fold into my docs
- Where to look when something misbehaves: `./scripts/doctor.sh`,
  `.aw/audit.jsonl`, and `docs/OPERATIONS.md`
