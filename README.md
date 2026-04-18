# Kyoko skills

Two **Agent Skills** that work together: **Kyoko** humanizes prose to a explicit persona; **AI writing auditor** scores copy for AI-isms and drives a confidence loop. Install both so the agent can chain **audit ↔ humanize** without inventing a default voice.

| Skill | Folder | Role |
|-------|--------|------|
| **kyoko-humanize** | `skill/kyoko-humanize/` | Persona setup + Humanize workflows (in-agent) |
| **ai-writing-auditor** | `skill/ai-writing-auditor/` | AI-ism audit, confidence 0–100, findings |

Canonical instructions live in each **`SKILL.md`** (they link to each other).

---

## What actually happens

1. **Persona setup** — Structured quiz, presets, optional **`.kyoko/persona.json`** (no silent default voice).
2. **Humanize** — Rewrite draft text to match **`proseGuidance`**; **persona gate** stops the run if persona isn’t in context yet.
3. **Auditor loop** — Score → humanize → re-score until shared adaptive rules say stop.

Details: **`skill/kyoko-humanize/SKILL.md`** and **`skill/ai-writing-auditor/SKILL.md`**.

---

## Install

### Claude Code (recommended): plugin

```bash
claude plugin add justjammin/kyoko-skill
claude plugin install kyoko
```

(Fork? Replace **`justjammin/kyoko-skill`** with your slug.)

The plugin id in **`.claude-plugin/plugin.json`** is **`kyoko`** — that’s the name **`claude plugin install`** expects.

**Local / unpublished checkout:**

```bash
claude plugin add /absolute/path/to/kyoko-skill
claude plugin install kyoko
```

(No SessionStart hook—invoke skills by description, **`@`**, or copy **`.claude/commands/`** for **`/kyoko`** / **`/persona`**.)

### Claude Code: copy install (no plugin)

From a clone of this repository:

```bash
node install.js
```

Requires **`~/.claude`**. Copies both **`SKILL.md`** files to:

- `~/.claude/skills/kyoko-humanize/SKILL.md`
- `~/.claude/skills/ai-writing-auditor/SKILL.md`

### npx

When this package is published (or via git specifier):

```bash
npx kyoko-skills
```

(`package.json` maps the **`kyoko-skills`** bin to **`install.js`**.)

### Manual

```bash
mkdir -p ~/.claude/skills/kyoko-humanize ~/.claude/skills/ai-writing-auditor
curl -o ~/.claude/skills/kyoko-humanize/SKILL.md \
  https://raw.githubusercontent.com/<your-org>/kyoko-skill/main/skill/kyoko-humanize/SKILL.md
curl -o ~/.claude/skills/ai-writing-auditor/SKILL.md \
  https://raw.githubusercontent.com/<your-org>/kyoko-skill/main/skill/ai-writing-auditor/SKILL.md
```

Replace **`<your-org>/kyoko-skill`** with your fork or upstream path.

### Symlinks (full skill folders)

Same installer; use when you want the whole **`skill/<name>/`** directory linked (not just **`SKILL.md`**):

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

**Skills:** Describe a task that matches each skill’s frontmatter **`description`**, or **`@`-attach** a **`SKILL.md`**.

**Optional slash commands:** Copy or symlink this repo’s **`.claude/commands/`** (Claude Code) and/or **`.cursor/commands/`** (Cursor) into your project:

| Command | Purpose |
|---------|---------|
| **`/kyoko`** | Humanize workflow (persona required — see skill **Persona gate**) |
| **`/persona`** | Persona quiz / presets; on completion the skill instructs the agent to write **`.kyoko/persona.json`** in the workspace root (unless the user opts out) |

Stubs only point at **`skill/kyoko-humanize/SKILL.md`**; all behavior stays in that file.

---

## Where it runs

| Environment | Install |
|-------------|---------|
| **Claude Code** | `claude plugin add justjammin/kyoko-skill` then `claude plugin install kyoko` — or `node install.js` |
| **Cursor** | `node install.js` (copy) or `node install.js --project` / `--user` for symlinks — Cursor also loads `~/.claude/skills/` |
| **Other Agent Skills hosts** | Copy **`skill/*/`** into that product’s skills directory, or use [agentskills.io](https://agentskills.io) tooling if you publish |

Docs: [Claude Code skills](https://code.claude.com/docs/en/skills), [Cursor Agent Skills](https://cursor.com/docs/context/skills).

---

## License

MIT 
