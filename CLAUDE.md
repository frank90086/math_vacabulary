# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, zero-dependency English vocabulary quiz app for Traditional Chinese speakers (words 1–160 of a junior-high word list). Everything — data, CSS, and logic — lives in [index.html](index.html). No build step, no package manager, no tests, no framework.

Despite the repo name (`math_vacabulary`), the content is English vocabulary, not math.

## Running it

Open the file directly:

```bash
open index.html
```

Or serve it if you need a real origin:

```bash
python3 -m http.server 8000
```

There is nothing to build, lint, or install. Verifying a change means loading the page and running a quiz.

## Structure of index.html

Three sections, in order:
1. `<style>` (lines ~7–78) — all CSS, driven by CSS custom properties on `:root` (`--paper`, `--ink`, `--hl` yellow highlight, `--ok`/`--no` for graded states). Mobile-first, max-width 640px, fixed bottom action bar.
2. `<body>` markup — a static `#setup` panel (type chips + question-count buttons) and an empty `#quiz` container that JS fills in.
3. `<script>` (lines ~125–451) — the `WORDS` data array followed by ~180 lines of plain DOM code.

### The WORDS data model

Each entry drives every question type, so all five fields matter:

```js
{no:1, en:"abroad", zh:"[副]在國外", keys:["在國外","國外","出國"], ex:["My sister is studying {}.", ...]}
```

- `zh` — display gloss, includes a part-of-speech tag like `[名]`/`[動]`; shown as the answer, never string-matched.
- `keys` — accepted Chinese answers for 英翻中 grading. Grading compares against `keys` only, so a missing synonym marks a correct answer wrong. Add generously.
- `ex` — 5 example sentences, each containing exactly one `{}` placeholder where the target word goes. Used by the cloze type; `{}` is replaced via `String.replace`, which substitutes only the **first** occurrence.

When adding words, keep `no` unique — `choice` distractor selection filters by `x.no !== w.no`. `no` is also the unit the range picker slices on, so keep the array in `no` order and contiguous.

### Range picker

`#rangeRow` offers preset spans plus 自訂 (two number inputs). Selection lands in the globals `rangeFrom`/`rangeTo`; `inRange()` is the only word source `start()` reads.

- `MAX_NO` is derived from the data (`WORDS.reduce`), so adding words extends 自訂's upper bound and the 全部 preset automatically. The preset buttons' `data-from`/`data-to` are hardcoded in markup — add a new preset row when the list grows past 160.
- `readCustom()` clamps to 1..`MAX_NO` and swaps reversed input, so the range is always non-empty; `updateHint()` writes the live "共 N 個單字，這次出 M 題" line and disables `#startBtn` on an empty range (currently unreachable, kept as a guard).
- The 題數 button `data-n="0"` means "all in range" — `count === 0` is a sentinel, not a quantity. Any other `count` is capped at the range size.
- `choice` distractors are drawn from the range first, falling back to all of `WORDS` when the range holds fewer than 4 words, so options stay at 4.

### Question types

`picked` (the five toggles) + `count` + the `rangeFrom`/`rangeTo` number range feed `start()`, which samples words via `inRange()` and assigns each a random enabled type:

| type | prompt | grading in `isRight()` |
|---|---|---|
| `choice` | random direction en→zh or zh→en, 4 options | exact match on stored option string |
| `spell` | zh + a partially masked word from `makeMask()` | `norm(input) === norm(w.en)` |
| `zh2en` | zh gloss | same as spell |
| `en2zh` | en word | any of `w.keys` matches after `norm()` |
| `cloze` | one random `ex` sentence with a blank | same as spell |

`cloze` silently falls back to `spell` when a word has no `ex`.

`makeMask()` picks one of four reveal strategies at random (first letter, first+last, vowels, ~45% random), with guards so it never reveals nothing or everything.

`norm()` is the only normalizer: lowercases and strips whitespace and CJK/ASCII punctuation. It does not strip articles or handle multi-word entries specially (e.g. `"French fries"` — spaces are removed, so `"frenchfries"` matches).

### App lifecycle

`start()` → `render()` → user answers → `check()` → `updateScore()` → `reset()` or `start()` again.

Two things to know when editing this flow:
- The bottom bar's buttons are **replaced by innerHTML** in `checkBar()` (pre-grading) and at the end of `check()` (post-grading), so listeners must be re-attached after every swap. The initial listeners bound at lines ~269–271 are discarded on the first `checkBar()` call.
- `graded` is a global flag that gates option clicks; `check()` also sets `pointerEvents:none` and `readOnly` on inputs. State lives entirely in module-scope globals (`picked`, `count`, `questions`, `graded`) — nothing is persisted, so a refresh wipes progress.

### Conventions

- All user-facing strings are Traditional Chinese and hardcoded inline. Match the existing terse tone.
- Any word/user data interpolated into HTML goes through `esc()`. Keep that up when adding new render paths.
- Style is deliberately compact: single-line functions, minimal spacing, no semicolonless lines. Match it rather than reformatting.
