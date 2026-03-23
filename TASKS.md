# code-size — Task Map

Status markers: `[ ]` todo · `[~]` in progress · `[x]` done

---

## Phase 1 — Project setup

- [ ] Install dependencies: `commander`, `tsx`, `typescript`
- [ ] `tsconfig.json` (ESM, strict, same pattern as web-speed-test)
- [ ] `src/index.ts` — entry point, CLI wiring with commander
- [ ] Verify: `npx tsx src/index.ts --help` works

---

## Phase 2 — File discovery

Goal: given a root path, return a list of files to analyze, plus a separate list of excluded files with reasons.

- [ ] Walk the directory tree recursively
- [ ] Respect `.gitignore` by default (use `git ls-files` for tracked files + fall back to manual walk)
- [ ] Exclusion rules (each category tracked separately):
  - [ ] `node_modules/`
  - [ ] `dist/`, `build/`, `.next/`, `.cache/`, `.output/`
  - [ ] `*.min.js`, `*.min.css` (minified — LOC is meaningless)
  - [ ] Binary files (detect via null-byte check on first 512 bytes)
  - [ ] `.git/`
- [ ] CLI flags to override defaults:
  - [ ] `--include-node-modules`
  - [ ] `--include-generated`
  - [ ] `--include-minified`
  - [ ] `--no-gitignore`
- [ ] Output from this phase: `{ included: File[], excluded: ExcludedFile[] }`
  - `ExcludedFile` carries `path` and `reason` (which rule matched)
- [ ] Unit tests: exclusion rules apply correctly, overrides work

---

## Phase 3 — Per-file line classification

Goal: for each file, count logical lines (LOC), blank lines, comment lines, total lines, and bytes.

**Line types:**
- `blank` — empty or whitespace-only
- `comment` — line that is purely a comment (after trimming). Block comment lines each count individually.
- `logical` (LOC) — everything else

**Comment syntax rules by extension (implement as a lookup table):**

| Extensions | Line comment | Block comment |
|---|---|---|
| `.ts` `.tsx` `.js` `.jsx` `.mjs` `.cjs` | `//` | `/* */` |
| `.css` `.scss` `.sass` `.less` | — | `/* */` |
| `.py` | `#` | `"""` (docstrings — treat as block comment) |
| `.rb` | `#` | `=begin / =end` |
| `.go` `.c` `.cpp` `.h` `.java` `.rs` `.swift` `.kt` | `//` | `/* */` |
| `.sh` `.bash` `.zsh` `.yml` `.yaml` `.toml` | `#` | — |
| `.html` `.xml` `.svg` | — | `<!-- -->` |
| `.sql` | `--` | `/* */` |
| (unknown extension) | — | — (count all as logical) |

**Important:** blank lines inside block comments count as `comment`, not `blank`. Track block comment state across lines.

- [ ] Implement `classifyFile(path): FileMetrics` — reads file, classifies each line
- [ ] Handle UTF-8 encoding; skip files that fail to decode (report as excluded: "encoding error")
- [ ] Unit tests for each language's comment rules
- [ ] Unit test: block comment spanning multiple lines (each line → comment)
- [ ] Unit test: inline comment after code (`x = 1; // comment`) → logical, not comment
- [ ] Benchmark: should handle 10k+ files without meaningful delay

---

## Phase 4 — Aggregation and distributions

Goal: from a list of `FileMetrics`, compute codebase-level statistics.

**Percentile calculation:** use linear interpolation (same method as web-speed-test).

- [ ] Codebase totals:
  - [ ] Total LOC, total blank, total comment, total lines, total bytes
  - [ ] File count
- [ ] Per-metric distributions (P10/P25/P50/P75/P90 + mean + stddev):
  - [ ] LOC per file
  - [ ] Total lines per file
  - [ ] File size in bytes
- [ ] Language breakdown:
  - [ ] Group files by extension
  - [ ] Per-language: file count, total LOC, LOC distribution (P50/P90)
  - [ ] Sort languages by total LOC descending
- [ ] Directory breakdown:
  - [ ] Group by top-level directory (first path segment relative to root)
  - [ ] Per-directory: file count, total LOC
  - [ ] Sort by total LOC descending
- [ ] Top N files by LOC (default: top 20, configurable via `--top-files N`)
- [ ] Excluded files summary:
  - [ ] Count per exclusion reason
  - [ ] Total excluded file count and estimated byte size

---

## Phase 5 — Git growth analysis (`--git` flag)

Goal: from git history, compute LOC growth over time.

**Implementation:** parse `git log --numstat --format="%H %ai"` output.

- [ ] Detect if the target path is inside a git repository (walk up looking for `.git/`)
- [ ] If not a git repo, `--git` prints a clear warning and skips (no crash)
- [ ] Parse `git log --numstat` output into per-commit records: `{ hash, date, added, removed }`
- [ ] Bucket commits into weeks (ISO week, last 12 weeks)
- [ ] Per-week: LOC added, LOC removed, net change
- [ ] Largest single-commit additions (top 5, with hash + date + message first line)
- [ ] Largest single-commit removals (top 5)
- [ ] Anomaly detection: flag weeks/commits with unusually large changes (>3× the median weekly change) — labeled as potential squash merge or bulk import. Not removed, just flagged.
- [ ] Note in methodology: "Growth rate is commit-based. Squash merges, rebases, and history rewrites affect accuracy."

---

## Phase 6 — Duplication detection

Goal: detect repeated N-line blocks across files. Report as raw numbers, no verdicts.

**Algorithm:**
1. For each file, extract all sliding windows of N consecutive non-blank lines (default N=6)
2. Normalize each window: trim leading/trailing whitespace per line, collapse internal whitespace runs to single space
3. Hash each normalized window (SHA-256, truncated to first 16 chars is sufficient)
4. Build a map: `hash → [{ file, startLine }]`
5. Entries with 2+ occurrences = duplicate blocks

**Metrics to report:**
- Total blocks analyzed
- Duplicate block count (blocks with ≥2 occurrences)
- Duplication % = duplicate blocks / total blocks × 100
- Top 10 most-duplicated blocks (hash, occurrence count, first location, all other locations)

- [ ] Implement with configurable block size (`--dup-block-size`, default 6)
- [ ] Skip files under `--dup-min-lines N` threshold (default: skip files with < N lines — too short to produce meaningful blocks)
- [ ] Unit tests: exact duplicate detected, near-duplicate (different whitespace) detected after normalization, different code not flagged
- [ ] Performance: should handle 50k+ blocks without hanging (use streaming per-file, not load all into memory at once)
- [ ] Note in methodology: "Duplication is syntactic, not semantic. Whitespace differences are normalized. Logically equivalent code written differently is not detected."

---

## Phase 7 — Output

### Markdown report

Structure (same philosophy as web-speed-test):

```
# code-size report — <project name>
<timestamp> · <file count> files · <total LOC> LOC

## Overview
<3-line TL;DR: file count, total LOC, median file size, P90 file size>

## Codebase totals
...

## File size distribution
...

## Language breakdown
...

## Directory breakdown
...

## Largest files
...

## Git growth (last 12 weeks)   ← only if --git
...

## Duplication
...

## Excluded files
...

## Methodology
<auto-generated from actual config used>

## Threats to validity
<static list from README, included in every report>
```

- [ ] Implement markdown report generator
- [ ] Auto-save to `reports/<project-name>/<timestamp>.md`
- [ ] `<project-name>` = last path segment of analyzed directory

### JSON report

- [ ] Full raw data: per-file metrics, all aggregates, git data, duplication details
- [ ] Save alongside markdown: `reports/<project-name>/<timestamp>.json`
- [ ] `--json` flag: write JSON to stdout, informational output to stderr

### Auto-generated methodology section

Must reflect the actual config used for that run — not a static template.

- [ ] Exclusion rules that were active
- [ ] Block size used for duplication
- [ ] Whether `.gitignore` was respected
- [ ] Git history depth (how many weeks)
- [ ] Percentile calculation method
- [ ] Comment detection: note that detection is heuristic and list which extensions have rules

---

## Phase 8 — Performance budgets (`--budget`)

- [ ] Load budget JSON file
- [ ] Available budget keys:
  - `totalLoc` — total LOC across all files
  - `medianFileLoc` — P50 LOC per file
  - `p90FileLoc` — P90 LOC per file
  - `totalFiles` — total file count
  - `duplicationPercent` — duplication %
  - `weeklyGrowthLoc` — latest week's net LOC growth (requires `--git`)
- [ ] Compare each budget key against actual value
- [ ] Print budget results in report (PASS / FAIL per key)
- [ ] Exit code 1 if any budget is exceeded
- [ ] Exit code 0 if no budget file or all budgets pass

---

## Phase 9 — End-to-end validation

- [ ] Run against `web-speed-test` repo — verify output makes sense
- [ ] Run against `code-size` repo itself
- [ ] Run against a large open-source repo (e.g. `express` or `chalk`) to test performance
- [ ] Verify excluded files report is accurate (spot-check manually)
- [ ] Verify git growth chart is accurate (compare against `git log --stat` manually)
- [ ] Review README against actual implementation — update anything that diverged

---

## Phase 10 — Polish

- [ ] `--methodology` flag: print methodology text without running analysis
- [ ] Sensible error messages for common failures (path doesn't exist, not a directory, git not available)
- [ ] `--help` output is clear and matches README
- [ ] Handle symlinks gracefully (don't follow into infinite loops)
- [ ] Handle very large files (>10MB) — warn and skip rather than crash

---

## Open questions (decide before implementing)

- [ ] Should we support a `--watch` mode that re-runs on file change? (Probably not for v1 — keep scope tight)
- [ ] For git growth, should we analyze the whole history or cap at N weeks? (Proposal: cap at 52 weeks default, configurable)
- [ ] For duplication: normalize identifier names? (e.g. `foo` and `bar` treated as same if structure is identical) — probably too complex for v1, and would require AST, not hashing. Defer.
- [ ] Should we support non-git project comparison (before/after snapshots)? Defer to v2.
