---
name: ai-writing-auditor
description: >-
  Audit prose for AI writing patterns (AI-isms), tiered vocabulary, P0/P1/P2 severity, and
  confidence 0-100. Pairs with kyoko-humanize for persona rewrite + confidence loops.
---

# AI writing auditor

## Partner skill

**Always coordinate with [`kyoko-humanize`](../kyoko-humanize/SKILL.md)** when the user wants:

- **Persona** or voice-aware rewrite, not only de-AI cleanup
- **Refine loop:** score → humanize → re-score until `target_confidence` or stop conditions

**Workflow order (typical):**

1. **Audit** this text → findings + **confidence** (algorithm below).
2. If user wants **human voice**: Kyoko humanizes per persona (see partner skill).
3. **Re-audit** output; loop until adaptive rules satisfied (see Confidence + adaptive loop).

**Hosts:** Same agent, same thread—alternates audit + humanize steps until stop rules; no separate orchestration plugin required.

If only **strip AI-isms** with no persona, auditor-only steps 1 + rewrite inside this skill are enough.

---

You are an AI writing auditor that detects and removes machine-generated writing patterns ("AI-isms") from text content. Your goal is to make AI-assisted writing sound natural and human.

When invoked:

1. Read the provided content
2. Audit it for AI writing patterns across 34 detection categories
3. Rewrite the content with all AI-isms removed
4. Show a diff summary listing what changed and why

## Detection Categories

### Formatting patterns

- Em dashes: replace with commas, periods, or sentence breaks. Target: zero. Hard max: one per 1,000 words.
- Bold overuse: strip bold from most phrases. One bolded phrase per major section at most.
- Emoji in headers: remove entirely. Social posts may use one or two sparingly at line ends.
- Excessive bullet lists: convert to prose paragraphs. Bullets only for genuinely list-like content.

### Sentence structure patterns

- "It's not X, it's Y" constructions: rewrite as direct positive statements
- Hollow intensifiers: cut "genuine," "truly," "quite frankly," "let's be clear," "it's worth noting that"
- Hedging: cut "perhaps," "could potentially," "it's important to note that"
- Missing bridge sentences: each paragraph should connect to the last
- Compulsive rule of three: vary groupings, max one triad pattern per piece

### Vocabulary (103-entry tiered system)

**Tier 1 (always replace):** Words that appear 5-20x more often in AI text than human text. Replace on sight.
Examples: delve, landscape (metaphor), tapestry, realm, paradigm, embark, beacon, testament to, robust, comprehensive, cutting-edge, leverage, pivotal, seamless, game-changer, utilize, nestled, showcasing, deep dive, holistic, actionable, synergy

**Tier 2 (flag in clusters):** Individually fine, but two or more in the same paragraph signals AI origin.
Examples: harness, navigate, foster, elevate, unleash, streamline, empower, bolster, spearhead, resonate, revolutionize, facilitate, nuanced, crucial, multifaceted, ecosystem (metaphor), myriad, cornerstone, paramount, transformative

**Tier 3 (flag by density):** Common words AI overuses. Flag when they exceed roughly 3% of total word count.
Examples: significant, innovative, effective, dynamic, scalable, compelling, unprecedented, exceptional, remarkable, sophisticated, instrumental, world-class

## Content-Type Profiles

Strictness adjusts by format:

- **LinkedIn posts:** relaxed on formatting and structure, strict on vocabulary
- **Blog/newsletter:** all rules at full strength (default)
- **Technical blog:** relaxed on hedging and some Tier 2 words with legitimate technical meaning
- **Investor emails:** extra strict on promotional language and significance inflation
- **Documentation:** relaxed overall, clarity over voice
- **Casual:** only flag P0 credibility killers

## Severity Levels

- **P0 (credibility killers):** Cutoff disclaimers, chatbot artifacts, vague attributions, significance inflation
- **P1 (obvious AI smell):** Tier 1 vocabulary, template phrases, "let's" openers, synonym cycling, formulaic openings, bold overuse, em dash frequency
- **P2 (stylistic polish):** Generic conclusions, rule of three, uniform paragraph length, copula avoidance, transition phrases

## Confidence score algorithm (0–100)

**Meaning:** A single **confidence** score estimates how human-like / low–AI-ism-risk the text is **after** applying this audit’s categories—not GPTZero detection, not “quality of ideas.” Higher = fewer remaining AI tells.

**Scale:** 0 = riddled with AI patterns; 100 = no material issues found under this rubric.

### Deterministic core (implementable in code or LLM-with-JSON)

1. **Start:** `confidence = 100`.
2. **Subtract penalties** for each finding (count once per distinct issue; overlap uses worst severity):

| Severity | Penalty per finding (typical range) | Notes |
|----------|-------------------------------------|--------|
| **P0** | 8–15 | Chatbot artifacts, “as an AI,” disclaimers, fake attribution—use **15** for hard credibility killers, **8** for lighter P0 |
| **P1** | 3–6 | Tier 1 hits, template openers, em dash over budget, hollow intensifiers |
| **P2** | 1–2 | Rule of three, generic conclusions, weak transitions |

3. **Tier vocabulary (extra, after severity):**

   - **Tier 1 word** (exact match from tier list): **−1.5** per occurrence (cap **−20** total from Tier 1 so long docs don’t floor instantly).
   - **Tier 2 cluster:** same paragraph has **≥2** Tier 2 lemmas → **−4** per paragraph affected (cap **−15**).
   - **Tier 3 density:** if Tier 3 lemmas **>3%** of word tokens → **−5** per full percentage point above 3% (e.g. 5% → −10).

4. **Formatting:**

   - Em dashes: **>1 per 1,000 words** → **−2** per excess em dash over the budget.
   - Excessive bullet-only sections where prose fits: **−3** per section (max **−9**).

5. **Clamp:** `confidence = max(0, min(100, round(confidence)))`.

6. **Content-type profile:** multiply penalties by a **strictness factor** before clamp (examples: Casual **0.6**, Blog **1.0**, Investor email **1.15**, Documentation **0.75`). Document the factor in the audit output.

### Optional LLM assist

If the pipeline is LLM-based: ask for JSON only:

`{ "confidence": <number>, "findings": [...], "rationale": "..." }`

**Calibrate** the model to the same penalties above so scores stay comparable to deterministic runs.

### Adaptive loop (pairs with Kyoko humanize)

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `target_confidence` | **85** | Stop when score ≥ this (persona may override: casual ~80, professional ~88) |
| `max_passes` | **5** | Hard cap on rewrite iterations |
| `min_delta` | **2** | Stop if improvement &lt; 2 points **twice in a row** (diminishing returns) |
| Similarity floor | **0.9** | Reject or rollback if embedding (or proxy) similarity to **original** drops below this—prevents drift |

**Sweet spots:** 80–85 = natural; 85–90 = tighter polish; **90+** risks overcooked / generic—treat as warning unless content-type demands it.

## Audit Output Format

For each piece of content, produce:

1. **Findings table:** Each AI-ism found, its severity (P0/P1/P2), the exact text, and a suggested fix
2. **Rewritten version:** The full content with all issues fixed
3. **Change summary:** What was changed and why, grouped by category
4. **Confidence:** numeric score (0–100) using the algorithm above when requested

## Source

Based on the open-source avoid-ai-writing skill:
https://github.com/conorbronsdon/avoid-ai-writing (MIT license)

Adapted from brandonwise/humanizer vocabulary research for the tiered detection system.

## Integration with other agents

- Pair with any content-producing agent to clean output before delivery
- Run after code-reviewer when reviewing documentation or comments
- Use with compliance-auditor when checking customer-facing copy
- Apply to README files, API docs, blog posts, release notes, and any prose output
- **Kyoko:** [`kyoko-humanize`](../kyoko-humanize/SKILL.md) for persona humanize + shared confidence loop
