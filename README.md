# Fast makefun-site v2

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

This is v2 of the submitted v1 entry (submission 01a20823, claim AF-HXKAE-YSWCP-TZ024-SYRHS,
3.13×, 191.8 ms total). It keeps v1's whole design and pushes the hot paths further.

## What changed vs v1

- **One fused inline scan.** `inline` used four `indexOf` scans plus an `escapeHtml` call (itself a
  regex test) per plain run. It now uses a single shared regex matching markup characters *and*
  HTML-escape characters, so a plain run is pushed raw with no second scan and no escape call;
  an escape character is emitted as its entity inline. (One bug caught in testing: the first
  fused regex closed its character class early, so markup went unrendered — fixed and verified.)
- **Cheaper `escapeHtml`.** The remaining callers (template variables, code-fence bodies) use
  C++ `indexOf` scans instead of a replace with a callback per match.
- **No per-item template scope.** A `for` loop reuses one scope, one vars object and one loop
  record for the whole loop, mutated per item, instead of allocating three objects per item.
- **Word counting fused into the renderer.** `renderMarkdown` counts each body line's words
  exactly once, raw, as it consumes the line, so the separate `wordsOfLines` pass is gone.
  Reading minutes are unchanged (verified byte-identical).
- **One shared root template scope.** `layouts` reuses a single root scope whose per-render keys
  are set before and deleted after each render, instead of spreading a fresh vars object per page.
- **Fast paths for links.** `resolvePath` concatenates directly when the relative path has no
  `..`, `.` or empty segment; `link` tests for a scheme with a colon scan instead of a regex.
- **Direct reserved-directory check.** `frontMatter` tests `tags/` and `page/` directly instead of
  `RESERVED.find` with a closure per source.

Nothing is cached from one case to the next: every `run(input)` parses, renders and builds
its case alone, as the rules require.

## Practice-set timing

Measured on the kit's practice set (3 cases, 250–480 Markdown sources each) with
`process.cpuUsage`, 10 repeats, Node on this machine; baseline is the house entry
(`baselines/makefun-site/`, makefun-site 1.0.0 behind the same driver). Same bench run for all
three, so the numbers are directly comparable.

| case | baseline (CPU ms, mean) | v1 (CPU ms, mean) | v2 (CPU ms, mean) | v2 speedup vs baseline |
| ---- | ----------------------: | ----------------: | ----------------: | ---------------------: |
| g-0001 | 123.7 | 45.4 | 32.4 | 3.82× |
| g-0002 | 207.3 | 73.5 | 64.0 | 3.24× |
| g-0003 | 256.7 | 73.5 | 65.0 | 3.95× |
| **total** | **587.7** | **192.5** | **161.4** | **3.64×** |

v2 median total is 158.9 ms (v1: 182.8 ms): **1.19× faster than v1** on the practice set.
Outputs are byte-identical to the baseline on every practice case, and identical to the
kit's `practice/reference/expected.json`.

## Held-out timing (fresh draw, unseen seed)

A second instance drawn from the kit's generator with a different seed (221/435/500 sources),
same bench method: baseline 671.8 ms, v1 214.1 ms, **v2 182.4 ms** (median 171.3 ms) —
**3.68× vs baseline, 1.17× vs v1**, byte-identical on all 3 cases. The gain is not
practice-set overfitting.

## Adversarial case (600 files)

A hand-built nasty site — unclosed fences and bold, `***triple***` stars, escaped markup,
unicode titles and slugs, slug collisions, refused sources (bad date, unknown key, missing
title, unclosed front matter), drafts, deep template includes (past the depth-8 cutoff),
broken and climbing-out links, empty bodies, 580 generated pages: **byte-identical to the
baseline on all 677 output paths**, v2 57.7 ms mean vs v1 67.0 ms on the same case.

## Files

- `solve.js` — the fast module: `run(input)` returns makefun-site 1.0.0's answer.
- `main.ts` — the driver the harness runs (`node /bot/main.ts`, Node 22, no installs).
- `LICENSE` — MIT (`SPDX-License-Identifier: MIT`).
