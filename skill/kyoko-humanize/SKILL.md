---
name: kyoko
description: >-
  Humanize AI-sounding prose with a user-supplied persona (proseGuidance + optional traits),
  optional confidence loop using ai-writing-auditor scoring. In-agent rewrite only. Two workflows:
  Persona setup vs Humanize.
---

# Kyoko (humanize)

## How to invoke (any host)

Behavior is **entirely in this file**. No Cursor- or Claude-specific logic required.

- **Skills:** Load `skill/kyoko-humanize/` via your product’s skill discovery, `@`-attach `SKILL.md`, or natural-language requests that match this skill’s `description`. **Claude Code:** install the **kyoko** plugin (`claude plugin add …` / `claude plugin install kyoko`) so **`skill/`** is registered from **`.claude-plugin/plugin.json`**, or use **`node install.js`** for a copy into `~/.claude/skills/`.
- **Optional slash commands:** If you copy command stubs into a project, [Claude Code](https://code.claude.com/docs/en/skills) uses **`.claude/commands/*.md`**; [Cursor](https://cursor.com/docs/context/skills) uses **`.cursor/commands/*.md`**. Stubs in this repo: **`kyoko`** → Humanize workflow, **`persona`** → Persona setup workflow. Same names, same canonical doc.

---

## Workflow: Persona setup

Use when the user wants to **create, edit, or persist** a writing persona—not when they are only passing draft text through humanize.

**Principle:** A persona is **structured instructions**, not only a label. Build **role**, **communication style**, and (when useful) **audience context**, then compile into **`proseGuidance`** + optional JSON fields.

**In-agent minimum**

- **`proseGuidance`** (string): single block the rewriter follows—cadence, register, phrases to use/avoid, audience.
- **Optional:** `name`, `role`, `communicationStyle`, `rtcF` object, `phrasesToUse` / `phrasesToAvoid`, example sentences.

**Loading (for later Humanize runs):** **`@`-reference** `.kyoko/persona.json`, paste inline, or rely on this workflow having **written** that file (see **Persist** below). If the user does not attach or paste a persona, the Humanize gate will auto-read `.kyoko/persona.json` from the workspace root before blocking.

**Optional homework (outside the agent):** Demographics, goals, and pain points can be drafted with tools such as [HubSpot Make My Persona](https://www.hubspot.com/make-my-persona) or [Xtensio](https://xtensio.com/user-persona/)—import highlights into the quiz answers below.

### Persona quiz / interview (agent-led)

Run in order; accept skips if the user already knows their voice.

1. **Role** — Who is this voice? Character and **tone** (e.g. gruff but helpful, enthusiastic, analytical). Who is the reader?
2. **Communication style** — **Vocabulary** level (plain, technical, playful). **Sentence shape** (short and punchy vs. flowing). **Phrases to use** and **phrases to avoid** (list explicitly).
3. **Goals & pain points (optional)** — What should this voice help the reader achieve? What should it *not* sound like?
4. **R-T-C-F check** — Assemble **Role, Task, Context, Format** for this project:
   - *Role:* the persona label in one line.
   - *Task:* what the rewrite is for (e.g. de-AI, blog, email).
   - *Context:* audience, stakes, taboos.
   - *Format:* length, headings, bullets allowed or not.
5. **Synthesize** — Draft **`proseGuidance`** that merges role + style + R-T-C-F into imperative instructions (not a character monologue). Add a one-line **example prompt** the user can reuse, e.g. *“Write this from the perspective of a Practical Mentor who simplifies complex topics and gives actionable steps from experience.”*

### Persist (automatic)

When **Persona setup** finishes (quiz, preset refinement, or edit merged into final JSON), **write the file yourself** using the project **workspace root** (the repo or folder the user has open):

- **Path:** **`.kyoko/persona.json`** (create directory **`.kyoko/`** if it does not exist).
- **Contents:** valid JSON with at least **`proseGuidance`**; include **`name`**, **`role`**, **`tone`**, **`communicationStyle`**, **`phrasesToUse`**, **`phrasesToAvoid`**, **`rtcF`**, and optional **`examplePrompt`** when you have them. Always initialize **`formattingDont`** with the universal em dash rule — add persona-specific entries to the array but never remove the em dash entry.
- **Override:** If the user names a different path, use that instead; if they say **do not save**, skip the write and only show JSON in chat.
- **Failure:** If you cannot write files (no workspace, permissions), paste the full JSON and tell them to create **`.kyoko/persona.json`** manually.

Do **not** wait for a separate “please save” step unless the user opted out above.

Suggested shape:

```json
{
  "name": "Practical Mentor",
  "role": "Experienced guide; simplifies complexity without talking down.",
  "tone": "Warm, direct, slightly informal.",
  "communicationStyle": "Short paragraphs; concrete verbs; no hype adjectives.",
  "phrasesToUse": ["here’s the tradeoff", "step one is"],
  "phrasesToAvoid": ["delve", "landscape", "leverage", "synergy"],
  "rtcF": {
    "role": "Practical Mentor",
    "task": "Humanize / rewrite user draft",
    "context": "Busy practitioners; low patience for fluff",
    "format": "Prose; bullets only for real lists"
  },
  "examplePrompt": "Write as a Practical Mentor who simplifies complex topics and gives actionable steps from experience.",
  "proseGuidance": "Write as a practical mentor: plain language, actionable steps, admit tradeoffs. Prefer concrete examples over abstractions. Avoid corporate AI diction and hollow intensifiers."
}
```

### Presets (quick pick)

Use as **starting templates**—always refine with the user’s **phrases to avoid** and **R-T-C-F**.

| Preset | Role & tone | Style sketch |
|--------|-------------|----------------|
| **Practical Mentor** | Experienced; simplifies hard topics | Actionable steps, tradeoffs named, no patronizing |
| **Analytical** | Measured, evidence-minded | Precise terms, qualified claims, tight logic |
| **Enthusiastic Guide** | Upbeat, approachable | Shorter sentences, vivid but not salesy |
| **Gruff but Helpful** | Direct, minimal fluff | Few adjectives, honest limitations, still respectful |
| **Plain Documentarian** | Neutral, clarity-first | Instructions/readme tone; no voice performance |

If the user picks a preset, **copy the row into `role` / `tone` / `communicationStyle` seeds**, run the **phrases to use/avoid** and **R-T-C-F** questions anyway (or use sane defaults), then write **`proseGuidance`** and **Persist** to **`.kyoko/persona.json`**.

Offer **presets** or the **quiz** when the user is in **Persona setup**. Do **not** improvise voice during **Humanize** (see gate below).

---

## Workflow: Humanize

Rewrite draft text to match an **explicit** persona, with optional **ai-writing-auditor** scoring loop.

### What it is

Kyoko rewrites stiff text to match a **persona** (traits + `proseGuidance`). Confidence comes from **`ai-writing-auditor`** rules in the sibling skill—not a separate ad-hoc scale.

### When to use

- User says Kyoko, humanize, persona, un-AI, “sound like me”
- After or alongside an **ai-writing-auditor** pass when a numeric confidence target matters

### Partner skill

**Always coordinate with [`ai-writing-auditor`](../ai-writing-auditor/SKILL.md):**

1. **Score** incoming or post-rewrite text using the auditor’s **confidence algorithm** (0–100) and findings.
2. **Humanize** with persona when the user wants voice, not only de-AI cleanup.
3. **Loop:** humanize → auditor score → if confidence &lt; `target_confidence` and passes remain, revise; stop on adaptive loop rules (see below).

**Hosts:** One agent session holds the text and runs the loop in chat (≤ `max_passes` iterations, often fewer). No skill-to-skill IPC.

If the user only needs AI-ism removal with no persona, **auditor alone** may suffice. If they need **voice + score**, use both.

### Persona gate (before humanize)

**Do not invent or assume a default voice** (no “neutral,” “generic conversational,” or silent preset).

If **`proseGuidance`** (or equivalent persona the user treats as authoritative) is **not** already in context:

1. **Auto-read** `.kyoko/persona.json` from the workspace root. If the file exists and contains `proseGuidance`, load it and proceed with the humanize pass — do not stop or ask.
2. If the file does not exist or contains no `proseGuidance`, **stop** the humanize pass.
3. Tell the user to complete **Workflow: Persona setup** (above)—paste persona, `@` **`.kyoko/persona.json`**, or run the **`persona`** slash command **if** their project includes that stub under `.claude/commands/` or `.cursor/commands/`.
4. **Do not** run the full persona interview inside a **Humanize** request unless the user **explicitly** asks for that interview in the same turn; otherwise keep setup separate.

If they only want **AI-ism stripping** with **no voice/persona**, use **[`ai-writing-auditor`](../ai-writing-auditor/SKILL.md)** alone instead of Kyoko humanize.

### Universal formatting constraints (apply to every rewrite, all personas)

These apply **regardless of persona**, pre-created or custom. They are not overridable by `proseGuidance`.

| Constraint | Rule | Severity | Replacement |
|-----------|------|----------|-------------|
| **Em dashes ( — )** | Budget: 1 per 1,000 words. Excess is a P1 AI-writing smell. | P1 | Use `:` (annotation/intro), `.` (new sentence), `,` (light pause), or `()` (aside). Never use as a clause joiner or separator. |

**When building or persisting a persona:** always write `formattingDont` into the JSON with at least the em dash rule:
```json
"formattingDont": ["em dashes ( — ): P1 AI smell, budget 1 per 1000 words. Replace with colon, period, comma, or parentheses depending on context."]
```
Add persona-specific entries to the array; never remove the em dash entry.

### Execution

With **explicit persona** (gate satisfied), rewrite in-chat using persona + auditor checklist + confidence rubric from [`ai-writing-auditor/SKILL.md`](../ai-writing-auditor/SKILL.md) **plus universal formatting constraints above**; output rewritten text + confidence + findings.

### Adaptive loop (shared with auditor doc)

- `max_passes` **5** · `target_confidence` **85** (persona may override) · `min_delta` **2**
- Stop: confidence ≥ target **or** passes ≥ max **or** improvement &lt; `min_delta` **twice** **or** similarity to original &lt; **0.9** (rollback)

---

## Repo paths

- User install: **`install.js`** at repository root (`node install.js` copies to `~/.claude/skills/`; **`--project`** / **`--user`** symlink full skill dirs — see repo **README.md**)
- Default persona file (project): **`.kyoko/persona.json`**
- This skill: `skill/kyoko-humanize/SKILL.md`
- Optional slash stubs: `.claude/commands/{kyoko,persona}.md`, `.cursor/commands/{kyoko,persona}.md`
- Auditor: `skill/ai-writing-auditor/SKILL.md`
