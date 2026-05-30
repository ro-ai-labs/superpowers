# Design Spec: Personal `superpowers` Fork

**Date:** 2026-05-30
**Owner:** mihai
**Status:** Draft for review
**Convention note:** This follows the brainstorming `YYYY-MM-DD-<topic>-design.md` convention. It lives in `~/dev/` (next to the future clone, not inside it, to avoid a `git clone` target conflict). After the fork repo exists, this can be relocated to `docs/superpowers/specs/` inside it.

---

## 1. Goal

Run a personal, git-tracked fork of the `superpowers` Claude Code plugin as the daily driver — with the official `superpowers@claude-plugins-official` disabled — carrying a vetted set of improvements that exploit recent Claude Code built-ins. The fork tracks upstream (`obra/superpowers`) so future releases can be pulled and the customizations rebased on top.

## 2. Decisions (locked)

- **Install/backup model:** Private GitHub repo, installed via a git-hosted marketplace. Gives off-machine backup, a stable `installLocation`, and clean update detection.
- **Change scope:** "Safe core + rework the risky items" — ship the safe changes, gate the test-first one, and rework (not blindly ship) the items the adversarial review flagged.
- **Repo:** `superpowers-fork`, **private**, under the user's GitHub account (`<your-gh-user>/superpowers-fork`).
- **Commit 2:** DROP (keep `finishing-a-development-branch` model-invocable; rely on its existing guards).
- **Commit 7:** Convenience-only for v1 (named `@agent-superpowers:*` invocation + `model` defaults, prose duplicated, no read-only claim); defer genuine sandbox-enforced read-only.
- **Commit 4:** KEEP as a tight optional worktree note, de-duplicated against commit 3.

## 3. Background: why the naive approach fails (from the expert pressure-test)

Two findings reshaped the design:

1. **A marketplace install is a *cache copy*, not a live read of the working tree.** Claude Code copies the plugin into `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`. Therefore git edits/rebases in the working tree do **not** change the running plugin until the cache is refreshed, and a pinned `version` that never changes makes `/plugin marketplace update` a silent no-op. (Evidence: plugins-reference "copies … rather than using them in-place"; version is the cache key.)
2. **"Everything actionable" from the first review contained broken items.** The adversarial pass found that one first-round *recommendation* (disable-model-invocation) actually breaks a workflow, and that the highest-value item (agents/) rests on a false premise and breaks 3 of 4 supported harnesses.

## 4. Architecture

### 4.1 Repo & remotes
- `git clone https://github.com/obra/superpowers ~/dev/superpowers-fork`
- `git remote rename origin upstream` (upstream = obra/superpowers)
- `git remote add origin <private-github-repo>` (the personal backup)
- Work on branch `custom`. Customizations are a small, re-appliable commit stack.

### 4.2 Naming (the key to a zero-rewrite swap)
- **Keep the plugin `name: superpowers`** in `.claude-plugin/plugin.json`. The skill namespace derives from the *plugin* name, so all 84 internal `superpowers:*` references, the SessionStart bootstrap (`superpowers:using-superpowers`), and skill-usage history keep resolving to the fork untouched.
- **Rename only the marketplace** in `.claude-plugin/marketplace.json`: `superpowers-dev` → `superpowers-mihai`. The enable key becomes `superpowers@superpowers-mihai`, distinct from `superpowers@claude-plugins-official`. (This is the maintainers' own side-by-side dev pattern, per their `docs/testing.md`.)

### 4.3 Versioning (deterministic, survives the GitHub install path)
- Use an explicit version with a personal suffix: `5.1.0` → `5.1.0-mihai.1`, bumped on every change meant to go live.
- Bump via the shipped `scripts/bump-version.sh`, which atomically updates all six declared files (keeps `plugin.json` and `marketplace.json` in sync, avoiding the "plugin.json wins silently" divergence trap). The suffix passes the script's version regex.
- Rationale: explicit bumps are deterministic regardless of whether a local-vs-git source is treated as SHA-versioned. Each bump + `/plugin marketplace update` is what makes edits actually load.

### 4.4 Cutover (use `/plugin` commands, not hand-edited JSON)
Editing `enabledPlugins` by hand risks the `settings.local.json` merge bug (#25086/#27247). Sequence:
1. `/plugin marketplace add <owner>/<repo>` (git-hosted)
2. `/plugin install superpowers@superpowers-mihai`
3. `/plugin disable superpowers@claude-plugins-official`
4. `/plugin enable superpowers@superpowers-mihai`
5. `/reload-plugins` or restart Claude Code

## 5. The change-set

Commits ordered for independence. Each is its own commit for clean revert. **Commit 0 is mandatory first.**

| # | Commit | File(s) | Disposition |
|---|--------|---------|-------------|
| 0 | Add `.in_use/` (PID lockfiles) to `.gitignore` | `.gitignore` | **Required first** — prevents runtime lockfiles polluting the tree / breaking rebases |
| 1 | Document current CC skill frontmatter fields (portability-annotated table, in body not frontmatter) | `skills/writing-skills/SKILL.md` | **SAFE — ship** |
| 3 | `EnterWorktree` `baseRef: fresh` gotcha note, scoped to Claude Code | `skills/using-git-worktrees/SKILL.md` | **SAFE — ship** |
| 5 | Drop manual JSON escaper on the CC branch; print bootstrap to stdout (keep Cursor/Copilot JSON branches; `printf '%s'`; no heredoc) | `hooks/session-start` | **TEST FIRST** — empirically confirm raw SessionStart stdout reaches the model before shipping |
| 6 | Structured `schema` returns as an **optional CC-only layer**; keep the prose `**Status:** DONE\|DONE_WITH_CONCERNS\|BLOCKED\|NEEDS_CONTEXT` contract canonical | `skills/subagent-driven-development/SKILL.md` | **RE-SCOPED** — additive layer, never replaces prose path |
| 2 | ~~`disable-model-invocation`~~ | `skills/finishing-a-development-branch/SKILL.md` | **DROP (decided)** — see 5.1 |
| 4 | (reworked) Tight, optional `isolation:"worktree"` note for overlapping-file fan-out; reference (don't duplicate) the baseRef caveat from commit 3 | `skills/dispatching-parallel-agents/SKILL.md` | **REWORK** — see 5.2 |
| 7 | (reworked) Add CC-native `agents/*.md` reviewer defs that **duplicate (not relocate)** role+criteria; drop the false read-only-enforcement claim; keep full templates for cross-harness | new `agents/`, `skills/subagent-driven-development/*`, `skills/requesting-code-review/*`, `skills/using-superpowers/references/*-tools.md` | **REWORK — highest risk** — see 5.3 |

### 5.1 Commit 2 rework — honest outcome
The original idea (`disable-model-invocation: true`) **breaks** the auto-handoff: `subagent-driven-development/SKILL.md:66,85` and `executing-plans/SKILL.md:35-36` invoke `finishing-a-development-branch` via the Skill tool with no user turn between. Disabling model invocation dead-ends both flows at "Done!" with work never integrated.

The skill **already guards destructive ops** (requires passing tests, typed "discard" confirmation, never force-pushes). So the rework is: **do not disable model invocation.** If extra safety is still wanted, reinforce the in-skill confirmation rather than the invocation gate. **Net recommendation: effectively drop commit 2; document the analysis so the decision is explicit.** (If the user wants the invocation gate anyway, the cost is rewriting the two callers to hand off to the user explicitly — a deliberate change to superpowers' "don't pause between tasks" philosophy, called out as out-of-scope unless chosen.)

### 5.2 Commit 4 rework
Keep it a brief, **optional** note in `dispatching-parallel-agents`: when parallel agents will edit overlapping files, give each `isolation:"worktree"` to avoid collisions. State the baseRef caveat **once** (it lives fully in commit 3's `using-git-worktrees`); here, reference it rather than duplicate it, to prevent drift. If on review this feels altitude-mismatched for a parallel-*debugging* skill, drop it.

### 5.3 Commit 7 rework (the big one)
- **Drop** the "`tools: …, Bash` enforces read-only" framing — it is false (a reviewer with Bash can write files). Do not claim enforcement.
- **Duplicate, don't relocate.** Add `agents/spec-reviewer.md`, `agents/code-quality-reviewer.md`, `agents/code-reviewer.md` that mirror the reviewer role + review criteria so Claude Code gets named `@agent-superpowers:*` invocation and `model` defaults — while the existing prompt templates stay **intact**, because Gemini/Codex/Copilot consume the full template text (`gemini-tools.md:33-35`) and have no `agents/` mechanism.
- **Preserve verbatim** the eval-tuned "do not trust the report; verify by reading code + SHAs" prose (`spec-reviewer-prompt.md:24-37`, `code-reviewer.md` critical rules).
- **Genuine read-only (optional stretch):** if real read-only review is wanted, restructure so the controller passes the diff/SHAs to a `Read/Grep/Glob`-only reviewer instead of having it run `git diff` via Bash. Decide during planning; do not over-reach in v1.
- Accept the maintenance cost of duplicated prose (CC agents vs cross-harness templates) as the price of not breaking other harnesses.

## 6. Verification & cutover safety

- **Single-bootstrap check:** after restart, the "You have superpowers" block must appear **exactly once**. Two = both plugins enabled (double injection); recheck `settings.local.json` and `/plugin` state.
- **Marker-based acceptance test:** embed a unique marker in one customized skill; confirm it loads and that `/plugin` shows the skill's source is `superpowers@superpowers-mihai`. (The naive "brainstorming auto-triggers" test is confounded — usageCount is already 24.)
- **Breadth smoke test:** confirm `superpowers:writing-plans`, `superpowers:test-driven-development`, `superpowers:requesting-code-review` still resolve.
- **`/plugin` → Errors tab** must be clean (catches manifest/path/hook load failures the prompt won't surface).
- **Commit-5 empirical test:** on a real CC session, inject a newline/quote-heavy bootstrap via raw stdout and confirm the model receives it; otherwise keep the JSON envelope.

## 7. Rollback (two tiers)
- **Tier 1 — instant revert:** flip the two `enabledPlugins` booleans (official → true, fork → false), restart. Fork stays installed-but-disabled. This is the everyday safety net.
- **Tier 2 — full teardown:** uninstall the fork, then `/plugin marketplace remove superpowers-mihai` (**destructive** — removing the marketplace uninstalls its plugins), then delete the cache dir. Only when deliberately decommissioning.
- Do **not** delete the official cache while sessions hold it (`.in_use/` PID locks); changes apply on a fresh session.

## 8. Maintenance (tracking upstream)
- Update: `git fetch upstream && git rebase upstream/main custom`. Push to private `origin` after; since `custom` is rebased, this is a force-push to your own backup branch (acceptable for a personal fork) — or use merge if you prefer non-rewritten history.
- **The six version-bump files conflict on every upstream release** (upstream bumps `5.x`, you carry `-mihai.N`). Strategy: keep the version bump as the **last** commit on `custom` so it's a single predictable re-apply; on conflict, take upstream's base version then re-run `bump-version.sh <upstream>-mihai.1`.
- After any update: `bump-version.sh` → commit → push → `/plugin marketplace update superpowers-mihai` → `/plugin update superpowers@superpowers-mihai` → `/reload-plugins`. Without the bump, the update is a no-op.
- Never PR customizations to `obra/superpowers` (their CLAUDE.md rejects fork-specific changes); the fork is private.

## 9. Execution roles
- **Claude:** clone, all file edits/commits, run `bump-version.sh`, push (if `gh`/git creds available), author the verification checklist, and (where possible) validate JSON/YAML + run any runnable tests in `tests/`.
- **User (interactive):** create the private GitHub repo, provide/confirm git credentials, run the `/plugin …` commands, restart Claude Code, and perform the post-restart smoke checks (single-bootstrap, marker, Errors tab).

## 10. Out of scope (v1)
- Rewriting `subagent-driven-development`/`executing-plans` callers to remove the finishing auto-handoff (only if the invocation-gate is explicitly chosen later).
- Genuine sandbox-enforced read-only review (the diff-passing restructure) unless chosen during planning.
- Any change to behavior-shaping prose beyond the additive/duplicative moves above.
- Cross-harness re-testing on actual Gemini/Codex/Copilot beyond static preservation of the template path.

## 11. Resolved decisions (was: open questions)
1. **Commit 2:** DROP — keep `finishing-a-development-branch` model-invocable; its existing guards (passing tests, typed "discard" confirmation, never force-push) cover the destructive-op concern. Callers are NOT rewritten.
2. **Commit 7:** Convenience-only for v1 — ship `agents/*.md` for named invocation + `model` defaults, prose duplicated (not relocated), no read-only-enforcement claim. Genuine sandbox-enforced read-only (diff-passing restructure) is deferred.
3. **Repo:** `<your-gh-user>/superpowers-fork`, private. Confirm the exact `<your-gh-user>` at execution time for `/plugin marketplace add <owner>/superpowers-fork`.
4. **Commit 4:** KEEP the optional worktree note, de-duplicated against commit 3.
