# @snrgy/code-size

Objective codebase measurement with transparent methodology.

## Why this tool exists

"How big is the codebase?" is a surprisingly hard question to answer honestly. Tools that answer it tend to:

- Silently exclude blank lines, comments, or generated files without telling you
- Report a single number when a distribution is more informative
- Conflate LOC with complexity or quality (they correlate weakly)
- Show "health scores" that bake in editorial opinions about what constitutes a large file or too much duplication

This tool takes a different approach: it measures raw properties of a codebase and reports them with their full statistical distribution. It does not tell you whether the numbers are good or bad — that depends on your language, team, domain, and context.

Think of it like a caliper: it measures and reports. Whether 400 lines per file is fine or alarming is your call.

## Installation

```bash
npm install
```

No API keys. No cloud services. No subscriptions. Everything runs locally.

## Quick start

```bash
# Measure current directory
npx tsx src/index.ts

# Measure a specific path
npx tsx src/index.ts --path /path/to/project

# Include git growth analysis
npx tsx src/index.ts --git

# JSON output (for scripting)
npx tsx src/index.ts --json
```

Reports are automatically saved to `reports/<project-name>/<timestamp>.md` and `.json`.

## What gets measured

### File metrics

| Metric | What it measures |
| --- | --- |
| Logical lines (LOC) | Non-blank, non-comment lines |
| Blank lines | Empty or whitespace-only lines |
| Comment lines | Lines that are purely comments |
| Total lines | Raw line count including all of the above |
| File size (bytes) | On-disk size |

### Codebase totals

| Metric | What it measures |
| --- | --- |
| Total LOC | Sum of logical lines across all files |
| Total lines | Sum of raw lines across all files |
| File count | Number of source files analyzed |
| Excluded files | Files excluded, with count and reason |

### Statistical distributions

Rather than just reporting averages (easily skewed by outliers), every per-file metric reports:

- **P10 / P25 / P50 (median) / P75 / P90** — the full shape of the distribution
- **Mean** — provided for completeness but sensitive to outliers
- **Standard deviation** — spread of the data

A codebase where P90 file size is 800 LOC tells a very different story from one where P50 is 800 LOC, even if the totals are the same.

### Language breakdown

LOC, file count, and distributions **per language** (by file extension). Cross-language totals are reported but should be interpreted carefully — a 200-line Go file is not equivalent to a 200-line Python file.

### Directory breakdown

LOC and file count per top-level directory. Useful for finding where size concentrates.

### Largest files (ranked)

Top N files by LOC, ranked. No threshold is baked in as "too large" — you decide what warrants attention in your context.

### Git growth rate (optional, requires `--git`)

If the path is a git repository, the tool reads git log to compute:

- **LOC added and removed per week** over the last 12 weeks
- **Net growth per week** (added − removed)
- **Largest single-commit additions and removals** — useful for spotting one-time bulk imports

Growth rate is a leading indicator for projects that accumulate code without cleanup. A codebase growing at 2,000 net LOC/week is a different situation than one growing at 200.

**Note:** Git growth reflects *commits*, not edits. Squash merges, rebases, and history rewrites affect these numbers — the report notes when history anomalies are detected.

### Duplication (block-level)

Normalized block hashing to detect repeated code:

- **% of N-line blocks that appear more than once** across all files
- **Count of duplicate block pairs**
- **Largest duplicated blocks** (location + line count)

**Important:** Duplication is reported as a raw number. It is not labeled "bad". Some duplication is intentional (e.g., test fixtures, vendored code, generated repetitive patterns). Treat this as a signal to investigate, not a verdict.

## What is excluded — and why

Every report explicitly lists what was excluded and why. You should always know what the tool did not count.

| Excluded | Default | Reason | Override |
| --- | --- | --- | --- |
| `node_modules/` | Yes | Third-party dependency code | `--include-node-modules` |
| `dist/`, `build/`, `.next/`, `.cache/` | Yes | Build output — not source | `--include-generated` |
| `*.min.js`, `*.min.css` | Yes | Minified — LOC is meaningless | `--include-minified` |
| Binary files | Yes | Not text | Always excluded |
| `.git/` | Yes | Version control metadata | Always excluded |
| `.gitignore`-listed paths | Yes | What the project itself ignores | `--no-gitignore` |

Excluded file counts and total sizes are always reported. If 60% of your files were excluded, you should know that.

## Performance budgets

Define maximum thresholds for any metric. Exit code 1 on failure — designed for CI pipelines.

```json
{
  "totalLoc": 50000,
  "medianFileLoc": 200,
  "p90FileLoc": 500,
  "duplicationPercent": 15
}
```

```bash
npx tsx src/index.ts --budget budget.json
```

All values are compared against the **median** for distribution metrics.

## Known limitations

### Fundamental (can't be fixed)

- **LOC does not measure complexity.** A 500-line file of well-structured simple logic may be cleaner than a 100-line file of dense abstractions. LOC measures size, not quality.
- **Language norms differ widely.** Go, Java, and C tend toward more lines for equivalent logic than Python or Ruby. The tool reports per-language distributions to help with this, but cross-language comparison remains noisy.
- **Comment detection is heuristic.** The tool uses extension-based comment syntax rules (`//`, `#`, `/* */`, etc.). Mixed comment styles and non-standard comment markers may not be detected.
- **Duplication is syntactic, not semantic.** Two blocks that do the same thing written differently are not detected as duplicates.
- **Git growth rate is commit-based.** Squash merges can make one commit look like a massive addition. Rebases and history rewrites distort trend analysis.

### Design tradeoffs (intentional)

- **No scores.** We will never add a "maintainability index" or "health score". These require editorial decisions about what matters — that decision belongs to you.
- **No built-in thresholds.** The tool does not warn you when a file exceeds 200 lines by default. Use `--budget` if you want enforcement.
- **No "good" or "bad" labels.** Output is always neutral: numbers, distributions, and ranked lists. Interpretation is your responsibility.

## Methodology

Every report includes an auto-generated methodology section describing:

- Which files were included and excluded, and how that was determined
- How LOC is counted (what constitutes a blank line, comment line, logical line)
- How duplication is detected (block size, normalization applied)
- How git history is read and what the known threats to validity are
- Statistical methods used (sample statistics, percentile calculation method)

Transparency about methodology is not a footnote — it is part of the output.

## License

MIT
