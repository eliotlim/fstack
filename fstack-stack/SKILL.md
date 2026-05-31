---
name: fstack-stack
version: 0.1.0
description: Declare and inspect the user's tech stack preferences. Other fstack skills read this to shape recommendations.
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
---

# /fstack-stack

Stack lives at `stack.*`. Each category is an array.

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
```

## Branch

1. show
2. set (replace category)
3. add to category
4. remove from category
5. full setup (all categories empty and no `declared_at`)

## 1. Show

```bash
"$_FSTACK_BIN/fstack-profile" stack
```

Empty? "No stack declared. Run `/fstack-stack`, or just tell me what you use."

## 2. Set

```bash
"$_FSTACK_BIN/fstack-config" set-json stack.<category> '["<value>"]'
"$_FSTACK_BIN/fstack-config" set stack.declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

Confirm in one line. "Frontend → Svelte."

## 3. Add

```bash
_current="$("$_FSTACK_BIN/fstack-config" get stack.<category>)"
[ -z "$_current" ] && _current='[]'
_new="$(jq -c --arg v "<value>" '. + [$v] | unique' <<<"$_current")"
"$_FSTACK_BIN/fstack-config" set-json stack.<category> "$_new"
```

## 4. Remove

```bash
_current="$("$_FSTACK_BIN/fstack-config" get stack.<category>)"
_new="$(jq -c --arg v "<value>" '. - [$v]' <<<"$_current")"
"$_FSTACK_BIN/fstack-config" set-json stack.<category> "$_new"
```

## 5. Full setup

Walk 10 categories via AskUserQuestion. 3-4 picks each. AskUserQuestion adds Other automatically.

Tilt recommendations to the declared profile:

- `bias_for_action ≥ 0.65`: batteries-included (Next.js, Vercel, Clerk, Drizzle).
- `architecture_care ≥ 0.65`: explicit and typed (Hono + Kysely, Fastify + raw, Drizzle over Prisma).
- `risk_tolerance ≤ 0.35`: boring (Postgres, Node, React).
- `risk_tolerance ≥ 0.65`: newer is fair game (Bun, Effect, latest frameworks).

After each pick:

```bash
"$_FSTACK_BIN/fstack-config" set-json stack.<category> "$(jq -c --arg v "<value>" '[$v]')"
```

After all 10:

```bash
"$_FSTACK_BIN/fstack-config" set stack.declared_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
"$_FSTACK_BIN/fstack-profile" stack
```

## Observation

User picked Other = strong opinion. Tiny bias_for_action nudge:

```bash
"$_FSTACK_BIN/fstack-observe" log bias_for_action 0.6 --weight 1 --skill fstack-stack --context "Other for <category>"
```

## Voice

Categories are nudges. Custom picks land without protest. No tradeoff lectures unless asked.
