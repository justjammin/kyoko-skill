---
name: code-humanizer
description: >-
  Strip AI tells from generated code. Transform pipeline: remove obvious comments,
  collapse needless helpers, deflate names, remove defensive noise, match surrounding
  style, inject intentional slop. Companion to kyoko-humanize — use when target is code,
  not prose. Trigger: "humanize this code", "make this look human", "remove AI tells",
  /code-humanize.
---

# Code Humanizer

Companion to **kyoko-humanize**. Same philosophy — strip AI performance, restore
human intent — but targets code AST instead of prose.

## Relation to Kyoko

- **Prose** → use `kyoko-humanize`
- **Code** → use this skill
- **Both** → run kyoko on comments/docs first, then this on the code structure

## Pipeline

Run stages in order. Each is independent — skip if not applicable.

---

### Stage 1 — `stripObviousComments`

Remove comments that restate the adjacent code.

**Remove:**
- Line comments that describe what the next line does: `// increment i`, `// return result`
- Empty TODOs with no body: `// TODO: handle error`
- Docblocks on trivial getters/setters with no params, no throws, obvious return
- Section dividers with no semantic meaning: `// ---- helpers ----`

**Keep:**
- Non-obvious intent: *why* a value is hardcoded, *why* an order matters
- Workarounds with ticket refs
- Public API docblocks (parameter contracts, throws, examples)

---

### Stage 2 — `collapseNeedlessHelpers`

Inline single-use extracted functions that add no abstraction value.

**Collapse when:**
- Function called exactly once
- Body ≤ 5 lines
- Not recursive, not async boundary, not a named test fixture
- Name doesn't encode a domain concept (just describes the lines it wraps)

**Keep when:**
- Called 2+ times
- Name is a domain term (`validateLead`, `buildQuery`)
- Extracted for testability (unit test references it directly)

---

### Stage 3 — `deflateNames`

Shorten names inflated with redundant type/role tokens.

| Pattern | Example | Fix |
|---------|---------|-----|
| Type in name | `userData` | `user` |
| Redundant noun | `getUserById` | `findUser` |
| Event suffix | `handleClickEvent` | `onClick` |
| Manager/Helper suffix | `UserDataManager` | `Users` or `UserStore` |
| Boolean prefix stutter | `isUserLoggedIn` | `isLoggedIn` (in User context) |

**Rule:** names should be as short as the call site makes unambiguous.

---

### Stage 4 — `removeDefensiveNoise`

Remove guards that aren't reachable given the actual data flow.

**Remove:**
- Null checks on values the type system guarantees non-null
- `try/catch` wrapping code that cannot throw in practice
- Unreachable `else` branches (only exists for visual symmetry)
- `|| null` / `|| undefined` on already-nullable values
- `=== undefined || === null` → replace with `== null` where appropriate

**Keep:**
- Guards at public API boundaries (external input)
- Checks where the type system doesn't reach (untyped legacy code)
- Error handling that logs or transforms (not silent swallowing)

---

### Stage 5 — `matchSurroundingStyle`

Infer style profile from context files and apply to output.

**Profile points to match:**
- Quote style: single vs double
- Semicolons: yes/no
- Trailing commas: ES5 / all / none
- Brace style: same-line vs new-line
- Spacing: `fn()` vs `fn ()`, `{x}` vs `{ x }`
- Arrow functions: `x => x` vs `(x) => x` vs `function(x)`

**Source:** look at 2–3 files the human owns in the same module. Never apply project
prettier config blindly — match what they *actually write*, which may diverge from config.

---

### Stage 6 — `reintroduceIntentionalSlop` *(hardest)*

Inject natural imperfection. Humans optimize for readability of *their mental model*,
not spec completeness.

**Transforms:**
- Drop redundant parens in conditions: `(x * 2)` → `x * 2` where precedence is obvious
- Collapse unnecessary ternary: `x ? true : false` → `!!x` or just `x` if boolean
- Drop `return undefined` / `return void` at end of void functions
- Remove braces on single-statement `if` when team style allows
- Use `arr.length` for truthiness instead of `arr.length > 0`
- Prefer `+=` / `-=` over `x = x +` 
- Consolidate chained `.filter().map()` only when the combined form reads cleaner

**Do NOT:**
- Introduce bugs for the sake of looking human
- Change semantics — only equivalent rewrites
- Apply all transforms — pick the ones that match the surrounding file's slop profile

---

## Confidence Scoring

After each stage, rate the output 0–100:

| Score | Meaning |
|-------|---------|
| 90–100 | Indistinguishable from senior human in this codebase |
| 70–89 | Mostly natural, minor AI tells remain |
| 50–69 | Improved but still reads AI-generated |
| < 50 | Still clearly AI — re-run pipeline |

Target: **≥ 85** before handing back.

## Adaptive Loop

- `max_passes`: 3
- `target_confidence`: 85
- `min_delta`: 5 (stop if improvement < 5 points two passes in a row)
- Rollback if Stage 6 changes semantics (revert that stage only)

## Limitations

Stage 6 (`reintroduceIntentionalSlop`) requires author fingerprint. Without access to
git history or surrounding human-written code, use conservative defaults — apply only
transforms that are universally safe, skip opinionated ones.

Full implementation of Stage 6 requires:
```
inferAuthorIntent(ast, gitBlame, surroundingCode) → styleFingerprint
```
This is a fine-tuned model problem: train on diffs between raw AI output and
human-edited versions of the same code. Not rule-based alone.
