---
name: bow-commit
description: Commit and push the Bow way on ANY stack — work on a `<type>/<PROJECT_KEY>-<num>-<slug>` branch (never straight on main/develop), run the repo's own gate and static checks first, then a strict Conventional-Commit message (capitalised Type, gitmoji, WHY body, `Refs:` footer) with NO AI-authorship trailer, then push. Use when the user says "bow commit", "<project> commit" (e.g. "octopus commit"), "commit + push", "push + commit", "đẩy code chuẩn bow", or asks to commit, push, or name a branch on this repo.
---

# Bow Mode — Commit & Push

One commit pipeline for every Bow repo, whatever the stack. Run the steps **in order**
and STOP on any failure (fix, then resume).

Steps marked *if the repo has it* are skipped when the file is not there — never invent
a script or a command the repo does not ship.

`<PROJECT_KEY>` is the repo's Jira project key (`ABC` in `ABC-123`). Take it from the
current branch name, else from recent history, else ask the user:
```bash
git log -20 --format='%s %b' | grep -oE '\b[A-Z][A-Z0-9]+-[0-9]+' | sort | uniq -c | sort -rn | head -3
```
A repo with no Jira project at all: confirm with the user once, then drop the ticket
segment from the branch name and the `Refs:` footer — everything else still applies.

## Hard rules — every repo, no exceptions

1. **No AI-authorship trailer.** The message ENDS at `Refs:`. Never write
   `Co-Authored-By: Claude …`, `Generated with …`, a 🤖 line, or anything else crediting
   an AI — not in the commit, not in the MR/PR description. Commit as the human author
   only. A `commit-msg` hook that scrubs the line is a backstop, not a licence: don't
   write it in the first place.
2. **Never commit on a default / protected branch** (`main`, `master`, `develop`, `dev`).
   Branch first.
3. **Never bypass a gate.** No `--no-verify`, no force-push, no `git add -A` without
   asking, nothing secret or machine-local staged.

If the repo documents its own message format (CLAUDE.md, CONTRIBUTING.md,
`docs/conventions/`), that format wins over § 3 — the three rules above do not bend.

## 0. Pre-flight
- `git status` and `git rev-parse --abbrev-ref HEAD` to see changes + branch.
- On a default branch → create one first: `<type>/<PROJECT_KEY>-<num>-<slug>`
  (type ∈ feat|fix|refactor|perf|test|docs|chore|style; slug = short kebab-case).
- Parse the Jira ticket from the branch name (`fix/<PROJECT_KEY>-800-…` → `<PROJECT_KEY>-800`).
  No ticket in the branch → ask the user for one (the footer needs `Refs: <PROJECT_KEY>-XXX`).
- Stage intended files with `git add <paths>` — scope to what this change touched
  (in a monorepo, the app dir: `git add apps/mobile`). NEVER stage local config or
  secrets: `.claude/settings.local.json`, `.env*`, key/credential files, scratch dirs.
  Confirm with the user before `git add -A`.

## 1. Project gate — BLOCKING
The gate is the repo's own pre-commit script. Its name is the repo's choice
(`scripts/bow-commit.sh`, `scripts/octopus-commit.sh`, …), so look for it instead of
assuming the name. This lists it from the repo ROOT, whatever sub-directory you are in:
```bash
git ls-files --full-name ':(top,glob)scripts/*-commit.sh'
```
- **One match:** open the script before the first run. A gate only CHECKS — it bundles
  that repo's safety scan, branch-name check, analyzer and checklist preview (in the
  Supabase monorepo: `scripts/check-quest.sh --staged` — RLS FK-ownership,
  `security_invoker` views, CORS wildcards, secrets — + `fvm flutter analyze` +
  `scripts/quest-checklist.sh`). If it runs `git add` / `git commit` / `git push` itself
  (`scripts/agent-commit-push.sh`), it is § 4's automated path, not the gate: treat the
  repo as **No match**. Otherwise run it from the repo root (`git rev-parse --show-toplevel`):
  ```bash
  bash scripts/<name>-commit.sh
  ```
- **Several matches:** the one the repo's CLAUDE.md / CONTRIBUTING.md names as the commit
  gate; still unsure → ask the user.
- **No match:** read the staged diff yourself (`git diff --cached`) for secrets
  (API keys, tokens, private keys, service-role keys, `.env` values), wildcard CORS,
  and debug leftovers (stray prints, commented-out blocks, `TODO: remove`).
> Non-zero exit or any hit → STOP, extract the finding (file:line), fix the code,
> re-stage, re-run. Do not commit around it.

## 2. Static analysis for what you touched
Skip what § 1's script already ran. Otherwise run the repo's OWN check for each area
in the staged diff, from that package's directory:

| Staged files live in | Check |
| --- | --- |
| a Flutter/Dart package (`pubspec.yaml`) | `fvm flutter analyze` when the repo pins FVM (`.fvmrc` / `.fvm/`), else `flutter analyze` |
| a Node/TS package (`package.json`) | whichever of the `typecheck`, `lint`, `test` scripts exist, via the repo's package manager |
| anything else | the check named in the repo's README / CLAUDE.md / CI config |

> Zero errors AND zero warnings required. Fix everything before committing.

## 3. Compose the commit message (strict Conventional Commits)
```text
Type(scope): <emoji> Subject matching the Jira task title or a short clear summary

Detailed explanation of WHY these changes were made — at least 3 lines
describing the engineering decisions and the structural / behaviour changes,
not a restatement of the diff.

Refs: <PROJECT_KEY>-XXX
```
Rules — get these exact:
- **Type**: capitalised (`Feat`, `Fix`, `Refactor`, `Perf`, `Test`, `Docs`, `Chore`, `Style`).
- **Gitmoji**: one icon **right after the colon**, never before `Type` (tooling matches
  `^(Feat|Fix|…)` at the start of the subject). Pick by the change's intent (map below),
  e.g. `Fix(notifications): 🌐 Localize push templates per recipient locale`.
- **Subject**: 10–72 chars *including the emoji*, imperative verb (`add`/`fix`/`update`,
  not `adding`/`fixed`).
- **Body**: ≥ 3 lines, explain WHY (not just what).
- **Footer**: `Refs: <PROJECT_KEY>-<num>` — the SPECIFIC ticket THIS commit implements,
  not just the branch's umbrella ticket. Nothing after it (Hard rule 1).
- **No duplicate refs on a branch.** List the refs already used and avoid reusing one
  unless this commit is genuinely the same unit of work (`<base>` = the branch you
  will merge into, e.g. `origin/develop` or `origin/main`):
  ```bash
  git log --format='%b' <base>..HEAD | grep -oE '<PROJECT_KEY>-[0-9]+' | sort -u
  ```
  A branch that touches several tickets should spread **distinct** `Refs:` across its
  commits — one ticket per commit where the work cleanly maps. If the only honest ticket
  is one already used, prefer **amending** the prior commit (when unpushed) over a second
  commit with the duplicate ref.

**Gitmoji map** (https://gitmoji.dev) — *intent wins over the type default*:
- Type default: `Feat` ✨ · `Fix` 🐛 · `Refactor` ♻️ · `Perf` ⚡️ · `Test` ✅ · `Docs` 📝 · `Chore` 🔧 · `Style` 🎨 · `Revert` ⏪️
- Intent override (when the change is mainly this): l10n/i18n 🌐 · security/RLS/secrets 🔒️ · DB schema / migration 🗃️ · remove dead code/files 🔥 · upgrade deps ⬆️ · CI pipeline 👷 · config files 🔧 · critical hotfix 🚑️ · UI/cosmetic only 💄
> Type says "what kind of commit"; emoji says "what it touched". A `Fix` that localizes copy → `Fix(...): 🌐 …` (intent beats the 🐛 default).

## 4. Commit & push
Write the message to a temp file then commit (avoids shell-escaping issues):
```bash
git commit -F <tempfile>
git push -u origin "$(git rev-parse --abbrev-ref HEAD)"
```
*If the repo has* `scripts/agent-commit-push.sh` (stage → ticket-parse → commit → push
in one go), use it only when the user wants the fully-automated path; prefer the explicit
steps above when you need control over staging or the message.

## 4b. Self-verify the quest checklist — *if the repo has `scripts/quest-checklist.sh`*
MANDATORY in those repos — do NOT defer it to the user. § 1 already printed the checklist
preview; READ it, then fill in the "Pre-commit preview" rubric from `.claude/CLAUDE.md`
yourself. For **every** row:
- Decide `✓ / ✗ / N/A` from the ACTUAL staged/branch diff — open the file and cite
  `file:line` as evidence. Not the heuristic's guess, not an aspiration.
- A box you cannot back with a concrete diff signal is `✗`, not `✓`.
- This is YOUR self-review of YOUR work before the user approves; it is NOT the forbidden
  self-*approve* of a teammate's MR.

Emit the full rubric (AI-review block + Process levers + `Score` + `Verdict`). Honest
self-score **< 80% → STOP, do not commit**: fix the code (add the missing check, split a
`Test:` commit, delete dead code…), re-stage, re-run, re-score. Only `Verdict: READY`
proceeds to push.

## 5. Report
Summarise: branch, commit hash + subject, files/insertions/deletions, push result, and
the MR/PR-create URL printed by the push (plus the § 4b rubric where it applies). One
branch = one MR/PR; do NOT self-merge — review goes to a teammate.
