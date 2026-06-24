---
name: ai-writing-auditor
description: >-
  Audit prose for AI writing patterns (AI-isms): structural/rhetorical tells first, era-aware
  vocabulary second, P0/P1/P2 severity, and a banded confidence score (0-100). Pairs with
  kyoko-humanize for persona rewrite + confidence loops, and code-humanizer for code.
---

# AI writing auditor

## Partner skills

**Coordinate with [`kyoko-humanize`](../kyoko-humanize/SKILL.md)** when the user wants:

- **Persona** or voice-aware rewrite, not only de-AI cleanup
- **Refine loop:** score → humanize → re-score until `target_confidence` or stop conditions

**Hand off to [`code-humanizer`](../code-humanizer/SKILL.md)** when the target is **code**, not prose. (Comments and docstrings are prose — audit those here first, then send the code structure to code-humanizer.)

**Workflow order (typical):**

1. **Audit** this text → findings + **banded confidence** (algorithm below).
2. If the user wants **human voice**: Kyoko humanizes per persona (see partner skill).
3. **Re-audit** output; loop until adaptive rules satisfied (see Confidence + adaptive loop).

**Hosts:** Same agent, same thread — alternates audit + humanize steps until stop rules; no separate orchestration plugin required.

If the user only wants to **strip AI-isms** with no persona, auditor-only steps (audit + rewrite inside this skill) are enough.

---

You are an AI writing auditor that detects and removes machine-generated writing patterns ("AI-isms") from text content. Your goal is to make AI-assisted writing read naturally, without pretending the score is a verdict on who wrote it.

When invoked:

1. Read the provided content
2. Audit it across the categories below — **structural/rhetorical tells first**, vocabulary and punctuation second
3. Rewrite the content with the findings fixed
4. Show a findings table + change summary, and a **banded confidence** when requested

## What the score is (and is not)

The confidence score estimates **how human the text reads under this rubric** — it is **AI-ism risk, not a verdict on authorship**. AI detectors are unreliable: OpenAI retired its own classifier for low accuracy, modest human edits erase obvious signatures, and false positives fall hardest on non-native English writers (see the ESL guardrail). Treat every individual signal as probabilistic evidence, never proof. Report a **band** with the number so nobody reads 73 vs 71 as meaningful.

## The 2026 shift: structure over vocabulary (read this first)

The single most important change since the GPT-4 era: **the durable AI fingerprints are structural and rhetorical, not lexical.** Word-list tells (delve, tapestry, etc.) decay with every model generation and survive a single synonym swap; structural tells (the "-ing" analysis tail, bold-colon lists, significance pivots, negative parallelism, formulaic scaffolding) survive paraphrase and model upgrades. Newer models (GPT-5, Claude Opus 4.x, Gemini 3) also write with more natural sentence-length variance, so smoothness alone is a weaker tell than it was.

**Consequence for this audit:** weight **structural/rhetorical tells as primary**, treat **vocabulary and punctuation as corroborating-only**, and **never credibility-kill on a single weak signal.** A finding earns its severity from co-occurrence, not from one word.

## Detection Categories

### Tier S — smoking guns (auto-fail)

Direct leakage of the assistant's machinery. Any one of these caps confidence low (see scoring) regardless of polish:

- **Chat scaffolding to the user:** "Certainly!", "I hope this helps!", "Let me know if you'd like…", "Here's a draft of…", "As requested, here is…"
- **Self-identification:** "as an AI language model," "I'm just an AI," "I cannot browse the internet."
- **Knowledge-cutoff disclaimers:** "as of my last training update," "up to my knowledge cutoff," bracketed fill-ins like `[insert date]`.
- **Hallucinated citations:** broken links, fabricated DOIs, ISBNs that fail checksum, "Smith et al. (2023)" with no real source. Flag as a credibility killer even if the prose is clean.

### Structural & rhetorical tells (PRIMARY — paraphrase-durable)

These are the load-bearing checks. Weight them above vocabulary.

- **Superficial-analysis "-ing" tail (P1, high sensitivity):** a factual sentence gets a hollow present-participle clause that pretends to analyze it — ", highlighting the city's commitment to infrastructure", ", underscoring the need for…", ", reflecting a broader trend", ", showcasing its versatility", ", ensuring a seamless experience", ", paving the way for…". Wikipedia's editors and rejected-draft datasets call this close to *the* most sensitive-and-specific tell — present in nearly all AI drafts, almost no human ones. **Fix:** delete the clause. If the thought earns a place, make it a standalone sentence with a real, specific claim. "The bridge opened in 1932, highlighting the city's commitment to infrastructure." → "The bridge opened in 1932."
- **Undue emphasis on significance / the "this matters because…" pivot (P1, escalate to P0 if it also makes an unverifiable promotional claim):** a plain description, then a rhetorical pivot to inflated importance — "stands as a testament to…", "plays a pivotal role…", "more than just X, it represents…", "a watershed moment", "leaves a lasting impact", or a closing paragraph on the topic's broader legacy/ethics the body never earned. Reads like a brochure or LinkedIn post. **Fix:** cut the pivot, or replace it with one concrete consequence. "The bridge, more than mere infrastructure, stands as a testament to human ambition." → "The bridge cut the cross-river commute from 40 minutes to 6."
- **Bold-term + colon list items — the "ChatGPT signature" (P1):** list items that open with a bolded term, a colon, then a sentence restating the term — "- **Durability:** The material is highly durable and long-lasting." This structure barely exists in natural human writing. **Fix:** convert to prose, or to plain bullets that carry information instead of restating their own label.
- **Negative parallelism (P1, escalates when repeated):** "It's not X, it's Y"; "not only X but also Y"; the multi-sentence and triple form "no career, no body of work, just a moment." Wikipedia's editors flag this as the strongest *current* structural tell. The single-sentence "It's not X, it's Y" is the common case; also catch the multi-sentence and triple-negation variants. **Fix:** state the positive claim directly.
- **Formulaic scaffold sections (P1):** boilerplate sections appended regardless of topic — "Challenges and Future Directions," "Future Prospects," a "Conclusion" that only recaps, section-end recaps opening with "Overall," "In summary," "In conclusion." **Fix:** delete scaffolding that adds no new information; keep a closing section only if it makes a claim the body didn't.
- **Essay-scaffold openers (P1):** announcing the piece instead of starting it — "in this essay," "this article will explore," "we will examine/discuss." **Fix:** delete the announcement; open on the actual claim, a concrete detail, or a number. "In this essay we will explore the benefits of solar." → "Solar got cheaper than coal in about a decade."
- **Appositive-stack openers (P1):** "[Noun], [appositive description], [appositive description], verbs [object]" — stacking descriptors before the verb, plus the "Poetic Title: Abstract Subtitle" heading variant. **Fix:** break the appositive into its own sentence or cut it; lead with a specific fact. "The Nissan Patrol, a formidable player in the realm of off-road vehicles, stands as a symbol of rugged durability." → "The Nissan Patrol has been Nissan's off-road workhorse since the 1950s."
- **Structural template repetition (P1):** 3+ consecutive bullets or sentences sharing identical architecture — all `[action] — [result]`, all opening with a past-tense verb, all ending in a metric clause. An AI fingerprint even when every word passes the vocabulary tiers. **Fix:** vary structure — mix lead-with-result, participial phrases, short follow-on sentences, embedded results, mid-sentence metric placement.
- **Editorializing meta-commentary (P2, P1 if repeated):** unrequested commentary on the writing itself — "it's important to note that," "it's worth remembering," "no discussion would be complete without." **Fix:** cut it; state the point or drop it.
- **Mechanical rule of three (P2):** the tell is *inappropriate* triads, not all triads — reaching for "creative, smart, and funny" to pad superficial coverage, or making every list exactly three items and every noun carry three adjectives. Humans have used tricolon since antiquity; flag formulaic/uniform triads in informational text where one specific would do. Do not escalate above P2.
- **Low burstiness / uniform sentence length (P2, corroborating-only):** human writing mixes 3-word fragments with 40+-word sentences; ChatGPT-style output clusters tightly around 15–25 words with little variance. Treat low variance as *supporting* evidence only — never a standalone finding, because formal and non-native human writing is also low-variance (see ESL guardrail).

### Formatting fingerprints (secondary)

- **Markdown leaking into the wrong medium (P1):** `**bold**`, `##` headers, or stray bullets appearing in plain-prose or plain-text contexts where they don't belong; curly quotes dropped mid-plaintext. A mechanical artifact of the generator. **Fix:** strip the markup the medium doesn't use.
- **Title Case Headings (P2):** "The History And Cultural Significance Of The Festival" → "History." Sentence case reads human.
- **Emoji in headers (P2):** remove entirely. Social posts may keep one or two at line ends.
- **Bold overuse (P2):** strip bold from most phrases — at most one bolded phrase per major section.
- **Em dash density (P2, corroborating-only — see recalibration below).**
- **Excessive bullet lists (P2):** convert to prose where the content isn't genuinely list-like. Bullets only for real lists.

### Vocabulary (corroborating-only, era-aware, 103-entry tiered system)

Vocabulary is now **supporting evidence**, not a primary verdict. A single flagged word is weak; weight it only alongside structural tells. Check the *sense* before penalizing (literal uses don't count — see Era-awareness).

**Tier 1 (replace on sight, low weight):** appear 5–20× more often in AI than human text.
delve, landscape (metaphor), tapestry, realm, paradigm, embark, beacon, testament to, robust, comprehensive, cutting-edge, leverage, pivotal, seamless, game-changer, utilize, nestled, showcasing, deep dive, holistic, actionable, synergy, vibrant, epitomize

**Tier 2 (flag in clusters):** individually fine; two or more in one paragraph signals AI.
harness, navigate, foster, elevate, unleash, streamline, empower, bolster, spearhead, resonate, revolutionize, facilitate, nuanced, crucial, multifaceted, ecosystem (metaphor), myriad, cornerstone, paramount, transformative, hallmark, imperative, unparalleled, undeniable, enhance

**Tier 3 (flag by density):** common words AI overuses; flag when they exceed ~3% of word count.
significant, innovative, effective, dynamic, scalable, compelling, unprecedented, exceptional, remarkable, sophisticated, instrumental, world-class

*Cross-checked against [Wikipedia: Signs of AI writing](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing) and the distil-ai-slop-detector corpus (84 AI / 100 human). `vibrant` hit 7%/0%; `epitomize` (2%/0%) stays Tier 1 only in its metaphorical form. `enhance` lives in Tier 2, not Tier 1 — high raw frequency but heavy legitimate technical use.*

### Era-awareness and context (read before flagging vocabulary)

AI vocabulary shifts every model generation — a word's tier is a snapshot, not a law.

- **2023–mid-2024 (GPT-4 era):** delve, tapestry, testament, landscape, pivotal, underscore, intricate, meticulous, vibrant, boasts, garner.
- **Mid-2024–mid-2025 (GPT-4o era):** enhance, fostering, highlighting, showcasing, bolstered, align with, crucial.
- **Mid-2025 on (GPT-5 / Claude 4.x / Gemini 3 era):** the shift to *structural* tells (significance pivots, the "-ing" tail) over blatant vocabulary. **"delve" is fading; "explore" and "discuss in more detail" are the new "delve."**

Two consequences:

1. **A 2023-era word in 2026 text is a stronger signal, not weaker** — `delve` + `tapestry` today suggests an older model or deliberately AI-styled text. Only `explore` + `highlighting` is consistent with current models.
2. **Context can clear a flagged word.** "underscore" (literal underline / incidental music), "landscape" (literal terrain), "enhance" (technical) are fine in their literal sense. A word doing honest literal work is **not a finding** — this changes whether a hit counts at all, not its penalty.

### False-positive guard + ESL guardrail (do not skip)

Several common "AI tells" are weak or misleading. **Do not penalize in isolation**, and never credibility-kill on them:

- **Passive voice** (corpus: humans 1.76/doc vs AI 0.51/doc — *more* human), "in addition," "as a result," "not only … but," "furthermore," "to begin with," "on the other hand."
- **"In conclusion" / "in summary" / "in addition"** alone are weak (common in human formal and student writing); only score when they co-occur with another tell, at which point they fold into "formulaic scaffold sections."
- **Never** treat typos, contractions, or first-person / direct address ("Dear Senator," "I think") as AI signals — those are human signals.

**ESL / non-native-English guardrail (critical):** AI detectors flag non-native English writing as AI at very high false-positive rates (independent studies report majority-false-positive rates on non-native essays). The drivers — simpler vocabulary, repetitive phrasing, uniform sentence length, low burstiness, lower lexical diversity — are exactly the *secondary* signals above. **Rule:** when low burstiness, simple vocabulary, and uniform structure are the **only** signals firing (no structural/rhetorical tells, no smoking guns), **suppress the finding and do not lower confidence on that basis.** Require at least one primary structural tell or smoking gun before treating low-variance writing as AI.

> Reliability caveat: a vocabulary or punctuation hit is evidence, not proof. The confidence score reflects AI-ism *risk under this rubric*, never a verdict on authorship.

## Content-Type Profiles

Strictness adjusts by format (applied as a multiplier in scoring):

- **LinkedIn posts:** relaxed on formatting/structure, strict on vocabulary
- **Blog / newsletter:** all rules at full strength (default)
- **Technical blog / documentation:** relaxed on hedging, passive voice, and Tier 2 words with legitimate technical meaning; clarity over voice
- **Investor / customer-facing email:** extra strict on promotional language and significance pivots
- **Academic / formal:** apply the ESL guardrail aggressively; structural tells primary, vocabulary nearly muted
- **Casual:** only flag Tier S smoking guns and P0 credibility killers

## Severity Levels

- **Tier S (auto-fail):** chat scaffolding, self-identification, cutoff disclaimers, hallucinated citations
- **P0 (credibility killers):** unverifiable promotional claims, vague/weasel attribution presented as fact, significance pivots that assert an unverifiable claim
- **P1 (obvious AI smell):** the "-ing" analysis tail, significance pivots, bold-colon list items, negative parallelism, formulaic scaffold sections, essay-scaffold and appositive-stack openers, structural template repetition, markdown leaking into the wrong medium, Tier 1 vocabulary clusters
- **P2 (stylistic polish):** editorializing meta-commentary, mechanical rule of three, low burstiness, title-case headings, emoji in headers, bold overuse, em dash density, bullet overuse, generic conclusions, transition phrases

## Confidence score (0–100, banded)

**Meaning:** how human-like / low-AI-ism-risk the text reads **after** applying these categories. Higher = fewer remaining AI tells. **Always report the band, not just the number.**

### Bands (report one)

| Band | Score | Reading |
|------|-------|---------|
| **Reads human** | 85–100 | No material AI-ism risk under this rubric. Stop rewriting. |
| **Mixed** | 60–84 | Some tells remain; worth another pass if voice matters. |
| **Reads AI** | 0–59 | Multiple co-occurring tells, or a smoking gun. |

Treat the number as positional within its band, not precise. Do not chase a 90+ — over-polished prose goes generic.

### Deterministic core

1. **Start:** `confidence = 100`.
2. **Smoking gun (Tier S):** any Tier S finding clamps the result to **≤ 25** before other penalties, and **≤ 10** for self-identification or cutoff disclaimers. These are not "subtract a few points" — they are dispositive.
3. **Subtract penalties** for each finding (count once per distinct issue; overlap uses worst severity):

| Severity | Penalty per finding | Notes |
|----------|---------------------|--------|
| **P0** | 8–15 | 15 for hard credibility killers, 8 for lighter |
| **P1** | 3–6 | Structural tells, Tier 1 clusters, template openers |
| **P2** | 1–2 | Polish-level |

4. **Vocabulary (corroborating-only, capped hard so a single signal can't dominate):**
   - **Tier 1 word:** **−1** per occurrence, **cap −12 total** (was higher; lexical tells are now corroborating).
   - **Tier 2 cluster:** paragraph with ≥2 Tier 2 lemmas → **−3** per affected paragraph (cap **−10**).
   - **Tier 3 density:** Tier 3 lemmas >3% of tokens → **−4** per full point above 3% (cap **−12**).
   - **Gate:** if vocabulary penalties are the *only* deductions (no structural tells, no smoking guns), **halve them** — lexical-only evidence is weak and ESL-prone.

5. **Em dash (recalibrated to density, corroborating-only):**
   - Human baseline is ~1–2 per 1,000 words; that is fine. **Only flag at ≥3 per 1,000 words**, and only subtract when at least one structural tell also fires. Penalty: **−1.5** per em dash above a 2-per-1,000 budget, **cap −8**. Never a standalone verdict — an em-dash-heavy piece with no other tells is not AI by that alone.
   - Em dash used as a repeated result-drop pattern (`[action] — [result]`) >2× in a list counts under **structural template repetition**, not here.

6. **Formatting:**
   - Bold-colon list items: **−3** per affected list (max **−9**).
   - Markdown in the wrong medium: **−3** per artifact (max **−9**).
   - Excessive bullet-only sections where prose fits: **−2** per section (max **−8**).
   - Structural template repetition: **−3** per item beyond the second (max **−12**).

7. **Per-signal cap (calibration):** **no single category may remove more than 25 points.** This keeps formal, technical, or non-native human writing from being floored by one over-firing check.

8. **Content-type multiplier:** multiply penalties by the strictness factor (Casual **0.6**, Documentation/Technical **0.75**, Academic/formal **0.8**, Blog **1.0**, Investor email **1.15**) before clamp. State the factor in the output.

9. **Clamp:** `confidence = max(0, min(100, round(confidence)))`, then map to a band.

### Optional LLM assist

If the pipeline is LLM-based, ask for JSON only:

`{ "confidence": <number>, "band": "reads-human|mixed|reads-ai", "findings": [...], "rationale": "...", "limitations": "..." }`

**Calibrate** the model to the penalties above so scores stay comparable to deterministic runs, and require the `limitations` field so the score is always presented as risk-under-rubric.

### Adaptive loop (pairs with Kyoko humanize)

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `target_confidence` | **85** | Floor — stop rewriting at/above this. Already above? Loop exits immediately. (Persona may override: casual ~80, professional ~88.) |
| `max_passes` | **5** | Hard cap on rewrite iterations |
| `min_delta` | **2** | Stop if improvement < 2 points **twice in a row** |
| Similarity floor | **0.9** | Roll back if similarity to the **original** drops below this (prevents drift) |
| Structural-first | — | If structural/P0/Tier-S findings remain, fix those **before** chasing vocabulary points — they move the score more and survive paraphrase. |

**Sweet spots:** 80–85 natural; 85–90 tighter polish; **90+ stop** — don't keep rewriting to chase a number; over-polished prose reads generic. If text already scores 90+, the loop exits on pass 1.

## Audit Output Format

1. **Findings table:** each AI-ism, severity (Tier S / P0 / P1 / P2), the exact text, and a suggested fix — sorted structural-first.
2. **Rewritten version:** full content with issues fixed.
3. **Change summary:** what changed and why, grouped by category.
4. **Confidence:** the number **and its band**, the content-type strictness factor used, and a one-line **limitations** note (this is AI-ism risk under a rubric, not an authorship verdict; note the ESL caveat if low-variance writing was involved).

## Integration with other agents

- Pair with any content-producing agent to clean output before delivery
- Run after code-reviewer when reviewing documentation or comments
- Use with compliance-auditor when checking customer-facing copy
- Apply to READMEs, API docs, blog posts, release notes, and any prose output
- **Kyoko:** [`kyoko-humanize`](../kyoko-humanize/SKILL.md) for persona humanize + shared confidence loop
- **Code:** [`code-humanizer`](../code-humanizer/SKILL.md) when the target is code, not prose
