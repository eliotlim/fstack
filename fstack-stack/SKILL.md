---
name: fstack-stack
version: 0.6.0
description: Declare the tech stack for the active subprofile. Different modes can hold different stacks (acme-prod = Postgres/Prisma, late-night = SQLite/Drizzle).
allowed-tools:
  - Bash
  - Read
  - Edit
  - AskUserQuestion
triggers:
  - set my stack
  - configure stack
  - tech stack
  - use postgres
  - use clerk
  - this project uses X
---

# /fstack-stack

Stack lives at `stack.*` of the **active subprofile**. Each category is an array. Switch profiles via `/fstack-profile use <key>` before declaring a different mode's stack.

| Key | Common values |
|-----|---------------|
| `language` | TypeScript, JavaScript, Python, Go, Rust, Ruby, Java, Kotlin |
| `runtime` | Node.js, Bun, Deno, Python, Go, JVM |
| `frontend` | React, Next.js, Vue, Nuxt, Svelte, SvelteKit, Solid, None |
| `ui` | Tailwind, shadcn/ui, Mantine, MUI, Chakra, CSS Modules |
| `backend_framework` | Hono, Express, Fastify, Next API, FastAPI, Django, Rails, Gin, Axum |
| `database` | Postgres, MySQL, SQLite, MongoDB, DynamoDB, Planetscale, Neon, Supabase |
| `orm` | Drizzle, Prisma, Kysely, SQLAlchemy, TypeORM, raw SQL |
| `auth` | Clerk, Auth.js, Supabase Auth, Lucia, Better-Auth, custom |
| `deploy` | Vercel, Netlify, Railway, Fly.io, Render, AWS, Cloudflare, Bare-metal |
| `testing` | Vitest, Jest, Playwright, Cypress, Pytest, Go test |

Values are nudges. User can put anything.

## Preamble

```bash
_FSTACK_BIN=""
_skill_md="$HOME/.claude/skills/fstack-stack/SKILL.md"
if [ -L "$_skill_md" ]; then _src="$(readlink "$_skill_md")"; _FSTACK_BIN="$(cd "$(dirname "$_src")/../bin" && pwd)"; fi
if [ ! -x "$_FSTACK_BIN/fstack-config" ]; then
  for cand in "$HOME/Workspaces/fstack/bin" "$HOME/fstack/bin"; do [ -x "$cand/fstack-config" ] && _FSTACK_BIN="$cand" && break; done
fi
"$_FSTACK_BIN/fstack-config" exists || { echo "uninitialized. run /fstack"; exit 0; }
"$_FSTACK_BIN/fstack-profile" suggest
"$_FSTACK_BIN/fstack-profile" calibrate stack
```

State the active subprofile up front. If the user's intent reads like a different mode ("for prod I use X"), offer to switch or create first. If `SUGGEST:` printed, surface it before proceeding.

## Branch

1. show
2. set (replace category)
3. add to category
4. remove from category
5. full setup (all categories empty on active and no `stack.declared_at`)

## 1. Show

```bash
"$_FSTACK_BIN/fstack-profile" stack
```

Empty? "No stack on `$ACTIVE`. Run `/fstack-stack`, or tell me what you use."

## 2. Set

```bash
"$_FSTACK_BIN/fstack-config" set-active-json stack.<category> '["<value>"]'
"$_FSTACK_BIN/fstack-config" set-active stack.declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

Confirm in one line. "Frontend → Svelte (on `$ACTIVE`)."

## 3. Add

```bash
_current="$("$_FSTACK_BIN/fstack-config" get-active stack.<category>)"
[ -z "$_current" ] && _current='[]'
_new="$(jq -c --arg v "<value>" '. + [$v] | unique' <<<"$_current")"
"$_FSTACK_BIN/fstack-config" set-active-json stack.<category> "$_new"
```

## 4. Remove

```bash
_current="$("$_FSTACK_BIN/fstack-config" get-active stack.<category>)"
_new="$(jq -c --arg v "<value>" '. - [$v]' <<<"$_current")"
"$_FSTACK_BIN/fstack-config" set-active-json stack.<category> "$_new"
```

## 5. Full setup

Walk 10 categories via AskUserQuestion. 3-4 picks each.

Tilt recommendations from the active subprofile's declared dimensions:

- `bias_for_action ≥ 0.65` (action): batteries-included (Next.js, Vercel, Clerk, Drizzle).
- `architecture_care ≥ 0.65` (principled): explicit + typed (Hono + Kysely, Fastify + raw, Drizzle over Prisma).
- `risk_tolerance ≤ 0.35` (stability): boring (Postgres, Node, React).
- `risk_tolerance ≥ 0.65` (speed): newer fair game (Bun, Effect, latest frameworks).
- `detail_preference ≤ 0.35` (detail-oriented): pin versions; specify storage, cache, queue choices; surface `engines` field.
- `detail_preference ≥ 0.65` (big-picture): use latest; trust ecosystem defaults; no version pinning.
- `autonomy ≤ 0.35` (seek-permission): walk every category individually via AskUserQuestion.
- `autonomy ≥ 0.65` (ask-forgiveness): propose the full stack in one block; user removes lines they don't want.

After each pick:

```bash
"$_FSTACK_BIN/fstack-config" set-active-json stack.<category> "$(jq -c --arg v "<value>" '[$v]')"
```

After all 10:

```bash
"$_FSTACK_BIN/fstack-config" set-active stack.declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
"$_FSTACK_BIN/fstack-profile" stack
```

## Observation

User picked Other = strong opinion. Small `bias_for_action` nudge on active:

```bash
"$_FSTACK_BIN/fstack-observe" log bias_for_action 0.6 --weight 1 --skill fstack-stack --context "Other for <category>"
```

## Voice

Always print which subprofile you're writing to. The user has multiple modes; ambiguity is the failure mode.
