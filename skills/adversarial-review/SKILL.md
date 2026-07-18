---
name: adversarial-review
description: Adversarial multi-agent debate for PR reviews, architectural reviews, and feature reviews. Committed opposing agents attack each other's cases across bounded rounds; only evidence-backed conclusions survive to a single ruthless verdict. Use when the user invokes $adversarial-review, asks for an adversarial, red-team, or debate review, or wants a PR, architecture, or feature judged brutally on evidence, best practices, and current data without nitpicking.
---

# Adversarial Review

## Overview

Put a PR, architecture, or feature through a structured debate between agents with opposing mandates. Debaters argue their assigned verdict as hard as the evidence allows, then tear apart each other's cases. The judge rules for the strongest surviving case — never a compromise. Brutal on substance, silent on style: nitpicks are banned outright.

This is a review skill. Report the verdict and findings; do not change code until the user approves a remediation plan.

## Workflow

1. **Scope.** Identify the review type (PR, architecture, feature) and the exact artifact: diff or branch, design doc or proposal, feature spec or implementation. Confirm scope only if genuinely ambiguous.
2. **Evidence pack.** Assemble the shared ground truth all debaters receive: the diff/doc/spec itself, the surrounding code it touches, tests, relevant metrics or analytics, and — when currency matters (framework APIs, platform policies, pricing, security advisories) — current documentation via search. Debaters argue from the same pack plus whatever they dig up themselves; every claim must cite its source.
3. **Round 1 — committed cases (parallel).** Spawn the debaters for the review type (see Roles). Each receives the evidence pack, its mandate, the Evidence Hierarchy, and the Nitpick Ban. Each must build the strongest possible case for its assigned verdict — steelmanned, not strawmanned — with every claim cited (file:line, test output, doc URL, data query). Debaters do not see each other in this round.
4. **Round 2 — cross-examination (parallel).** Give each debater the full opposing case(s). Mandate: dismantle them. Attack every claim's evidence, reproduce or refute disputed behavior, find counter-examples in the actual code/data/docs, expose reasoning that outruns its citations. A claim that is challenged and not defended with evidence is dead. Concede points honestly — a debater that defends the indefensible forfeits credibility on its surviving points.
5. **Verdict.** A judge (fresh agent, or you) rules on what survived. The judge scores arguments strictly by the Evidence Hierarchy, may not average or split the difference, and must name the winning case and why each losing argument lost. If one factual dispute still decides the outcome, resolve it by direct verification — read the code, run the test, fetch the doc — not by more debate. Two debate rounds maximum.
6. **Report** (see Output Format), then stop and wait for the user.

## Roles

Pick 2–4 debaters. Two committed opponents are mandatory; add a specialist only when the artifact's risk profile demands it (security-sensitive surface, money paths, data migration).

- **PR review:** *Prosecution* — this change must not merge; find the defects, regressions, contract breaks, and untested paths that prove it. *Defense* — this change is correct and should merge as-is; justify every decision and show the tests/behavior that prove it. Optional specialist prosecutor for security, data integrity, or performance.
- **Architectural review:** *Champion* — the proposed architecture is right; argue it from constraints, scale, cost, and failure modes. *Challenger* — propose the strongest concrete alternative (including "do nothing" or "simpler variant") and prove it dominates. Vague "it depends" challenges are forfeits; the Challenger must commit to an alternative.
- **Feature review:** *Advocate* — build/ship it; argue from user evidence, product analytics, and revenue mechanics. *Executioner* — kill it; argue opportunity cost, existing-feature overlap, and weak demand evidence. Optional *Simplifier* — same outcome at a fraction of the scope; must specify the exact smaller cut.

## Evidence Hierarchy

Strongest to weakest. When claims collide, higher evidence wins; equal tiers go to the more specific claim.

1. Reproduced behavior: a failing test, executed repro, or measured number from real data.
2. Direct reading of the actual code/diff/config with file:line citations.
3. Authoritative current documentation, release notes, advisories, or platform data — searched fresh when recency matters, not recalled.
4. Established best practice with a citable source and an argument for why it applies *here*.
5. Unsupported opinion, analogy, or "common knowledge" — loses to everything, including silence.

## Nitpick Ban

A finding qualifies only if it materially affects: correctness, security, data loss/integrity, money paths, API/contract compatibility, performance at realistic scale, operational cost or failure recovery, or complexity that will demonstrably cause defects. Banned regardless of merit: style, formatting, naming taste, "I would have structured this differently", speculative flexibility, error handling for impossible states, and hypothetical inputs the system cannot receive. A debater who pads its case with nitpicks has those points struck and the rest of its case discounted. Severity floor for the report: every finding states the concrete failure it causes; no concrete failure, no finding.

## Output Format

1. **Verdict line first**: `MERGE` / `BLOCK (n findings)` for PRs; `ADOPT` / `REJECT — adopt <alternative>` for architecture; `BUILD` / `KILL` / `CUT TO <scope>` for features. One verdict, with the single strongest reason.
2. **Surviving findings** table: severity (P0/P1/P2), finding, evidence (file:line / test / source), impact, required action.
3. **Killed in debate**: arguments that died, each with the evidence that killed it — this is how the user spot-checks the debate was real.
4. **Unresolved** (only if truly undecidable from available evidence): what to measure or test to settle it.

## Fallbacks

- No subagent support: run the same roles as strictly separated sequential passes — write Round 1 cases fully before starting cross-examination, and never let one role's conclusions leak into another's case-building.
- Artifact too small to debate (one-line diff, trivial rename): say so and do a direct evidence-cited review instead of theater.
- If the debate deadlocks on missing ground truth (no tests, no data, no doc), the verdict is the conservative one (`BLOCK`/`REJECT`/`KILL`) and the report says exactly what evidence would flip it.
