---
name: fstack-profile
version: 0.5.0
description: Inspect, edit, and switch fstack archetypes. Archetypes are behavior-derived (production, sprint, exploration, etc.) and reference repo/branch combinations via context_distribution.
allowed-tools:
  - Bash
  - Read
  - Edit
  - AskUserQuestion
triggers:
  - show my profile
  - edit my profile
  - update my profile
  - which archetype am I in
  - profile gap
  - switch archetype
  - new manual subprofile
  - create profile
  - list profiles
---

# /fstack-profile

Writes target the active subprofile. To affect a different one, switch first.

Seven dimensions at `developer.*`. Both ends of each spectrum are legitimate working styles; pick where the user actually lives, not where you think is "best."

- `risk_tolerance` (0 cautious, 1 bold)
- `bias_for_action` (0 deliberate, 1 action-oriented)
- `scope_appetite` (0 focused, 1 ambitious)
- `test_rigor` (0 lean, 1 rigorous)
- `architecture_care` (0 pragmatic, 1 principled)
- `detail_preference` (0 terse, 1 thorough)
- `autonomy` (0 collaborative, 1 autonomous)

Five preferences at `preferences.*`:

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
echo "ACTIVE: $("$_FSTACK_BIN/fstack-config" active)"
"$_FSTACK_BIN/fstack-profile" list
```

## Branch

1. show / inspect
2. list subprofiles
3. switch (use)
4. create
5. clone (create + copy from another)
6. rename
7. remove
8. edit a dimension (writes to active)
9. edit a preference (writes to active)
10. show gap (active)
11. reset (active)

Ambiguous? Ask which.

## 1. Show

```bash
"$_FSTACK_BIN/fstack-profile" show          # active
"$_FSTACK_BIN/fstack-profile" show <key>    # specific
```

If active has `sample_count >= 5`:

```bash
"$_FSTACK_BIN/fstack-observe" rollup >/dev/null
"$_FSTACK_BIN/fstack-observe" gap
```

Surface any |gap| ≥ 0.2.

## 2. List

```bash
"$_FSTACK_BIN/fstack-profile" list
```

Active marked with `*`.

## 3. Switch (use)

```bash
"$_FSTACK_BIN/fstack-profile" use <key>
```

Confirm in one line: "active: <key>." Show the vibe afterward.

If the user said "switch to <name>" but no subprofile by that name exists, offer to create.

## 4. Create

Ask:

1. **Key** (slug, kebab-case). Validate.
2. **Label** (free text, defaults to key).
3. **Description** (one line, optional but helpful).
4. **Walk setup now or leave unset?**

```bash
"$_FSTACK_BIN/fstack-profile" create <key> --label "<label>" --description "<desc>"
```

If "walk setup now": switch to the new key, then run /fstack's setup flow.

## 5. Clone

Use when the new profile is mostly the same as an existing one with tweaks.

```bash
"$_FSTACK_BIN/fstack-profile" create <new-key> --label "<label>" --clone-from <existing-key>
```

Clone copies developer + preferences + stack. Resets inferred + declared_at so the new profile records its own observations.

After clone, ask: "Tweak any dimensions now?" Branch into 8 if yes.

## 6. Rename

```bash
"$_FSTACK_BIN/fstack-profile" rename <old> <new>
```

Confirm via AskUserQuestion before mutating.

## 7. Remove

```bash
"$_FSTACK_BIN/fstack-profile" remove <key>
```

Cannot remove the last subprofile (binary enforces). Confirm via AskUserQuestion. If removed and was active, active falls back to the first remaining key.

## 8. Edit a dimension (writes to active)

Map phrasing. Both directions are normal moves; don't assume "up" is improvement.

- more cautious / measured → `risk_tolerance` down
- more bold / experimental → `risk_tolerance` up
- more deliberate / plan more → `bias_for_action` down
- more action / ship more → `bias_for_action` up
- more focused / smaller scope → `scope_appetite` down
- more ambitious / boil-the-ocean → `scope_appetite` up
- more rigorous / thorough tests → `test_rigor` up
- leaner tests → `test_rigor` down
- more principled / structured → `architecture_care` up
- more pragmatic / direct → `architecture_care` down
- terser → `detail_preference` down
- more thorough explanations → `detail_preference` up
- more collaborative / consult more → `autonomy` down
- more autonomous / delegate more → `autonomy` up

Read current. Compute new (delta ±0.2, clamp 0..1, or use explicit number).

Confirm via AskUserQuestion. Then:

```bash
"$_FSTACK_BIN/fstack-config" set-active developer.<dim> <new>
"$_FSTACK_BIN/fstack-config" set-active declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

If the user phrasing implies the change is mode-specific ("for prod I'm careful"), ask whether to apply to active or create a new subprofile.

## 9. Edit a preference (writes to active)

```bash
"$_FSTACK_BIN/fstack-config" set-active preferences.<key> <value>
```

Reject values outside the enum.

## 10. Show gap (active)

```bash
"$_FSTACK_BIN/fstack-observe" rollup >/dev/null
"$_FSTACK_BIN/fstack-observe" gap
```

Translate per line:

- |gap| < 0.1: "close."
- 0.1..0.3: "drifting."
- > 0.3: "mismatch. Either declared is stale or behavior isn't what you'd say."

Ask: update declared to observed? keep declared? leave as-is? Apply only what the user agrees to:

```bash
"$_FSTACK_BIN/fstack-config" set-active developer.<dim> <observed>
"$_FSTACK_BIN/fstack-config" set-active declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

## 11. Reset (active)

Confirm. If yes:

```bash
"$_FSTACK_BIN/fstack-config" set-active-json developer '{"risk_tolerance":null,"bias_for_action":null,"scope_appetite":null,"test_rigor":null,"architecture_care":null,"detail_preference":null,"autonomy":null}'
"$_FSTACK_BIN/fstack-config" set-active declared_at null
```

Invoke `/fstack` to re-declare.

## Voice

Confirm before mutating declared values. Writes default to **active** — the user has to switch first to edit a different subprofile.
