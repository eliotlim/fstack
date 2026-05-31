---
name: fstack-profile
version: 0.1.0
description: Inspect, edit, and reconcile the fstack developer profile. Shows the gap between declared and observed.
allowed-tools:
  - Bash
  - Read
  - Edit
  - AskUserQuestion
triggers:
  - show my profile
  - edit my profile
  - update my profile
  - developer vibe
  - profile gap
---

# /fstack-profile

Five dimensions at `profile.developer.*`:

- `risk_tolerance` (0 cautious, 1 bold)
- `bias_for_action` (0 plan first, 1 ship now)
- `scope_appetite` (0 MVP, 1 boil the ocean)
- `test_rigor` (0 happy path, 1 full coverage)
- `architecture_care` (0 ship now, 1 principled)

Five preferences at `profile.preferences.*`:

- `code_style` (functional, oop, mixed)
- `comment_level` (none, minimal, verbose)
- `abstraction` (low, medium, high)
- `error_handling` (try-catch, result-types, propagate)
- `type_safety` (strict, balanced, loose)

## Preamble

```bash
_FSTACK_BIN=""
_skill_md="$HOME/.claude/skills/fstack-profile/SKILL.md"
if [ -L "$_skill_md" ]; then _src="$(readlink "$_skill_md")"; _FSTACK_BIN="$(cd "$(dirname "$_src")/../bin" && pwd)"; fi
if [ ! -x "$_FSTACK_BIN/fstack-config" ]; then
  for cand in "$HOME/Workspaces/fstack/bin" "$HOME/fstack/bin"; do [ -x "$cand/fstack-config" ] && _FSTACK_BIN="$cand" && break; done
fi
"$_FSTACK_BIN/fstack-config" exists || { echo "uninitialized. run /fstack"; exit 0; }
```

## Branch

1. show / inspect
2. edit a dimension
3. edit a preference
4. show gap
5. reset

Ambiguous? Ask which.

## 1. Show

```bash
"$_FSTACK_BIN/fstack-profile" show
```

If `sample_count >= 5`:

```bash
"$_FSTACK_BIN/fstack-observe" rollup >/dev/null
"$_FSTACK_BIN/fstack-observe" gap
```

If any |gap| ≥ 0.2, surface it.

Offer: "Edit any of these? Tell me what to change."

## 2. Edit a dimension

Map phrasing to `(dim, delta)`:

- more cautious / careful → `risk_tolerance` down
- more bold / faster → `risk_tolerance` up
- plan more / deliberate → `bias_for_action` down
- ship faster → `bias_for_action` up
- smaller scope → `scope_appetite` down
- complete / boil the ocean → `scope_appetite` up
- more coverage / full tests → `test_rigor` up
- lighter tests → `test_rigor` down
- principled / design first → `architecture_care` up
- ship now / pragmatic → `architecture_care` down

Read current. Compute new (delta ±0.2, clamp 0..1, or use the user's explicit number).

Confirm via AskUserQuestion before writing.

Write:

```bash
"$_FSTACK_BIN/fstack-config" set profile.developer.<dim> <new>
"$_FSTACK_BIN/fstack-config" set profile.declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

Report: "Set `<dim>` to <new>."

## 3. Edit a preference

Same flow. Write strings, not numbers. Reject values outside the enum.

```bash
"$_FSTACK_BIN/fstack-config" set profile.preferences.<key> <value>
```

## 4. Show gap

```bash
"$_FSTACK_BIN/fstack-observe" rollup >/dev/null
"$_FSTACK_BIN/fstack-observe" gap
```

Translate per line:

- |gap| < 0.1 → "close."
- 0.1..0.3 → "drifting."
- > 0.3 → "mismatch. Either declared is stale or behavior isn't what you'd say."

Ask: update declared to observed? keep declared? leave it?

Only update what the user agrees to:

```bash
"$_FSTACK_BIN/fstack-config" set profile.developer.<dim> <observed>
"$_FSTACK_BIN/fstack-config" set profile.declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

## 5. Reset

Confirm via AskUserQuestion. If yes:

```bash
"$_FSTACK_BIN/fstack-config" set-json profile.developer '{"risk_tolerance":null,"bias_for_action":null,"scope_appetite":null,"test_rigor":null,"architecture_care":null}'
"$_FSTACK_BIN/fstack-config" set profile.declared_at null
```

Invoke `/fstack` to re-declare.

## Voice

Confirm before mutating declared. Free-form input + profile writes is a trust boundary.
