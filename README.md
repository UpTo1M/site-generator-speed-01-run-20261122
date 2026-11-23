# Fast makefun-site

[![make.fun](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/UpTo1M/site-generator-speed-01-run-20261122/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/UpTo1M/site-generator-speed-01-run-20261122/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)

A faster drop-in for [makefun-site 1.0.0](https://make.fun/bounty/site-generator-speed-01-run-20261122),
the reference static site generator of the
[“Build a static site faster, 22 November 2026” bounty](https://make.fun/bounty/site-generator-speed-01-run-20261122):
`solve.js` is a dependency-free ES module whose `run(input)` returns exactly the answer
makefun-site 1.0.0's `build` returns for the same input — the output tree as a JSON object of
every path it writes (each page's `index.html`, the index and tag pages, `build-report.txt` and
each static file) and that file's text, byte for byte. `main.ts` is the program the harness
runs: it reads every case from `/input/cases.json`, answers each one alone with `run`, and
writes `/output/answers.json`.

## How the speedup works

The reference is a faithful reimplementation of makefun-site's own rules (front matter,
template dialect, Markdown subset, slug rule, report — the kit's `vendor/makefun-site/`
code is the reference wherever its README differs), written to avoid the reference's
repeated work:

- **Inline Markdown without a per-character loop.** Text with no markup character at all
  is just escaped (one scan decides that). Otherwise the next markup character is found
  with `indexOf` scans and plain runs are escaped in bulk, instead of one regex per
  character. Escapes pair left to right, exactly as the reference does — including that
  `"` is *not* an escapable character, so `\"` stays a backslash and a quote.
- **One fused Markdown pass.** Lexing and rendering happen in a single pass over the
  body's lines — no token objects in between — and each line is dispatched on its first
  character, so a plain text line runs none of the block checks.
- **Hand-rolled front matter.** The front-matter region and the body are split separately
  (the whole text is never split at once), keys and values are parsed by hand, and the body
  stays as lines so the renderer never re-splits it.
- **Allocation-free word counting.** Words are counted with a whitespace table (one indexed
  read per character) instead of a regex match that builds an array of every word.
- **Cheaper slugs and dates.** Slugs take an ASCII fast path (NFKD is the identity on
  ASCII), long dates are read with slices instead of a split, and a post's card is built
  once and shared between its list entry and its page.
- **One shared output array for templates.** Rendering writes into a single array per page
  instead of concatenating a string per node and branch.

Nothing is cached from one case to the next: every `run(input)` parses, renders and builds
its case alone, as the rules require.

## Practice-set timing

Measured on the kit's practice set (3 cases, 250–480 Markdown sources each) with
`process.cpuUsage`, 10 repeats, Node on this machine; baseline is the house entry
(`baselines/makefun-site/`, makefun-site 1.0.0 behind the same driver).

| case | baseline (CPU ms, mean) | this entry (CPU ms, mean) | speedup |
| ---- | ----------------------: | ------------------------: | ------: |
| g-0001 | 137.1 | 38.4 | 3.57× |
| g-0002 | 204.1 | 78.6 | 2.60× |
| g-0003 | 259.0 | 74.8 | 3.46× |
| **total** | **600.2** | **191.8** | **3.13×** |

Outputs are byte-identical to the baseline on every practice case, and identical to the
kit's `practice/reference/expected.json`.

## Files

- `solve.js` — the fast module: `run(input)` returns makefun-site 1.0.0's answer.
- `main.ts` — the driver the harness runs (`node /bot/main.ts`, Node 22, no installs).
- `LICENSE` — MIT (`SPDX-License-Identifier: MIT`).
