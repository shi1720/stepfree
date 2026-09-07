# StepFree

**Accessibility repairs, with evidence.**

StepFree takes a static website, proposes source-code accessibility fixes, and checks whether the changes actually helped. Independent scans and content-integrity checks decide whether to accept a repair, retry it or roll it back. Judgment calls go into a human review queue with drafted changes.

[![Repository quality](https://github.com/shi1720/stepfree/actions/workflows/repository-quality.yml/badge.svg)](https://github.com/shi1720/stepfree/actions/workflows/repository-quality.yml)

[Architecture](docs/architecture.md) · [Recorded evaluation](EVALUATION.md) · [Reproduce the experiments](REPRODUCTION.md) · [Change history](CHANGELOG.md)

![Example evidence report](docs/img/sample-report-top.png)

## Why it exists

A scanner can identify an accessibility violation. A coding model can suggest a repair. Neither step alone establishes that useful content was preserved or that a person can now use the page. StepFree makes verification and review part of the repair workflow.

## Workflow

```text
Source copy → axe scan → specialist repair agents → independent verification
                                                     ├─ accept
                                                     ├─ retry / roll back
                                                     └─ human review
             fixed source + evidence report ←──────────────┘
```

The specialist paths handle document structure, images, forms, ARIA and contrast. Image descriptions require inspecting the actual image; contrast changes use luminance calculations. The report collects before/after evidence, scan results, content checks and unresolved review items.

## Run the offline checks

Use Node.js 22 or newer. From a clean checkout:

```bash
git clone https://github.com/shi1720/stepfree.git
cd stepfree
npm ci --prefix stepfree
cd stepfree
npx playwright install chromium
npm test
cd ..
```

The self-test checks contrast math, scanner deltas, content-loss detection and unchanged-page integrity. It runs a browser against synthetic local files and makes no paid model calls. GitHub Actions runs the same deterministic checks.

## Try a repair

From the repository root after setup:

```bash
export ANTHROPIC_API_KEY='your-provider-key'
node stepfree/src/cli.mjs fix fixtures/08-shelter --out runs/demo
```

Open `runs/demo/report.html` to inspect the result. Repairs use an external model, require current provider access and incur charges. Start with the bundled synthetic fixtures; preserve an untouched copy of any source you supply. See [REPRODUCTION.md](REPRODUCTION.md) for the complete experiment commands and configuration.

## What the recorded evaluation shows

The committed evaluation covers **12 synthetic sites and 13 pages**. Both the one-shot baseline and StepFree repaired the scanner-visible violation instances in that corpus. The useful distinctions were in image grounding, preservation of informative content, evidence generation and the review path for issues a scanner cannot resolve.

These are recorded experiment results, not a guarantee for arbitrary websites. The [evaluation report](EVALUATION.md) includes the raw measurements, failure examples, judge disagreements and limitations. [Research notes](docs/research/prior-art.md) explain the prior work and design rationale.

## Scope and limits

- The fixture corpus consists of small static sites. Complex authenticated applications need separate integration and evaluation work.
- Automated checks cover only part of accessibility. Human testing remains necessary; this is not a compliance certification.
- A screenshot difference can reflect harmless layout movement. The [change history](CHANGELOG.md) explains why a strict pixel-area gate was removed.
- Model-based judgments are treated as evidence to inspect, not as an independent correctness oracle.

## Repository map

| Path | Contents |
| --- | --- |
| [`stepfree/src/`](stepfree/src/) | Orchestrator, specialist agents, tools and report generation |
| [`stepfree/eval/`](stepfree/eval/) | Deterministic checks, evaluation runners and recorded results |
| [`fixtures/`](fixtures/) | Synthetic sites and ground-truth notes |
| [`trajectories/`](trajectories/) | Representative agent traces |
| [`reports/`](reports/) | Example report artifacts |
| [`docs/architecture.md`](docs/architecture.md) | Component boundaries and verification design |

## Authorship and contribution

Built by **Shivam Gupta** for the micro1 Agentic Workflows Hackathon, with AI-assisted implementation and iterative evaluation. Quantitative results come from the committed evaluation artifacts; the project does not claim that a model's own assessment proves correctness.

Use synthetic examples in issues and pull requests. Run `npm test` from `stepfree/` after changes to the deterministic core. Report security issues through the account's [private reporting process](https://github.com/shi1720/.github/blob/main/SECURITY.md).

Licensed under [MIT](LICENSE). Preserve upstream attribution and the distinction between recorded measurements and claims that have not been evaluated.
