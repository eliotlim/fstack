---
name: fstack-skill
version: 0.6.0
description: Create, list, edit, and remove the user's own custom fstack skills. Codify personal patterns into slash commands.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - AskUserQuestion
triggers:
  - add a skill
  - make me a skill
  - I always do X
  - codify this
  - list my skills
---

# /fstack-skill

Custom skills live at `~/.fstack/skills/<name>/SKILL.md`, symlinked into `~/.claude/skills/<name>/SKILL.md`. Registered in `config.custom_skills`.

User owns the content. fstack does the wiring.

## Preamble

```bash
_FSTACK_BIN=""
_skill_md="$HOME/.claude/skills/fstack-skill/SKILL.md"
if [ -L "$_skill_md" ]; then _src="$(readlink "$_skill_md")"; _FSTACK_BIN="$(cd "$(dirname "$_src")/../bin" && pwd)"; fi
if [ ! -x "$_FSTACK_BIN/fstack-config" ]; then
  for cand in "$HOME/Workspaces/fstack/bin" "$HOME/fstack/bin"; do [ -x "$cand/fstack-config" ] && _FSTACK_BIN="$cand" && break; done
fi
"$_FSTACK_BIN/fstack-config" exists || { echo "uninitialized. run /fstack"; exit 0; }
"$_FSTACK_BIN/fstack-profile" calibrate skill
_CUSTOM_DIR="$HOME/.fstack/skills"
_INSTALL_DIR="$("$_FSTACK_BIN/fstack-config" get install.install_path)"
[ -z "$_INSTALL_DIR" ] && _INSTALL_DIR="$HOME/.claude/skills"
mkdir -p "$_CUSTOM_DIR"
"$_FSTACK_BIN/fstack-config" get custom_skills | jq -r 'length as $n | "CUSTOM_SKILLS: \($n)"'
```

## Branch

1. list
2. create
3. show
4. edit
5. remove

Ambiguous? List first, then ask.

## Calibrate

| Dim | Effect |
|-----|--------|
| `detail_preference` detail-oriented | Use the Workflow template (numbered branches, explicit AskUserQuestion gates); fill triggers list; add an `Examples` section |
| `detail_preference` big-picture | Use the Checklist or Knowledge template; minimal frontmatter; no examples block |
| `autonomy` seek-permission | Three questions max (slug, description, style); confirm slug; confirm before symlink |
| `autonomy` ask-forgiveness | Derive slug from description, pick Checklist as default style, create + symlink, report path. User edits after |

## 1. List

```bash
"$_FSTACK_BIN/fstack-config" get custom_skills | jq -r '
  if length == 0 then "no custom skills yet"
  else .[] | "/\(.name): \(.description)" end
'
```

Empty? "Try `/fstack-skill make a skill that runs my deploy checklist`."

## 2. Create

Three questions max. Use the user's first message as seed.

1. **Slug.** Derive kebab-case from the description. AskUserQuestion: accept derived or get one from user. Validate: lowercase, alphanumeric + dash, ≥ 3 chars, no `fstack-`/`gstack-` prefix.
2. **Description.** One line. Required for frontmatter.
3. **Style.**
   - A) Checklist (Claude reads + executes steps)
   - B) Workflow (branches, AskUserQuestion gates)
   - C) Knowledge (pure reference)
   - D) Stub (you write it)

Generate SKILL.md from template (below). Write to `~/.fstack/skills/<slug>/SKILL.md`. Symlink:

```bash
mkdir -p "$_INSTALL_DIR/<slug>"
ln -sf "$_CUSTOM_DIR/<slug>/SKILL.md" "$_INSTALL_DIR/<slug>/SKILL.md"
```

Register:

```bash
_entry=$(jq -nc \
  --arg name "<slug>" --arg desc "<description>" \
  --arg path "$_CUSTOM_DIR/<slug>/SKILL.md" \
  --arg style "<style>" \
  --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  '{name:$name, description:$desc, path:$path, style:$style, created_at:$ts}')
_existing=$("$_FSTACK_BIN/fstack-config" get custom_skills)
[ -z "$_existing" ] && _existing='[]'
_new=$(jq -c --argjson e "$_entry" '. + [$e]' <<<"$_existing")
"$_FSTACK_BIN/fstack-config" set-json custom_skills "$_new"
```

Report: "Created /<slug> at <path>. New session picks it up."

Stub style: open in `$EDITOR` if set, else print the path.

Creating a custom skill is a high `bias_for_action` signal:

```bash
"$_FSTACK_BIN/fstack-observe" log bias_for_action 0.75 --weight 1 --skill fstack-skill --annotation "created custom skill: <slug>"

# Confirmed each step = 0.2, autopiloted = 0.8
"$_FSTACK_BIN/fstack-observe" log autonomy <signal> --skill fstack-skill \
  --annotation "<confirmed each step | autonomous>"

# Workflow template = 0.2, Checklist = 0.5, Knowledge or Stub = 0.7
"$_FSTACK_BIN/fstack-observe" log detail_preference <signal> --skill fstack-skill \
  --annotation "<style> template"
```

## 3. Show

```bash
_path=$("$_FSTACK_BIN/fstack-config" get custom_skills | jq -r --arg n "<name>" '.[] | select(.name==$n) | .path')
[ -z "$_path" ] && echo "no such skill" && exit 0
```

Use Read on the path.

## 4. Edit

Read the file. Ask what to change. Use Edit. Don't rewrite the whole file unless asked.

## 5. Remove

Confirm via AskUserQuestion. If yes:

```bash
rm -f "$_INSTALL_DIR/<name>/SKILL.md"
rmdir "$_INSTALL_DIR/<name>" 2>/dev/null || true
rm -rf "$_CUSTOM_DIR/<name>"
_new=$("$_FSTACK_BIN/fstack-config" get custom_skills | jq -c --arg n "<name>" 'map(select(.name != $n))')
"$_FSTACK_BIN/fstack-config" set-json custom_skills "$_new"
```

## Templates

### Checklist

```markdown
---
name: <slug>
version: 0.1.0
description: <description>
allowed-tools: [Bash, Read, Edit, Write]
triggers: [<phrases>]
---

# /<slug>

<one-paragraph purpose>

## Steps

1. <step>
2. <step>
3. <step>
```

### Workflow

```markdown
---
name: <slug>
version: 0.1.0
description: <description>
allowed-tools: [Bash, Read, Edit, Write, AskUserQuestion]
triggers: [<phrases>]
---

# /<slug>

<purpose>

## Branch

1. <A>
2. <B>

## 1. <A>

<steps>

## 2. <B>

<steps>
```

### Knowledge

```markdown
---
name: <slug>
version: 0.1.0
description: <description>
allowed-tools: [Read]
triggers: [<phrases>]
---

# /<slug>

Reference. Read, don't execute.

## <topic>

<content>
```

## Voice

Skill belongs to user. Don't second-guess naming unless it hits a reserved prefix. Mirror their described pattern. No ceremony.
