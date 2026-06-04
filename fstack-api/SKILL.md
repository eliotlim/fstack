---
name: fstack-api
version: 0.6.0
description: Design and stub a new API endpoint. Detects style (REST, tRPC, GraphQL, Hono, FastAPI). Calibrates validation to the active subprofile.
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
3. Calibrate from profile (visible).
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
"$_FSTACK_BIN/fstack-profile" suggest
"$_FSTACK_BIN/fstack-profile" calibrate api
echo "STACK:"
"$_FSTACK_BIN/fstack-config" get-active stack.backend_framework
"$_FSTACK_BIN/fstack-config" get-active stack.language
```

If `SUGGEST:` printed, surface it in one short line before proceeding: "Heads up: this looks like `<match>` mode. Continue with `<active>` or switch?" Don't block on AskUserQuestion; the user can ignore.

The `CALIBRATION` block prints the active subprofile name and the dim values driving the proposal. Re-print on the proposal so the user sees how it shaped the plan.

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

## 3. Calibrate (visible)

Print the calibration block, then map each dim to the proposal.

```
CALIBRATION (production):
  risk_tolerance       0.20  (stability)
  architecture_care    0.85  (principled)
  test_rigor           0.80  (rigorous)
```

| Dim | Effect |
|-----|--------|
| `architecture_care` pragmatic | inline validation |
| `architecture_care` balanced | adjacent validator file |
| `architecture_care` principled | separate schema (zod / valibot / pydantic), input + output types, service-layer call |
| `risk_tolerance` stability | explicit types everywhere, 4xx on every missing key |
| `risk_tolerance` speed | trust callers, validate at boundaries |
| `test_rigor` lean | skip tests |
| `test_rigor` balanced | smoke (happy + one 4xx) |
| `test_rigor` rigorous | smoke + auth-failure + edges + integration |
| `detail_preference` detail-oriented | field-level error responses, error codes, named status enums |
| `detail_preference` big-picture | single 400 with `flatten()`, generic error shape |
| `autonomy` seek-permission | AskUserQuestion at every decision point (auth required? validation level? error shape?) before writing |
| `autonomy` ask-forgiveness | decide based on profile + repo patterns, write, report |

Unset dim is treated as balanced.

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
Calibration: <one-line restatement of the dim values applied>
```

If `autonomy` reads as seek-permission, AskUserQuestion: apply / change / cancel.
If `autonomy` reads as ask-forgiveness, write and report; the user can amend after.
Default (unset / balanced): AskUserQuestion.

## 5. Write

Read 1-2 existing handlers. Mirror them.

**tRPC:**

```ts
export const <resource>Router = router({
  <action>: publicProcedure
    .input(z.object({ ... }))
    .mutation(async ({ input, ctx }) => {
      return { ok: true }
    }),
})
```

**Hono:**

```ts
app.post('<path>', zValidator('json', InputSchema), async (c) => {
  const body = c.req.valid('json')
  return c.json({ ok: true })
})
```

**Next.js Route Handler:**

```ts
export async function POST(req: NextRequest) {
  const parsed = InputSchema.safeParse(await req.json())
  if (!parsed.success) return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 })
  return NextResponse.json({ ok: true })
}
```

**FastAPI:**

```python
@router.post("<path>")
async def <action>(payload: <Schema>) -> <Output>:
    return <Output>(ok=True)
```

Use the project's existing error shape. Don't invent new ones.

## 6. Log

Annotations are short, concrete, human-readable. They feed clustering.

```bash
# validation: inline=0.2, extracted=0.5, schema+types=0.85
"$_FSTACK_BIN/fstack-observe" log architecture_care <signal> --skill fstack-api \
  --annotation "<METHOD> <path> in <style>, <validation kind>"
"$_FSTACK_BIN/fstack-observe" log test_rigor <signal> --skill fstack-api \
  --annotation "<test style>"
[ "$tests_dropped" = "yes" ] && "$_FSTACK_BIN/fstack-observe" log test_rigor 0.2 \
  --skill fstack-api --annotation "User dropped tests on <path>"

# detail_preference: generic 400=0.2, named statuses + per-field errors=0.85
"$_FSTACK_BIN/fstack-observe" log detail_preference <signal> --skill fstack-api \
  --annotation "<error shape used>"

# autonomy: pre-write confirmation=0.2, wrote+reported=0.8
# Log on the choice that actually happened, not the configured value.
"$_FSTACK_BIN/fstack-observe" log autonomy <signal> --skill fstack-api \
  --annotation "<asked|wrote-then-reported>"
```

Good annotations: "POST /teams/invite in tRPC, separate schema + types", "Quick GET /me handler, no validation", "Per-field error codes for /signup", "Wrote /me handler without confirm".

## 7. Report

```
✓ <METHOD> <path> stubbed (<style>)
  Files: <list>
  Validation: <inline|extracted|schema>
  Tests: <none|smoke|full>
  Calibration: <archetype> · <dim-1>=<band> · <dim-2>=<band> · <dim-3>=<band>
  TODO: <one line>
```

## Voice

User knows REST. Don't explain it. Repo pattern beats declared stack. Always show calibration so the user can see how the active subprofile shaped the response.
