# Reproduction guide

Everything below starts from a clean machine and reproduces the main result: the staged
evaluation of StepFree against the one-shot baseline on the 12-fixture corpus.

## What you need

| Requirement | Version used | Notes |
|---|---|---|
| Node.js | 22.x | any ≥ 20 should work |
| Chromium (via Playwright) | Playwright 1.62 | one command below installs it |
| Anthropic access | `ANTHROPIC_API_KEY` **or** a Claude Code login | the Agent SDK accepts either |
| OS | Linux x64 (tested) | macOS should work; Windows untested |

No other services, no databases, no accounts. All evaluation data (the fixture sites) is
synthetic and lives in this repository.

## Setup (one time, ~2 minutes)

```bash
git clone https://github.com/shi1720/stepfree.git
cd stepfree

cd stepfree && npm install && cd ..

# Browser for the scanner (skip if Chromium already provided via STEPFREE_CHROMIUM)
cd stepfree && npx playwright install chromium && cd ..

export ANTHROPIC_API_KEY=sk-ant-...   # or be logged into Claude Code
```

Sanity check the deterministic core (no API calls, ~30s):

```bash
node stepfree/eval/selftest.mjs        # contrast math, scanner, integrity checks
```

## Step 1 - verify the fixture ground truth (~3 min, no API calls)

```bash
node stepfree/eval/run-eval.mjs verify-fixtures
```

Expected output: one ✅ line per fixture and
`Ground truth locked: 134 WCAG instances across 12 fixtures.`
The evaluator refuses to run if any fixture's live scan disagrees with its recorded
ground truth, so results cannot drift silently.

## Step 2 - run one site end to end (the demo, ~5-8 min, ≈ $1)

```bash
node stepfree/src/cli.mjs fix fixtures/08-shelter --out runs/demo --stage final
```

Watch the console: routing → specialists → per-fix verification → review pass. Then open:

- `runs/demo/report.html` - the audit report (before/after evidence, review queue)
- `runs/demo/site/` - the fixed site (the original in `fixtures/` is never touched)
- `runs/demo/trajectories/*.md` - every agent's full trajectory, human-readable
- `runs/demo/run.json` - the machine-readable ledger

The baseline on the same site, for contrast (~1 min, ≈ $0.15):

```bash
node stepfree/baseline/run.mjs fixtures/08-shelter --out runs/demo-baseline
node stepfree/eval/run-eval.mjs measure --stage demo-baseline --only 08-shelter  # optional
```

## Step 3 - reproduce the full staged evaluation (~2.5-4 h, ≈ $30-60)

```bash
bash stepfree/eval/run-all-stages.sh 3      # 3 = fixtures fixed concurrently
```

This runs six configurations over all 12 fixtures - `baseline` (one direct prompt),
`v1` (naive agent), `v2` (+verification loop), `v3` (+specialists/knowledge/tools),
`v4` (+integrity guardrails & rollback), `final` (+memory, review lane, report) - and
measures each stage with an evaluator that is independent of the system under test
(it re-scans and re-screenshots every page itself; run ledgers are used only for
cost/time). Results land in:

- `stepfree/eval/results/<stage>.json` - full per-page measurements
- `stepfree/eval/results/summary.md` - the comparison tables (regenerate any time with
  `node stepfree/eval/run-eval.mjs tables`)

Any single stage can be run alone:

```bash
node stepfree/eval/run-eval.mjs run --stage v2 --concurrency 3
node stepfree/eval/run-eval.mjs measure --stage v2
```

## Step 4 - alt-text quality judging (~15 min/stage, ≈ $3/stage)

axe verifies an alt exists; it cannot verify it is true. This scores every
agent-written alt against the rendered image (vision judge, Claude Opus by default,
a different model tier than the fixer, to reduce same-model bias):

```bash
node stepfree/eval/judge-alts.mjs --stage baseline
node stepfree/eval/judge-alts.mjs --stage final
```

Output: `stepfree/eval/results/alts-<stage>.json` with a 0-2 score per image and the
distribution.

## Step 5 - the removed experiment (optional, ~10 min)

The changelog documents a strict "zero visual change" gate we built and removed
because it rejects legitimate fixes (contrast repairs must repaint pixels). Reproduce
its false rollbacks on one fixture:

```bash
STEPFREE_STRICT_VISUAL=1 node stepfree/src/cli.mjs fix fixtures/01-bakery --out runs/strict-test --stage v4
```

Expect rollbacks of contrast/heading fixes that the normal gate accepts.

## What to expect

Headline numbers from our run of record are in `EVALUATION.md` (per-stage tables are
auto-generated into `stepfree/eval/results/summary.md`). LLM outputs vary between runs;
violation counts, verification verdicts, and integrity checks are deterministic given
the same fixed page, so stage-to-stage ordering and the baseline gap reproduce robustly
even when individual numbers wiggle by a few instances.

## Determinism & environment notes

- The scanner (axe-core), contrast math, screenshots (fixed 1280×900 viewport, animations
  frozen, reduced motion), and integrity checks are fully deterministic.
- If you have a pre-installed Chromium, point `STEPFREE_CHROMIUM` at the binary and skip
  `playwright install` (the code auto-detects `/opt/pw-browsers/chromium` too).
- Approximate costs above were measured with `claude-sonnet-5` as the fixer model
  (override with `--model` or `STEPFREE_MODEL`) and are reported per run in each
  `run.json` (`totals.costUsd`).
- Agents run sandboxed: file access is denied outside the run's working copy, no shell,
  no network. The input site directory is never modified.
