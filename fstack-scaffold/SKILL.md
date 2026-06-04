---
name: fstack-scaffold
version: 0.6.0
description: Scaffold a new feature, route, page, or component in line with stack and profile. Logs observations.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - AskUserQuestion
triggers:
  - scaffold a feature
  - new route
  - new page
  - start a new feature
  - new component
  - new module
---

# /fstack-scaffold

1. Figure out what's being scaffolded.
2. Read stack + profile.
3. Read the repo (existing patterns win over declared stack).
4. Propose file list before writing.
5. Write. Log observations.

## Preamble

```bash
_FSTACK_BIN=""
_skill_md="$HOME/.claude/skills/fstack-scaffold/SKILL.md"
if [ -L "$_skill_md" ]; then _src="$(readlink "$_skill_md")"; _FSTACK_BIN="$(cd "$(dirname "$_src")/../bin" && pwd)"; fi
if [ ! -x "$_FSTACK_BIN/fstack-config" ]; then
  for cand in "$HOME/Workspaces/fstack/bin" "$HOME/fstack/bin"; do [ -x "$cand/fstack-config" ] && _FSTACK_BIN="$cand" && break; done
fi
"$_FSTACK_BIN/fstack-config" exists || { echo "uninitialized. run /fstack"; exit 0; }
"$_FSTACK_BIN/fstack-profile" suggest
"$_FSTACK_BIN/fstack-profile" calibrate scaffold
echo "STACK:"
"$_FSTACK_BIN/fstack-profile" stack
```

If `SUGGEST:` printed, surface it in one short line before proceeding. Don't block.

All dimensions unset? Warn once: "Profile undeclared. Using balanced defaults. Declare via /fstack-profile or just keep working; clustering will learn." Continue.

## 1. What's the shape?

Ask only if unclear:

1. Page / route
2. Component (reusable UI)
3. Feature module (page + data + api)
4. Backend module (service, worker, job)

## 2. Read the repo

Glob/Grep for:

- `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`
- `app/`, `pages/`, `src/`, `routes/`
- `api/`, `server/`, `backend/`
- 1-2 recently-modified files in the target area

Existing patterns beat declared stack. If they disagree (declared React, repo Vue), ask once: use repo or migrate? Almost always: use repo.

## 3. Calibrate (visible)

Re-print the calibration block, then map.

| Dim | Low end | Balanced | High end |
|-----|---------|----------|----------|
| `scope_appetite` (focused → ambitious) | 1 file | folder + index | folder + sub-routes + tests + types + README |
| `test_rigor` (lean → rigorous) | none | smoke | unit + integration + e2e |
| `architecture_care` (pragmatic → principled) | inline | clean exports | interfaces + DI + types-first |
| `bias_for_action` (deliberate → action) | skip TODOs | TODO at boundaries | TODO every edge with acceptance criteria |
| `risk_tolerance` (stability → speed) | battle-tested | modern + supported | newer patterns (server components, signals) |
| `detail_preference` (detail-oriented → big-picture) | inline JSDoc, prop tables, usage snippet in README | minimal types | no comments, no README, code-only |
| `autonomy` (seek-permission → ask-forgiveness) | propose file list + confirm + write | propose + write | write + show diff |

Conflict (focused scope + rigorous tests) → scope wins. Coverage on small surface beats partial on sprawl.

## 4. Propose

```
Plan: <one line>
Files:
  + <path>: <one line>
Edits:
  ~ <path>: <one line>
Depth: <MVP|balanced|complete> (scope_appetite=<v>)
Tests: <none|smoke|full> (test_rigor=<v>)
Calibration: <archetype> · <dim>=<band> · ...
```

If `autonomy` reads as seek-permission: AskUserQuestion (apply / change / cancel).
If `autonomy` reads as ask-forgiveness: write + show diff. User redirects after.
Default: AskUserQuestion.

## 5. Write

Mirror the repo's voice:

- Comment density: read 1-2 nearby files, match.
- Import order: copy.
- Naming (kebab vs camel): copy.
- TS strictness: read `tsconfig.json`.

Run any safety check already wired (`tsc --noEmit`, `bun test`, `pytest -q`). Don't add tooling.

## 6. Log

```bash
# files written: 1=0.2, 2-3=0.4, 4-6=0.6, 7+=0.8
"$_FSTACK_BIN/fstack-observe" log scope_appetite <signal> --skill fstack-scaffold \
  --annotation "Scaffolded <feature> at <path>, <N files> + <test-style>"
# tests: none=0.2, smoke=0.5, full=0.85
"$_FSTACK_BIN/fstack-observe" log test_rigor <signal> --skill fstack-scaffold \
  --annotation "<tests outcome>"
[ "$user_shrank" = "yes" ] && "$_FSTACK_BIN/fstack-observe" log scope_appetite 0.25 \
  --skill fstack-scaffold --annotation "User shrank scaffold plan"

# detail_preference: code only = 0.8, JSDoc + README + types = 0.2
"$_FSTACK_BIN/fstack-observe" log detail_preference <signal> --skill fstack-scaffold \
  --annotation "<docs depth>"

# autonomy: confirmed plan = 0.2, wrote + diffed = 0.8
"$_FSTACK_BIN/fstack-observe" log autonomy <signal> --skill fstack-scaffold \
  --annotation "<confirmed | autonomous>"
```

Good annotations: "Scaffolded teams page with smoke test", "Built admin route, no tests requested", "Quick prototype, single file", "Wrote without plan-confirm, just diff".

Bad annotations: "did the work", "logged signal", "scaffolding done".

## 7. Report

```
✓ Scaffolded <feature> at <path>
  Files: <N created>, <M edited>
  Tests: <none|smoke|full>
  Calibration: <archetype> · <dim>=<band> · ...
  Next: <one line>
```

## Voice

Lead with the file list. Match the repo, not fstack defaults. Redirects land without explanation. Show calibration so the user can see why the proposed depth is what it is.
