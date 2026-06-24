# Kyoko: clean up your model’s writing

**KYOKO** = *Know Your Output, Keep it Organic.* Cheesy acronym? Maybe. Accurate? When the draft sounds like a press release wrote it, yeah.

# Kyoko skills

Three **Agent Skills** that stack on purpose: **Kyoko** humanizes prose to **an explicit persona** (you choose it—no ghost “neutral voice”), the **AI writing auditor** scores the text for AI-isms and runs a confidence loop, and **code-humanizer** strips AI tells from generated code. Put them in so the agent can **audit ↔ humanize** without improvising who you sound like.

| Skill | Folder | Role |
|-------|--------|------|
| **kyoko-humanize** | `skill/kyoko-humanize/` | Persona setup + Humanize workflows (in-agent) |
| **ai-writing-auditor** | `skill/ai-writing-auditor/` | AI-ism audit (structural-first), banded confidence 0–100, findings |
| **code-humanizer** | `skill/code-humanizer/` | Strip AI tells from code + commit/PR prose |

The real instructions are in each **`SKILL.md`** (they point at each other—read them if you’re wiring the loop).

**2026 detection model:** the auditor weights **structural/rhetorical tells** (the “-ing” analysis tail, significance pivots, bold-colon lists, negative parallelism, formulaic scaffolding) as primary, and treats vocabulary and em dashes as **corroborating-only**. Single-word lists (delve, tapestry…) decay every model generation; structure survives paraphrase. The confidence score is reported as a **band** (reads-human / mixed / reads-AI) with per-signal caps and an ESL false-positive guardrail—it’s AI-ism *risk under a rubric*, never a verdict on authorship.

---

## What actually happens

1. **Persona setup** — Quiz, presets, and (by default) **`.kyoko/persona.json`**. Nobody sneaks in a default persona.
2. **Humanize** — Rewrite against **`proseGuidance`**, structural tells before vocabulary. If persona isn’t in context, **Kyoko stops** instead of faking a voice.
3. **Auditor loop** — Score → humanize → re-score until the shared “good enough / stop” rules fire.
4. **Code** — If the target is code, hand off to **code-humanizer** (run the auditor on comments/docs first, then humanize the structure).

Nitty-gritty: **`skill/kyoko-humanize/SKILL.md`**, **`skill/ai-writing-auditor/SKILL.md`**, and **`skill/code-humanizer/SKILL.md`**.

---

## Install

### Claude Code (recommended): plugin

```bash
claude plugin add justjammin/kyoko-skill
claude plugin install kyoko
```

Forking? Swap **`justjammin/kyoko-skill`** for your slug.

The plugin id in **`.claude-plugin/plugin.json`** is **`kyoko`** — that’s what **`claude plugin install`** wants.

**Local / unpublished checkout:**

```bash
claude plugin add /absolute/path/to/kyoko-skill
claude plugin install kyoko
```

There’s **no SessionStart hook** here. You pull skills in via description, **`@`**, or drop **`.claude/commands/`** into the project if you like **`/kyoko`** / **`/persona`**.

### Claude Code: copy install (no plugin)

From a clone:

```bash
node install.js
```

Needs **`~/.claude`**. Copies all three **`SKILL.md`** files to:

- `~/.claude/skills/kyoko-humanize/SKILL.md`
- `~/.claude/skills/ai-writing-auditor/SKILL.md`
- `~/.claude/skills/code-humanizer/SKILL.md`

### npx

When the package is published (or you point npx at git):

```bash
npx kyoko-skills
```

(`package.json` wires **`kyoko-skills`** → **`install.js`**.)

### Manual

```bash
mkdir -p ~/.claude/skills/kyoko-humanize ~/.claude/skills/ai-writing-auditor ~/.claude/skills/code-humanizer
curl -o ~/.claude/skills/kyoko-humanize/SKILL.md \
  https://raw.githubusercontent.com/<your-org>/kyoko-skill/main/skill/kyoko-humanize/SKILL.md
curl -o ~/.claude/skills/ai-writing-auditor/SKILL.md \
  https://raw.githubusercontent.com/<your-org>/kyoko-skill/main/skill/ai-writing-auditor/SKILL.md
curl -o ~/.claude/skills/code-humanizer/SKILL.md \
  https://raw.githubusercontent.com/<your-org>/kyoko-skill/main/skill/code-humanizer/SKILL.md
```

Swap **`<your-org>/kyoko-skill`** for your fork or upstream.

### Symlinks (full skill folders)

Same **`install.js`**; use this when you want the **whole** **`skill/<name>/`** tree linked, not just **`SKILL.md`**:

```bash
# From the project where .claude / .cursor should live:
node /path/to/kyoko-skill/install.js --project
```

```bash
# User-wide symlinks (~/.claude/skills and ~/.cursor/skills):
node install.js --user
```

Flags: **`--claude-only`** · **`--cursor-only`** · **`node install.js --help`**

---

## Usage

**Skills:** Say what you’re doing in a way that matches each skill’s frontmatter **`description`**, or **`@`-attach** a **`SKILL.md`**.

**Optional slash commands:** Copy or symlink this repo’s **`.claude/commands/`** (Claude Code) and/or **`.cursor/commands/`** (Cursor) into your project:

| Command | Purpose |
|---------|---------|
| **`/kyoko`** | Humanize workflow (persona required — see skill **Persona gate**) |
| **`/persona`** | Persona quiz / presets; when you’re done, the skill says to write **`.kyoko/persona.json`** at the workspace root unless you opt out |
| **`/code-humanize`** | Strip AI tells from generated code + commit/PR prose (see **code-humanizer** pipeline) |

The command stubs are thin; **`skill/kyoko-humanize/SKILL.md`** owns the behavior.

---

## Where it runs

| Environment | Install |
|-------------|---------|
| **Claude Code** | `claude plugin install kyoko@kyoko` — or `node install.js` |
| **Cursor** | `node install.js` (copy) or `node install.js --project` / `--user` for symlinks — Cursor also picks up `~/.claude/skills/` |
| **Other Agent Skills hosts** | Copy **`skill/*/`** into that tool’s skills directory, or use [agentskills.io](https://agentskills.io) flows if you ship a package |

Docs: [Claude Code skills](https://code.claude.com/docs/en/skills), [Cursor Agent Skills](https://cursor.com/docs/context/skills).

---

## License

MIT. [justjammin](https://github.com/justjammin).