# Personal superpowers Fork — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a private, git-tracked fork of the `superpowers` plugin carrying a vetted set of improvements, installed via a private GitHub marketplace with the official plugin disabled.

**Architecture:** Clone `obra/superpowers`, keep the plugin name `superpowers` (preserves all `superpowers:*` refs), rename only the marketplace to `superpowers-mihai`, apply changes as a small commit stack on branch `custom`, push to a private GitHub repo, install via git marketplace, and swap via `/plugin` commands.

**Tech Stack:** git, GitHub (`gh`), bash (hook + `bump-version.sh`), Markdown skills, Claude Code plugin/marketplace system.

**Source spec:** `~/dev/superpowers-fork-design-2026-05-30.md`

---

## TDD adaptation note (read first)

This plan is part infrastructure (git/plugin) and part content (Markdown skills + one bash hook). Classic unit-test TDD doesn't apply to prose edits, so "verification" takes the strongest available form per task:
- **JSON/YAML**: parse-validate (`python3 -c json.load` / frontmatter parse).
- **Bash hook**: execute and inspect output.
- **Skill edits**: grep the inserted anchor/marker + confirm frontmatter still parses.
- **Plugin load**: post-install smoke tests (single-bootstrap, marker, Errors tab, breadth).
Every task ends in a commit. Conventional-commit messages.

**Execution roles:** Tasks 0–13 are Claude-executable. Tasks 14–17 are **user-interactive** (GitHub repo creation may need auth; `/plugin` commands and restart are user actions). Each interactive task says so.

---

## Phase 0 — Repo setup

### Task 0: Verify prerequisites

**Files:** none

- [ ] **Step 1: Check tooling and clean target**

Run:
```bash
git --version && gh --version && gh auth status 2>&1 | head -3
test ! -e ~/dev/superpowers-fork && echo "TARGET CLEAR" || echo "TARGET EXISTS — stop"
```
Expected: git + gh present; `gh auth status` shows logged in; `TARGET CLEAR`.
If `gh` is absent or unauthenticated, stop and tell the user (they may run `! gh auth login`).

---

### Task 1: Clone, set remotes, create branch

**Files:** creates `~/dev/superpowers-fork/`

- [ ] **Step 1: Clone and set up remotes/branch**

Run:
```bash
git clone https://github.com/obra/superpowers ~/dev/superpowers-fork
cd ~/dev/superpowers-fork
git remote rename origin upstream
git switch -c custom
```
Expected: clone succeeds; `git remote -v` shows only `upstream` → obra/superpowers; on branch `custom`.

- [ ] **Step 2: Confirm manifests present**

Run:
```bash
cd ~/dev/superpowers-fork
test -f .claude-plugin/plugin.json && test -f .claude-plugin/marketplace.json && echo "MANIFESTS OK"
```
Expected: `MANIFESTS OK`.

- [ ] **Step 3: No commit yet** (branch creation only). Proceed.

---

### Task 2: Commit 0 — gitignore the `.in_use/` runtime locks

**Files:**
- Modify: `~/dev/superpowers-fork/.gitignore`

- [ ] **Step 1: Append ignore rules**

Append these lines to `.gitignore`:
```gitignore

# Runtime PID lockfiles created when the plugin is live (must never be committed)
.in_use/
*.in_use
```

- [ ] **Step 2: Verify and clean any stray locks**

Run:
```bash
cd ~/dev/superpowers-fork
git check-ignore .in_use/ && echo "IGNORED OK"
git status --porcelain | grep -i in_use && echo "WARN: locks tracked" || echo "clean"
```
Expected: `IGNORED OK`; `clean`.

- [ ] **Step 3: Commit**

```bash
cd ~/dev/superpowers-fork
git add .gitignore
git commit -m "chore: gitignore runtime .in_use lockfiles"
```

---

### Task 3: Fork identity — marketplace rename, version, load-marker

**Files:**
- Modify: `~/dev/superpowers-fork/.claude-plugin/marketplace.json` (name)
- Modify (via script): the six version-bump files
- Modify: `~/dev/superpowers-fork/skills/using-superpowers/SKILL.md` (append marker)

- [ ] **Step 1: Rename the marketplace**

In `.claude-plugin/marketplace.json`, change `"name": "superpowers-dev"` to `"name": "superpowers-mihai"`. Leave the inner plugin `"name": "superpowers"` UNCHANGED.

- [ ] **Step 2: Bump version with personal suffix**

Run:
```bash
cd ~/dev/superpowers-fork
bash scripts/bump-version.sh 5.1.0-mihai.1
```
Expected: script reports updating package.json, both plugin.json files, both other plugin manifests, marketplace.json `plugins.0.version`, gemini-extension.json. If the script rejects the suffix, set `"version": "5.1.0-mihai.1"` by hand in `.claude-plugin/plugin.json` AND `.claude-plugin/marketplace.json` `plugins[0].version` (keep them identical).

- [ ] **Step 3: Add a load-marker to the bootstrap skill**

Append exactly this line to the END of `skills/using-superpowers/SKILL.md` (HTML comment — invisible when rendered, present in the injected bootstrap text so we can prove the fork loaded):
```markdown

<!-- superpowers-mihai-fork: v5.1.0-mihai.1 -->
```

- [ ] **Step 4: Validate JSON**

Run:
```bash
cd ~/dev/superpowers-fork
for f in .claude-plugin/plugin.json .claude-plugin/marketplace.json package.json; do
  python3 -c "import json,sys; json.load(open('$f')); print('OK $f')"
done
grep -n '"name": "superpowers-mihai"' .claude-plugin/marketplace.json
grep -n '"name": "superpowers"' .claude-plugin/plugin.json
grep -c 'superpowers-mihai-fork' skills/using-superpowers/SKILL.md
```
Expected: three `OK`; marketplace name is `superpowers-mihai`; plugin name still `superpowers`; marker count `1`.

- [ ] **Step 5: Commit**

```bash
cd ~/dev/superpowers-fork
git add -A
git commit -m "chore: brand fork (marketplace superpowers-mihai, v5.1.0-mihai.1, load-marker)"
```

---

## Phase 1 — Safe content edits

### Task 4: Commit 1 — document CC frontmatter fields in writing-skills

**Files:**
- Modify: `~/dev/superpowers-fork/skills/writing-skills/SKILL.md` (after the Frontmatter list, ~line 104, before the ```markdown example at line 105)

- [ ] **Step 1: Verify field names against current docs**

Before inserting, open https://code.claude.com/docs/en/skills (Frontmatter reference) and confirm the field names below still exist. Correct any drift. (Wrong frontmatter names are worse than none.)

- [ ] **Step 2: Insert the table**

Insert this subsection immediately AFTER line 103 (`  - Keep under 500 characters if possible`) and BEFORE the ```markdown block at line 105:
```markdown

**Optional frontmatter fields.** `name` + `description` are all you need and the most portable. These additional fields unlock behavior; the **Std** column marks open-standard (portable, per agentskills.io/specification) vs **CC** (Claude Code extension — ignored on other harnesses, so a skill must not *depend* on them for correctness):

| Field | Effect | Std |
|---|---|---|
| `license`, `compatibility`, `metadata` | SPDX license / declared compatibility / arbitrary key-values | Std |
| `allowed-tools` | Pre-approves the listed tools while the skill is active (fewer prompts). NOT a sandbox — does not restrict other tools | Std (experimental) |
| `disallowed-tools` | Blocks the listed tools while the skill is active (this is the real restrictor) | CC |
| `disable-model-invocation` | Skill runs only when a human invokes it; the model can't auto-trigger it. Use for side-effecting/destructive workflows | CC |
| `user-invocable` | Whether a user can run it as a `/command` (skills are user-invocable by default) | CC |
| `argument-hint`, `arguments` | Slash-command argument hinting and `$ARGUMENTS`/`$1` substitution | CC |
| `context: fork` | Run the skill in a forked context | CC |
| `agent`, `model`, `effort` | Bind to a named agent / model hint / effort hint | CC |
| `paths`, `hooks`, `when_to_use`, `shell` | Path scoping / skill-scoped hooks / alt trigger field / shell selection | CC |

Treat CC fields as enhancements layered on a skill that already works with just `name` + `description`.
```

- [ ] **Step 3: Verify**

Run:
```bash
cd ~/dev/superpowers-fork
grep -n "Optional frontmatter fields" skills/writing-skills/SKILL.md
head -1 skills/writing-skills/SKILL.md | grep -q '^---' && echo "frontmatter intact"
```
Expected: the heading is found; frontmatter intact.

- [ ] **Step 4: Commit**

```bash
cd ~/dev/superpowers-fork
git add skills/writing-skills/SKILL.md
git commit -m "docs(writing-skills): document optional CC frontmatter fields with portability column"
```

---

### Task 5: Commit 3 — EnterWorktree baseRef note in using-git-worktrees

**Files:**
- Modify: `~/dev/superpowers-fork/skills/using-git-worktrees/SKILL.md` (inside Step 1a, after line 55)

- [ ] **Step 1: Insert the note**

Insert this paragraph immediately AFTER line 55 (`Native tools handle directory placement...`) and BEFORE line 57 (`Only proceed to Step 1b...`):
```markdown

**Base-ref caveat (Claude Code `EnterWorktree`):** the new worktree's base is governed by the `worktree.baseRef` setting — `fresh` (default) branches from `origin/<default-branch>`, `head` branches from your current local HEAD. So by default a native worktree does **not** carry your uncommitted/local-only work. If you intended to build on local commits, request `head` base or use the Step 1b git fallback (which branches from current HEAD via `-b`).
```

- [ ] **Step 2: Verify**

Run:
```bash
cd ~/dev/superpowers-fork
grep -n "Base-ref caveat" skills/using-git-worktrees/SKILL.md
```
Expected: one match around line 56.

- [ ] **Step 3: Commit**

```bash
cd ~/dev/superpowers-fork
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): note EnterWorktree baseRef:fresh branches from origin not HEAD"
```

---

### Task 6: Commit 4 — optional worktree-isolation note in dispatching-parallel-agents

**Files:**
- Modify: `~/dev/superpowers-fork/skills/dispatching-parallel-agents/SKILL.md` (in the "Dispatch in Parallel" subsection)

- [ ] **Step 1: Locate the anchor**

Run:
```bash
cd ~/dev/superpowers-fork
grep -n "Dispatch in Parallel\|All three run concurrently" skills/dispatching-parallel-agents/SKILL.md
```
Note the line just after the parallel-dispatch code block (the `// All three run concurrently` comment).

- [ ] **Step 2: Insert the optional note**

Immediately after that code block, insert:
```markdown

**If the parallel agents will EDIT overlapping files** (not just read), give each its own isolation so they can't collide — e.g. dispatch with `isolation:"worktree"` where your harness supports it, then review/merge each worktree. Note the base-ref caveat from `superpowers:using-git-worktrees` (a fresh worktree branches from the default branch, not your current HEAD), so this fits independent fixes on a clean base — not work that stacks on uncommitted changes. For agents editing disjoint files, the shared workspace is fine and simpler.
```

- [ ] **Step 3: Verify**

Run:
```bash
cd ~/dev/superpowers-fork
grep -n 'isolation:"worktree"' skills/dispatching-parallel-agents/SKILL.md
```
Expected: one match.

- [ ] **Step 4: Commit**

```bash
cd ~/dev/superpowers-fork
git add skills/dispatching-parallel-agents/SKILL.md
git commit -m "docs(dispatching-parallel-agents): optional worktree isolation note for overlapping-file fan-out"
```

---

### Task 7: Commit 6 — optional CC schema layer in subagent-driven-development

**Files:**
- Modify: `~/dev/superpowers-fork/skills/subagent-driven-development/SKILL.md` (after the "Handling Implementer Status" section, after line 120, before "## Prompt Templates")

- [ ] **Step 1: Insert the optional layer (prose contract stays canonical)**

Insert AFTER line 120 (`**Never** ignore an escalation...`) and BEFORE line 122 (`## Prompt Templates`):
```markdown

### Optional: structured status (Claude Code)

The four statuses above are the **canonical** contract — every harness relies on the implementer writing `**Status:** ...` in prose, so keep requiring it. On Claude Code you *may* additionally pass a JSON `schema` on the implementer/reviewer dispatch to get a validated field for the control-flow switch, e.g. implementer `{ status: enum[DONE,DONE_WITH_CONCERNS,BLOCKED,NEEDS_CONTEXT], concerns: string }`, reviewer `{ verdict: enum[approved,issues_found], issues: [...] }`. This is additive: the prose status remains the source of truth and the fallback on harnesses without structured returns. Do not remove the prose `**Status:**` requirement.
```

- [ ] **Step 2: Verify**

Run:
```bash
cd ~/dev/superpowers-fork
grep -n "Optional: structured status" skills/subagent-driven-development/SKILL.md
```
Expected: one match.

- [ ] **Step 3: Commit**

```bash
cd ~/dev/superpowers-fork
git add skills/subagent-driven-development/SKILL.md
git commit -m "docs(subagent-driven-development): optional CC structured-status layer (prose stays canonical)"
```

---

## Phase 2 — Test-first hook change (OPTIONAL / GATED)

### Task 8: Commit 5 — simplify SessionStart on the Claude Code branch

> This is polish (removes a fragile JSON escaper) and is **gated on an empirical test**. The current code deliberately JSON-wraps the CC branch. If the test fails or you'd rather not risk the load-bearing bootstrap, **skip this task** — everything else is independent of it.

**Files:**
- Modify: `~/dev/superpowers-fork/hooks/session-start` (CC branch only, ~lines 49–51)

- [ ] **Step 1: Baseline — current hook emits valid JSON**

Run:
```bash
cd ~/dev/superpowers-fork
CLAUDE_PLUGIN_ROOT="$PWD" bash hooks/session-start | python3 -c "import json,sys; d=json.load(sys.stdin); print('VALID JSON; has context:', 'additionalContext' in d.get('hookSpecificOutput',{}))"
```
Expected: `VALID JSON; has context: True`.

- [ ] **Step 2: LIVE TEST (user) — does raw SessionStart stdout reach the model?**

The user temporarily configures a throwaway SessionStart hook that prints a raw, multi-line, quote-containing string to stdout (no JSON), starts a fresh Claude Code session, and confirms the model received that text. If yes → proceed. If no → STOP, keep the JSON envelope, skip the rest of this task.

- [ ] **Step 3: Apply the change (only if Step 2 passed)**

Replace the Claude Code branch. Find:
```bash
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
  # Claude Code sets CLAUDE_PLUGIN_ROOT without COPILOT_CLI
  printf '{\n  "hookSpecificOutput": {\n    "hookEventName": "SessionStart",\n    "additionalContext": "%s"\n  }\n}\n' "$session_context"
```
Replace with (raw stdout — no escaping needed on this branch; use `printf '%s'` to avoid format-string injection):
```bash
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
  # Claude Code: SessionStart stdout is injected as context verbatim, so emit raw text
  # (no JSON envelope, no escaping). Cursor/Copilot branches still need JSON below.
  printf '%s\n' "$session_context_raw"
```
Then, near where `session_context` is built (after line 35), add a raw (unescaped) variant used only by the CC branch:
```bash
session_context_raw="<EXTREMELY_IMPORTANT>
You have superpowers.

**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'Skill' tool:**

${using_superpowers_content}

${warning_message}
</EXTREMELY_IMPORTANT>"
```
(Leave the existing escaped `session_context` and the Cursor/Copilot `printf` branches untouched. Do NOT introduce a heredoc — see the issue #571 comment in the file header.)

- [ ] **Step 4: Verify both paths still produce output**

Run:
```bash
cd ~/dev/superpowers-fork
echo "--- CC (raw) ---"; CLAUDE_PLUGIN_ROOT="$PWD" bash hooks/session-start | head -3
echo "--- Cursor (json) ---"; CURSOR_PLUGIN_ROOT="$PWD" CLAUDE_PLUGIN_ROOT="$PWD" bash hooks/session-start | python3 -c "import json,sys; json.load(sys.stdin); print('cursor JSON valid')"
```
Expected: CC path prints raw `<EXTREMELY_IMPORTANT>` text starting at line 1; Cursor path prints valid JSON.

- [ ] **Step 5: Commit**

```bash
cd ~/dev/superpowers-fork
git add hooks/session-start
git commit -m "refactor(hook): emit raw stdout on Claude Code SessionStart branch (keep JSON for Cursor/Copilot)"
```

---

## Phase 3 — Reworked agents (Commit 7, convenience-only)

> Convenience-only: named `@agent-superpowers:*` invocation + `model` defaults. Prose is **duplicated, not relocated** — the existing prompt templates stay intact for Gemini/Codex/Copilot. **No read-only-enforcement claim** (a `tools:` list with `Bash` does not prevent writes).

### Task 9: Create `agents/spec-reviewer.md`

**Files:**
- Create: `~/dev/superpowers-fork/agents/spec-reviewer.md`

- [ ] **Step 1: Write the agent file**

Create `agents/spec-reviewer.md` with this frontmatter, then a body that **copies verbatim** the reviewer instructions from `skills/subagent-driven-development/spec-reviewer-prompt.md` lines 11–60 (the "You are reviewing whether…" through the "Report:" block), adapting only the leading sentence to drop the `Task tool` wrapper:
```markdown
---
name: spec-reviewer
description: Use to verify an implementation matches its specification — nothing more, nothing less — by reading the actual code, not trusting the implementer's report.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are reviewing whether an implementation matches its specification.

<!-- BODY: paste verbatim from skills/subagent-driven-development/spec-reviewer-prompt.md lines 21-60
     (## CRITICAL: Do Not Trust the Report  …through…  the ✅/❌ Report block).
     The controller supplies "What Was Requested" and "What Implementer Claims" at dispatch time. -->
```

- [ ] **Step 2: Verify frontmatter parses + prose preserved**

Run:
```bash
cd ~/dev/superpowers-fork
python3 - <<'PY'
import re
t=open('agents/spec-reviewer.md').read()
fm=re.match(r'^---\n(.*?)\n---\n', t, re.S)
assert fm, "no frontmatter"
for k in ('name:','description:'): assert k in fm.group(1), k
assert "Do Not Trust the Report" in t, "missing eval-tuned prose"
assert "Verify by reading code" in t, "missing verify-by-reading prose"
print("spec-reviewer agent OK")
PY
```
Expected: `spec-reviewer agent OK`.

- [ ] **Step 3: Commit**

```bash
cd ~/dev/superpowers-fork
git add agents/spec-reviewer.md
git commit -m "feat(agents): add CC-native spec-reviewer (prose duplicated from template)"
```

---

### Task 10: Create `agents/code-quality-reviewer.md`

**Files:**
- Create: `~/dev/superpowers-fork/agents/code-quality-reviewer.md`
- Reference (read for verbatim copy): `~/dev/superpowers-fork/skills/subagent-driven-development/code-quality-reviewer-prompt.md`

- [ ] **Step 1: Write the agent file**

Read `code-quality-reviewer-prompt.md` fully. Create `agents/code-quality-reviewer.md` with this frontmatter, then copy the reviewer's role + criteria + output-format verbatim from that template (drop only the `Task tool (general-purpose):` wrapper lines):
```markdown
---
name: code-quality-reviewer
description: Use to review implemented code for quality — naming, structure, duplication, error handling, test design — by reading the actual diff, not trusting the report.
tools: Read, Grep, Glob, Bash
model: inherit
---

<!-- BODY: paste verbatim the reviewer instructions from
     skills/subagent-driven-development/code-quality-reviewer-prompt.md
     (everything inside its prompt: |  block). The controller supplies the task text and BASE_SHA/HEAD_SHA at dispatch. -->
```

- [ ] **Step 2: Verify**

Run:
```bash
cd ~/dev/superpowers-fork
python3 - <<'PY'
import re
t=open('agents/code-quality-reviewer.md').read()
assert re.match(r'^---\n.*?name:.*?description:.*?\n---\n', t, re.S), "frontmatter incomplete"
assert len(t.split('---',2)[2].strip())>200, "body too short — prose not copied"
print("code-quality-reviewer agent OK")
PY
```
Expected: `code-quality-reviewer agent OK`.

- [ ] **Step 3: Commit**

```bash
cd ~/dev/superpowers-fork
git add agents/code-quality-reviewer.md
git commit -m "feat(agents): add CC-native code-quality-reviewer (prose duplicated from template)"
```

---

### Task 11: Create `agents/code-reviewer.md`

**Files:**
- Create: `~/dev/superpowers-fork/agents/code-reviewer.md`
- Reference (read for verbatim copy): `~/dev/superpowers-fork/skills/requesting-code-review/code-reviewer.md`

- [ ] **Step 1: Write the agent file**

Read `skills/requesting-code-review/code-reviewer.md` fully. Create `agents/code-reviewer.md` with this frontmatter, then copy its reviewer role, severity calibration, and DO/DON'T critical rules verbatim:
```markdown
---
name: code-reviewer
description: Use for a thorough pre-merge code review of a change set — correctness, severity-calibrated issues, merge readiness — verifying by reading the diff and code directly.
tools: Read, Grep, Glob, Bash
model: inherit
---

<!-- BODY: paste verbatim the reviewer instructions, severity calibration, and Critical Rules (DO/DON'T)
     from skills/requesting-code-review/code-reviewer.md. Controller supplies the diff/SHAs at dispatch. -->
```

- [ ] **Step 2: Verify**

Run:
```bash
cd ~/dev/superpowers-fork
python3 - <<'PY'
import re
t=open('agents/code-reviewer.md').read()
assert re.match(r'^---\n.*?name:.*?description:.*?\n---\n', t, re.S), "frontmatter incomplete"
assert len(t.split('---',2)[2].strip())>200, "body too short — prose not copied"
print("code-reviewer agent OK")
PY
```
Expected: `code-reviewer agent OK`.

- [ ] **Step 3: Commit**

```bash
cd ~/dev/superpowers-fork
git add agents/code-reviewer.md
git commit -m "feat(agents): add CC-native code-reviewer (prose duplicated from template)"
```

---

### Task 12: Wire the optional CC dispatch path (templates stay intact)

**Files:**
- Modify: `~/dev/superpowers-fork/skills/subagent-driven-development/SKILL.md` (Prompt Templates section, ~line 122)
- Modify: `~/dev/superpowers-fork/skills/requesting-code-review/SKILL.md` (where it dispatches the reviewer)

- [ ] **Step 1: Add an opt-in pointer in subagent-driven-development**

After line 126 (the `./code-quality-reviewer-prompt.md` bullet), insert:
```markdown

**On Claude Code**, you may instead dispatch the named agents `superpowers:spec-reviewer` and `superpowers:code-quality-reviewer` (defined in this plugin's `agents/`), passing the same per-task payload (task text, claims, `BASE_SHA`/`HEAD_SHA`). They carry the identical review criteria. On other harnesses, use the prompt templates above — they remain the source of truth.
```

- [ ] **Step 2: Add the analogous pointer in requesting-code-review**

Run `grep -n "code-reviewer" skills/requesting-code-review/SKILL.md` to find where the reviewer is dispatched, and add a one-line note there:
```markdown

(On Claude Code you may dispatch the named `superpowers:code-reviewer` agent instead of pasting the template; it carries the same instructions.)
```

- [ ] **Step 3: Verify templates untouched**

Run:
```bash
cd ~/dev/superpowers-fork
git diff --name-only HEAD | grep -E 'prompt\.md|code-reviewer\.md$' && echo "WARN: a template changed — revert it" || echo "templates intact"
grep -n "superpowers:spec-reviewer" skills/subagent-driven-development/SKILL.md
```
Expected: `templates intact`; the pointer line is present.

- [ ] **Step 4: Commit**

```bash
cd ~/dev/superpowers-fork
git add skills/subagent-driven-development/SKILL.md skills/requesting-code-review/SKILL.md
git commit -m "docs: point CC users to named reviewer agents (cross-harness templates unchanged)"
```

---

## Phase 4 — Local validation

### Task 13: Validate the whole fork statically

**Files:** none (read-only checks)

- [ ] **Step 1: All JSON valid**

Run:
```bash
cd ~/dev/superpowers-fork
find . -path ./.git -prune -o -name '*.json' -print | while read f; do
  python3 -c "import json; json.load(open('$f'))" || echo "BAD JSON: $f"
done; echo "json scan done"
```
Expected: only `json scan done` (no `BAD JSON`).

- [ ] **Step 2: Every skill + agent frontmatter parses**

Run:
```bash
cd ~/dev/superpowers-fork
python3 - <<'PY'
import re,glob,sys
bad=[]
for f in glob.glob('skills/*/SKILL.md')+glob.glob('agents/*.md'):
    t=open(f).read()
    m=re.match(r'^---\n(.*?)\n---\n', t, re.S)
    if not m or 'name:' not in m.group(1) or 'description:' not in m.group(1):
        bad.append(f)
print("BAD:",bad) if bad else print("all frontmatter OK")
PY
```
Expected: `all frontmatter OK`.

- [ ] **Step 3: Run any shipped test runner if present**

Run:
```bash
cd ~/dev/superpowers-fork
python3 -c "import json;print(json.load(open('package.json')).get('scripts',{}))"
ls tests/ 2>/dev/null
```
If `package.json` has a `test` script, run it (`npm test`) and record the result. If tests need a harness not available, note "tests require <harness>; skipped" — do not fake a pass.

- [ ] **Step 4: Commit marker (no code change) — none.** Validation only; proceed.

---

## Phase 5 — Publish, install, cutover (USER-INTERACTIVE)

### Task 14: Push to a private GitHub repo

**Files:** none (remote)

- [ ] **Step 1: Create the private repo and push (confirm the GH user first)**

Confirm the exact GitHub account, then run (replace `<your-gh-user>`):
```bash
cd ~/dev/superpowers-fork
gh repo create <your-gh-user>/superpowers-fork --private --source=. --remote=origin --push
git push -u origin custom
```
Expected: repo created private; `custom` pushed; `git remote -v` shows `origin` → your repo and `upstream` → obra.

- [ ] **Step 2: Verify remote**

Run: `gh repo view <your-gh-user>/superpowers-fork --json visibility,defaultBranchRef`
Expected: `"visibility":"PRIVATE"`.

---

### Task 15: Register the marketplace and install (user runs `/plugin`)

**Files:** none

- [ ] **Step 1: Add marketplace + install**

In Claude Code, the user runs:
```
/plugin marketplace add <your-gh-user>/superpowers-fork
/plugin install superpowers@superpowers-mihai
```
Expected: marketplace `superpowers-mihai` registered; plugin installs without error.

- [ ] **Step 2: Confirm install record**

Run (shell): `python3 -c "import json;print([k for k in json.load(open('$HOME/.claude/plugins/installed_plugins.json')) if 'superpowers' in k])"`
Expected: list includes `superpowers@superpowers-mihai`.

---

### Task 16: Swap and restart (user)

**Files:** `~/.claude/settings.json` (via `/plugin`)

- [ ] **Step 1: Disable official, enable fork**

User runs:
```
/plugin disable superpowers@claude-plugins-official
/plugin enable superpowers@superpowers-mihai
```

- [ ] **Step 2: Confirm exactly one is enabled**

Run (shell):
```bash
python3 -c "import json;d=json.load(open('$HOME/.claude/settings.json')).get('enabledPlugins',{});print({k:v for k,v in d.items() if 'superpowers' in k})"
grep -i superpowers ~/.claude/settings.local.json 2>/dev/null && echo "CHECK local override" || echo "no local override"
```
Expected: official `False`, fork `True`; no conflicting `settings.local.json` key.

- [ ] **Step 3: Restart Claude Code** (or `/reload-plugins`). Required for hooks to load.

---

### Task 17: Post-restart smoke tests

**Files:** none

- [ ] **Step 1: Single-bootstrap check**

In a fresh session, confirm the "You have superpowers" bootstrap appears **exactly once** (two = both plugins enabled; recheck Task 16).

- [ ] **Step 2: Load-marker proves the fork loaded**

Confirm the injected bootstrap context contains `superpowers-mihai-fork: v5.1.0-mihai.1` (the marker from Task 3). Since it's an HTML comment at the end of `using-superpowers`, it rides along in the injected text. If absent, the official copy loaded — recheck the swap.

- [ ] **Step 3: Errors tab + breadth**

User opens `/plugin` → Errors tab (must be clean) and confirms `superpowers:writing-plans`, `superpowers:test-driven-development`, `superpowers:requesting-code-review` resolve. Optionally confirm `@agent-superpowers:code-reviewer` is offered.

- [ ] **Step 4: Done.** Record results. Rollback if any check fails: flip the two `enabledPlugins` booleans back and restart (Tier-1).

---

## Self-Review (completed by plan author)

**Spec coverage:** §4.1 repo/remotes → T1; §4.2 naming → T3; §4.3 versioning → T3; §4.4 cutover → T15–16; commit 0 → T2; commits 1/3/4/6 → T4–7; commit 5 (gated) → T8; commit 7 (convenience-only) → T9–12; commit 2 dropped → (no task, by decision); §6 verification → T13, T17; §7 rollback → T17 Step 4; §8 maintenance → documented in spec (not a build task). All covered.

**Placeholder scan:** The `<!-- BODY: paste verbatim … -->` markers in T9–11 are *located verbatim-copy instructions with exact source files/line ranges*, not content placeholders — the source prose exists in-repo and the executing subagent copies it. `<your-gh-user>` is an intentional parameter resolved at T14. No TBD/TODO/"handle errors" placeholders.

**Type/name consistency:** marketplace `superpowers-mihai`, plugin `superpowers`, version `5.1.0-mihai.1`, marker `superpowers-mihai-fork: v5.1.0-mihai.1`, agent names `spec-reviewer`/`code-quality-reviewer`/`code-reviewer` — used consistently across tasks and the post-install checks.

**Known soft spots (intentional):** T8 depends on a user live-test and is skippable; T9–11 require the executing agent to read the named template files to copy prose; T13 Step 3 may skip tests if the harness is unavailable (must report, not fake).
