---
name: fstack
version: 0.3.0
description: Full-stack engineer's companion. Router and setup entrypoint. Auto-detects which mode you're in (work prod vs side project vs late-night) by clustering your observations.
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
  - which mode am I in
  - switch profile
---

# /fstack

Entrypoint. Setup on first use. Otherwise: auto-detect mode, route, summarize.

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
  "$_FSTACK_BIN/fstack-config" migrate >/dev/null 2>&1 || true
  echo "FSTACK_STATE: ready"

  # Auto-cluster if stale (older than 1 day) and there's enough new signal
  _last="$("$_FSTACK_BIN/fstack-config" get profile.last_cluster_at)"
  _obs="$("$_FSTACK_BIN/fstack-observe" path 2>/dev/null)"
  _n_obs=0; [ -f "$_obs" ] && _n_obs="$(wc -l < "$_obs" | tr -d ' ')"
  _do_cluster=no
  if [ "$_n_obs" -ge 5 ]; then
    if [ -z "$_last" ] || [ "$_last" = "null" ]; then
      _do_cluster=yes
    else
      _last_epoch=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$_last" +%s 2>/dev/null || date -d "$_last" +%s 2>/dev/null || echo 0)
      _now_epoch=$(date +%s)
      _age_h=$(( (_now_epoch - _last_epoch) / 3600 ))
      [ "$_age_h" -gt 24 ] && _do_cluster=yes
    fi
  fi
  [ "$_do_cluster" = "yes" ] && "$_FSTACK_BIN/fstack-cluster" cluster >/dev/null 2>&1 || true

  # Auto-switch if enabled and not pinned
  _auto="$("$_FSTACK_BIN/fstack-config" get profile.auto_switch)"
  _pinned="$("$_FSTACK_BIN/fstack-config" get profile.pinned)"
  _active="$("$_FSTACK_BIN/fstack-config" active)"
  echo "ACTIVE: $_active"
  echo "AUTO_SWITCH: $_auto"
  [ -n "$_pinned" ] && [ "$_pinned" != "null" ] && echo "PINNED: $_pinned"
  if [ "$_auto" = "true" ] && { [ -z "$_pinned" ] || [ "$_pinned" = "null" ]; }; then
    _match="$("$_FSTACK_BIN/fstack-cluster" match 2>/dev/null)"
    if [ -n "$_match" ] && [ "$_match" != "$_active" ]; then
      "$_FSTACK_BIN/fstack-profile" use "$_match" >/dev/null
      echo "SWITCHED: $_active -> $_match (context match)"
      _active="$_match"
    fi
  fi

  echo "INSTALL_MODE: $("$_FSTACK_BIN/fstack-config" get install.mode)"
  "$_FSTACK_BIN/fstack-profile" list | sed 's/^/  /'
  _declared="$("$_FSTACK_BIN/fstack-config" get-active declared_at)"
  [ -z "$_declared" ] && echo "DECLARED: no" || echo "DECLARED: yes"
  echo "OBSERVATIONS_TOTAL: $_n_obs"
else
  echo "FSTACK_STATE: uninitialized"
fi
```

## Branch

1. `missing-bin`. Tell the user fstack isn't on disk. Point at the installer. STOP.
2. `uninitialized`. Run `./install --dev` from the repo root, then retry. STOP.
3. `DECLARED: no` and only one subprofile (default). Run **Setup**.
4. `SWITCHED:` line printed. Mention the switch in one short line, then route.
5. Otherwise. **Route** or **Summarize**.

## Setup

Setup writes into the **active** subprofile (always `default` on first run).

Five AskUserQuestion calls, one at a time. A=0.2, B=0.5, C=0.8.

1. `risk_tolerance`. Check carefully / Balanced / Move fast.
2. `bias_for_action`. Plan first / Balanced / Ship now.
3. `scope_appetite`. Smallest useful / Balanced / Complete.
4. `test_rigor`. Happy path / Balanced / Full coverage.
5. `architecture_care`. Ship now / Balanced / Design first.

After each:

```bash
"$_FSTACK_BIN/fstack-config" set-active developer.<dim> <value>
```

After all five:

```bash
"$_FSTACK_BIN/fstack-config" set-active declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
"$_FSTACK_BIN/fstack-profile" vibe
```

Then: "Declare your stack? Takes a minute." Yes → invoke `/fstack-stack`.

Also: "As you use fstack, observations get clustered into auto-detected modes (work vs side-project, day vs late-night). I'll switch to the right one based on context. Disable with `/fstack-profile unpin off`."

## Route

Match intent. Invoke via Skill tool.

| User wants | Route |
|------------|-------|
| see profile / nothing typed | inline summary below |
| edit dimension or preference | `/fstack-profile` |
| list / switch / pin profile | `/fstack-profile` |
| set stack / use X | `/fstack-stack` |
| add a skill | `/fstack-skill` |
| scaffold / new feature | `/fstack-scaffold` |
| design an api / endpoint | `/fstack-api` |
| new table / migration | `/fstack-schema` |
| ambiguous | ask: profile, stack, custom skill, scaffold, api, or schema? |

## Inline summary

```bash
"$_FSTACK_BIN/fstack-profile" vibe
"$_FSTACK_BIN/fstack-profile" stack
```

Then one line of next step:

- Only `default` and few observations → "Just go build. I'll learn your modes as you go."
- Multiple inferred subprofiles + just switched → "I switched you to `<key>` because you're in `<repo>` at this hour."
- 20+ observations and no inferred subprofiles emerged → "Your sessions look uniform so far — no distinct modes yet."
- Inferred subprofile active, ≥ 0.2 gap on any dim → "Your behavior in this mode is drifting from declared. `/fstack-profile gap`."
- Otherwise → "Use `/fstack-scaffold`, `/fstack-api`, or `/fstack-schema`."

## Observation logging

This skill doesn't log dimension signals. It can drop an annotation if the user states a behavioral preference in chat:

```bash
"$_FSTACK_BIN/fstack-observe" annotate "User wants full coverage on all backend changes" --skill fstack
```

Domain skills do the dimension logging.

## Voice

Short. Direct. When you auto-switch, say it in one line ("Switched to `<key>` because you're in `<repo>` after hours"). Don't apologize for switching; the user opted in by default.
