---
name: fstack
version: 0.1.0
description: Full-stack engineer's companion. Router and setup entrypoint for the fstack skill family. Profile-driven recommendations for shipping production-ready apps at scale.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - AskUserQuestion
  - Skill
triggers:
  - fstack
  - set up fstack
  - configure fstack
  - show my fstack profile
---

# /fstack

Entrypoint. Setup if needed. Otherwise route or summarize.

## Preamble

```bash
_FSTACK_BIN=""
_skill_md="$HOME/.claude/skills/fstack/SKILL.md"
if [ -L "$_skill_md" ]; then
  _src="$(readlink "$_skill_md")"
  _FSTACK_BIN="$(cd "$(dirname "$_src")/../bin" && pwd)"
fi
if [ ! -x "$_FSTACK_BIN/fstack-config" ]; then
  for cand in "$HOME/Workspaces/fstack/bin" "$HOME/fstack/bin" "$HOME/src/fstack/bin"; do
    [ -x "$cand/fstack-config" ] && _FSTACK_BIN="$cand" && break
  done
fi
echo "FSTACK_BIN: $_FSTACK_BIN"
if [ ! -x "$_FSTACK_BIN/fstack-config" ]; then
  echo "FSTACK_STATE: missing-bin"
elif "$_FSTACK_BIN/fstack-config" exists; then
  echo "FSTACK_STATE: ready"
  echo "FSTACK_VERSION: $("$_FSTACK_BIN/fstack-config" get version)"
  echo "INSTALL_MODE: $("$_FSTACK_BIN/fstack-config" get install.mode)"
  _declared="$("$_FSTACK_BIN/fstack-config" get profile.declared_at)"
  [ -z "$_declared" ] && echo "PROFILE: undeclared" || echo "PROFILE: declared"
  echo "OBSERVATIONS: $("$_FSTACK_BIN/fstack-config" get profile.inferred.sample_count)"
else
  echo "FSTACK_STATE: uninitialized"
fi
```

## Branch

1. `missing-bin`. Tell the user fstack isn't on disk. Point at the installer. STOP.
2. `uninitialized`. Tell them to run `./install --dev` from the repo root, then retry. STOP.
3. `PROFILE: undeclared`. Run **Setup** below.
4. Otherwise. Run **Route**.

## Setup

Ask five questions via AskUserQuestion, one at a time. Each maps to a 0.0..1.0 value at `profile.developer.<dim>`. A=0.2, B=0.5, C=0.8.

1. `risk_tolerance`. Check carefully / Balanced / Move fast.
2. `bias_for_action`. Plan first / Balanced / Ship now.
3. `scope_appetite`. Smallest useful / Balanced / Complete.
4. `test_rigor`. Happy path / Balanced / Full coverage.
5. `architecture_care`. Ship now / Balanced / Design first.

After each answer:

```bash
"$_FSTACK_BIN/fstack-config" set profile.developer.<dim> <value>
```

After all five:

```bash
"$_FSTACK_BIN/fstack-config" set profile.declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
"$_FSTACK_BIN/fstack-profile" vibe
```

Then: "Declare your stack now? Takes a minute." Yes → invoke `/fstack-stack`.

## Route

Match intent. Invoke via Skill tool, don't paraphrase.

| User wants | Route |
|------------|-------|
| see profile / vibe / nothing typed | inline summary below |
| edit profile / change dimension | `/fstack-profile` |
| set stack / use X | `/fstack-stack` |
| add a skill / codify a pattern | `/fstack-skill` |
| scaffold / new feature / new route | `/fstack-scaffold` |
| design an api / new endpoint | `/fstack-api` |
| new table / migration / schema | `/fstack-schema` |
| ambiguous | ask: profile, stack, custom skill, scaffold, api, or schema? |

## Inline summary

When not routing, end with:

```bash
"$_FSTACK_BIN/fstack-profile" vibe
"$_FSTACK_BIN/fstack-profile" stack
```

Then one line of next step:

- Profile declared, stack empty → "Run `/fstack-stack`."
- Both set, no observations → "Use `/fstack-scaffold`, `/fstack-api`, or `/fstack-schema`."
- 10+ observations and any gap ≥ 0.2 → "Drift on X. `/fstack-profile gap` for details."

## Observation logging

This skill doesn't log. Domain skills do. Exception: if the user states a clear preference here ("I always want full coverage"), log it:

```bash
"$_FSTACK_BIN/fstack-observe" log <dim> <signal> --skill fstack --context "user explicit"
```

Bar: would this nudge future suggestions? Yes → log. No → skip.

## Voice

Short. Direct. No throat-clearing. Numbered lists for steps.
