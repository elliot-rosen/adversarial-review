<h1 align="center">/adversarial-review</h1>

<p align="center">
  <a href="https://agentskills.io"
    ><img
      alt="Agent Skills"
      src="https://img.shields.io/badge/Agent%20Skills-package-blue?style=flat-square"
  /></a>
  <a href="LICENSE"
    ><img
      alt="License"
      src="https://img.shields.io/badge/license-MIT-green?style=flat-square"
  /></a>
  <a href="https://github.com/elliot-rosen"
    ><img
      alt="GitHub"
      src="https://img.shields.io/badge/GitHub-@elliot--rosen-black?style=flat-square"
  /></a>
</p>

<h3 align="center">Your PR gets a prosecutor. Your architecture gets a challenger. Only evidence survives.</h3>

Ask an agent to review your code and you get agreeable mush: "looks good, consider adding a comment here." Ask it to be critical and you get the opposite failure — a wall of nitpicks with no sense of what actually matters.

**adversarial-review** is an [Agent Skill](https://agentskills.io) that replaces the single reviewer with a structured debate. Committed opposing agents — one mandated to block your PR, one mandated to defend it — build their strongest cases in isolation, then tear each other's cases apart. A judge rules for the strongest surviving case. No averaging, no split-the-difference, no "it depends."

- **Committed opponents** — each debater argues its assigned verdict as hard as the evidence allows. A challenger that won't commit to a concrete alternative forfeits.
- **Evidence hierarchy** — reproduced behavior beats code reading beats fresh documentation beats best practices beats opinion. Unsupported claims lose to silence.
- **Nitpick ban** — style, naming taste, and hypothetical inputs are struck outright. Every finding must name the concrete failure it causes.
- **One ruthless verdict** — `MERGE` / `BLOCK`, `ADOPT` / `REJECT`, `BUILD` / `KILL` / `CUT TO <scope>`. The judge must name why every losing argument lost.

## Quick Start

```sh
# install (global recommended)
$ npx skills add elliot-rosen/adversarial-review -g
# or: copy skills/adversarial-review → ~/.claude/skills/adversarial-review

# in your agent
/adversarial-review the diff on this branch
```

Works on three review types:

| Type | Debaters | Verdict |
| ---- | -------- | ------- |
| **PR review** | Prosecution vs Defense (+ optional specialist prosecutor for security / data / perf) | `MERGE` or `BLOCK (n findings)` |
| **Architectural review** | Champion vs Challenger — the Challenger must commit to a concrete alternative, including "do nothing" | `ADOPT` or `REJECT — adopt <alternative>` |
| **Feature review** | Advocate vs Executioner (+ optional Simplifier who must name the exact smaller cut) | `BUILD`, `KILL`, or `CUT TO <scope>` |

Example report shape (illustrative — structure is what the skill produces):

```markdown
## Verdict: BLOCK (2 findings)

Strongest reason: the retry path re-enters the payment charge without an
idempotency key — reproduced double-charge in the test below.

## Surviving findings

| Sev | Finding | Evidence | Impact | Required action |
|-----|---------|----------|--------|-----------------|
| P0 | Retry re-charges without idempotency key | `charge.ts:88`; repro test attached | Double-billing on network flake | Pass `Idempotency-Key` from the original attempt |
| P1 | Migration drops `NOT NULL` before backfill completes | `0042_alter.sql:12` vs backfill job ordering | Window of NULL rows breaks the report query | Reorder: backfill → constraint |

## Killed in debate

- Prosecution's "N+1 query regression" — Defense reproduced the query plan:
  the loop is bounded at 3 by `PLAN_LIMIT` (`plans.ts:14`). Died in Round 2.
- Defense's "covered by integration tests" — the cited suite never constructs
  a failed-then-retried charge; no test reaches the failing state.

## Unresolved

- p95 latency impact of the new middleware: needs a staging measurement;
  neither side had numbers.
```

The **Killed in debate** section is how you spot-check that the debate was real — every dead argument ships with the evidence that killed it.

## How It Works

```
artifact (diff · design doc · feature spec)
  │
  ▼
┌──────────────────┐
│ evidence pack    │  diff/doc + surrounding code + tests + fresh docs
└────────┬─────────┘
         ▼
┌──────────────────┐
│ round 1          │  committed cases, built blind (parallel)
│ prosecution ▲    │  every claim cited: file:line, test, doc URL
│ defense     ▼    │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ round 2          │  cross-examination (parallel)
│ attack · refute  │  challenged + undefended = dead
│ concede honestly │
└────────┬─────────┘
         ▼
┌──────────────────┐
│ judge            │  rules by evidence hierarchy — no averaging
└────────┬─────────┘
         ▼
      verdict + report
```

- **Two rounds maximum.** If one factual dispute still decides the outcome, it's settled by direct verification — read the code, run the test, fetch the doc — not more debate.
- **Deadlock rule:** missing ground truth (no tests, no data, no docs) resolves to the conservative verdict, and the report says exactly what evidence would flip it.
- **Review-only:** the skill reports; it never changes code until you approve a remediation plan.

## The Evidence Hierarchy

When claims collide, higher evidence wins; equal tiers go to the more specific claim.

1. Reproduced behavior — a failing test, executed repro, or measured number
2. Direct reading of the actual code/diff/config with `file:line` citations
3. Authoritative *current* documentation, release notes, advisories — searched fresh, not recalled
4. Established best practice with a citable source and an argument for why it applies *here*
5. Unsupported opinion, analogy, "common knowledge" — loses to everything, including silence

## The Nitpick Ban

A finding qualifies only if it materially affects correctness, security, data integrity, money paths, API compatibility, performance at realistic scale, operational cost, or defect-causing complexity. Banned regardless of merit: style, formatting, naming taste, "I would have structured this differently," speculative flexibility, and error handling for impossible states. A debater who pads its case with nitpicks has those points struck — and the rest of its case discounted.

## Usage

| Invoke | Example |
| ------ | ------- |
| Slash | `/adversarial-review PR #142` |
| Natural language | "red-team this architecture proposal" |
| Natural language | "debate whether we should build the referral feature" |

The agent should auto-invoke when the description matches: adversarial review, red-team review, debate review, or "judge this brutally on evidence."

## Harness notes

Built for harnesses with parallel subagents (Claude Code, Codex, and friends) — Round 1 and Round 2 fan out concurrently. On harnesses without subagents the skill falls back to strictly separated sequential passes: each role's case is written in full before cross-examination starts, and no role's conclusions leak into another's case-building.

## Package layout

```
skills/adversarial-review/
  SKILL.md                 # agent contract: workflow, roles, hierarchy, ban, output format
  agents/
    openai.yaml            # Codex interface metadata
```

Ships as an [`npx skills`](https://github.com/vercel-labs/skills) package (`skills/<name>/SKILL.md`) for install convenience.

## Development

```sh
# edit the skill
$EDITOR skills/adversarial-review/SKILL.md

# smoke-test discovery locally
npx skills add ./ -l

# install from this checkout
npx skills add ./ -g -y
# or symlink into ~/.claude/skills/adversarial-review
```

## License

[MIT](LICENSE)
