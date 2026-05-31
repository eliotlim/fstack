---
name: fstack-api
version: 0.1.0
description: Design and stub a new API endpoint. Detects style (REST, tRPC, GraphQL, Hono, FastAPI). Calibrates validation to risk tolerance and architecture care.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - AskUserQuestion
triggers:
  - new endpoint
  - design an api
  - new api route
  - graphql query
  - trpc procedure
  - rest endpoint
---

# /fstack-api

1. Get the endpoint summary.
2. Detect API style in the repo.
3. Calibrate from profile.
4. Propose.
5. Write. Log.

## Preamble

```bash
_FSTACK_BIN=""
_skill_md="$HOME/.claude/skills/fstack-api/SKILL.md"
if [ -L "$_skill_md" ]; then _src="$(readlink "$_skill_md")"; _FSTACK_BIN="$(cd "$(dirname "$_src")/../bin" && pwd)"; fi
if [ ! -x "$_FSTACK_BIN/fstack-config" ]; then
  for cand in "$HOME/Workspaces/fstack/bin" "$HOME/fstack/bin"; do [ -x "$cand/fstack-config" ] && _FSTACK_BIN="$cand" && break; done
fi
"$_FSTACK_BIN/fstack-config" exists || { echo "uninitialized. run /fstack"; exit 0; }
echo "PROFILE:"
"$_FSTACK_BIN/fstack-profile" dimensions
echo "STACK:"
"$_FSTACK_BIN/fstack-config" get stack.backend_framework
"$_FSTACK_BIN/fstack-config" get stack.language
```

## 1. Endpoint summary

User hasn't said? Ask for one sentence: method, resource, intent. Example: `POST /api/teams/:id/invite` (send an email invite).

## 2. Detect style

First match wins:

| Signal | Style |
|--------|-------|
| `trpc` in deps + `appRouter.ts` / `routers/*` | tRPC |
| `graphql` / `@apollo` / `schema.graphql` / `pothos` | GraphQL |
| `app/api/` or `pages/api/` | Next.js Route Handler |
| `hono` + `app.get(` / `app.post(` | Hono |
| `express` + `app.get(` | Express |
| `fastify` | Fastify |
| `@fastapi` decorators in Python | FastAPI |
| Django `urls.py` + views | Django |
| Rails `routes.rb` | Rails |
| `.proto` files | gRPC |

None? Fall back to declared `stack.backend_framework`. Empty? Ask.

## 3. Calibrate

| Dim | Effect |
|-----|--------|
| `architecture_care` low | inline validation |
| `architecture_care` mid | adjacent validator file |
| `architecture_care` high | separate schema (zod/valibot/pydantic), input + output types, service-layer call |
| `risk_tolerance` low | explicit types everywhere, 4xx on every missing key |
| `risk_tolerance` high | trust callers, validate at boundaries |
| `test_rigor` low | skip tests |
| `test_rigor` mid | smoke (happy + one 4xx) |
| `test_rigor` high | smoke + auth-failure + edges + integration |

Empty dim = 0.5.

## 4. Propose

```
Endpoint: <METHOD> <path>
Style: <style>
Files:
  + <handler>
  + <schema>
  + <test>
  ~ <router> (register)
Inputs: <fields + types>
Output: <shape>
Errors: <4xx list>
Auth: <required|optional|none>
```

AskUserQuestion: apply / change / cancel.

## 5. Write

Read 1-2 existing handlers. Mirror them.

**tRPC:**

```ts
export const <resource>Router = router({
  <action>: publicProcedure
    .input(z.object({ ... }))
    .mutation(async ({ input, ctx }) => {
      // TODO
      return { ok: true }
    }),
})
```

**Hono:**

```ts
app.post('<path>', zValidator('json', InputSchema), async (c) => {
  const body = c.req.valid('json')
  // TODO
  return c.json({ ok: true })
})
```

**Next.js Route Handler:**

```ts
export async function POST(req: NextRequest) {
  const parsed = InputSchema.safeParse(await req.json())
  if (!parsed.success) return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 })
  // TODO
  return NextResponse.json({ ok: true })
}
```

**FastAPI:**

```python
@router.post("<path>")
async def <action>(payload: <Schema>) -> <Output>:
    # TODO
    return <Output>(ok=True)
```

Use the project's existing error shape. Don't invent new ones.

## 6. Log

```bash
# validation: inline=0.2, extracted=0.5, schema+types=0.85
"$_FSTACK_BIN/fstack-observe" log architecture_care <signal> --skill fstack-api --context "<pattern>"
"$_FSTACK_BIN/fstack-observe" log test_rigor <signal> --skill fstack-api --context "<tests>"
[ "$tests_dropped" = "yes" ] && "$_FSTACK_BIN/fstack-observe" log test_rigor 0.2 --skill fstack-api --context "user dropped tests"
```

## 7. Report

```
✓ <METHOD> <path> stubbed (<style>)
  Files: <list>
  Validation: <inline|extracted|schema>
  Tests: <none|smoke|full>
  TODO: <one line>
```

## Voice

User knows REST. Don't explain it. Repo pattern beats declared stack.
