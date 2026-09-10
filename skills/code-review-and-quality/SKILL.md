---
name: code-review-and-quality
description: Conducts multi-axis code review. Use before merging any change. Use when reviewing code written by yourself, another agent, or a human. Use when you need to assess code quality across multiple dimensions before it enters the main branch.
---

# Code Review and Quality

## Overview

Multi-dimensional review with quality gates. Every change gets reviewed before merge — no exceptions. Review covers five axes: correctness, readability, architecture, security, performance.

**The approval standard:** approve when a change definitely improves overall code health, even if imperfect. Don't block because it isn't how you would have written it.

## When to Use

Before merging any PR; after a feature, refactor, or bug fix (review the fix *and* its regression test); whenever another agent or model produced code you must evaluate.

## The Five-Axis Review

### 1. Correctness

- Does it match the spec, including edge cases (null, empty, boundaries) and error paths, not just the happy path?
- Do tests pass, and are they testing the right things?
- Any off-by-one errors, race conditions, or state inconsistencies?

### 2. Readability & Simplicity

- Are names descriptive and consistent with project conventions? (No bare `temp`, `data`, `result`.)
- Is control flow straightforward — no nested ternaries, deep callbacks, or "clever" tricks? **Could this be done in fewer lines?** 1000 lines where 100 suffice is a failure.
- **Are abstractions earning their complexity?** Don't generalize until the third use case.
- Any dead code artifacts: no-op variables, backwards-compat shims, `// removed` comments?
- **Is a new conditional bolted onto an unrelated flow?** That's a design smell, not a nit — push the logic into its own helper, state, or policy.
- **Do repeated conditionals on the same shape appear?** They signal a missing model or dispatcher. A "temporary" branch is usually permanent debt.

### 3. Architecture

- Does it follow existing patterns? If it introduces a new one, is that justified? Are module boundaries clean, dependencies acyclic, duplication shared?
- **Does this refactor reduce complexity or just relocate it?** Count the concepts a reader must hold. If a "cleaner" version leaves that count unchanged, it isn't cleaner — prefer restructurings that make whole branches, modes, or layers disappear. Prefer deleting an abstraction to polishing it.
- **Is feature-specific logic leaking into a shared module?** Keep logic in its owning layer, reuse the canonical helper instead of a near-duplicate, and don't normalize architectural drift.
- **Are type boundaries explicit?** Question gratuitous `any`/`unknown`/optionals/casts and silent fallbacks that paper over an unclear invariant.

### 4. Security

See `security-and-hardening` for depth, and `../../references/security-checklist.md`.

- Is user input validated and sanitized at system boundaries, and is external data (APIs, logs, user content, config) treated as untrusted?
- Are secrets kept out of code, logs, and version control? Are auth/authz checks present where needed?
- Are SQL queries parameterized and outputs encoded (no injection, no XSS)?

### 5. Performance

See `performance-optimization` for depth, and `../../references/performance-checklist.md`.

- Any N+1 queries, unbounded loops, or unconstrained fetching?
- Any sync operations that should be async, missing pagination on list endpoints, unnecessary re-renders, or large objects created in hot paths?

## Structural Remedies

When you flag a structural problem, propose the move — "this is complex" leaves the author guessing:

- Replace a chain of conditionals with a typed model or explicit dispatcher; collapse duplicate branches into one flow; separate orchestration from business logic.
- Move feature-specific logic into the package that owns the concept; reuse the canonical helper instead of a bespoke near-duplicate.
- Make a type boundary explicit so downstream branching disappears; delete a pass-through wrapper; extract a helper; split a large file.

Prefer the remedy that removes moving pieces over one that spreads the same complexity around.

## Change Sizing

- `~100 lines changed` is good, `~300` is acceptable for a single logical change, `~1000` is too large — split it.
- **Watch file size, not just diff size.** Around 1000 *total* lines in one file is an inspection signal, not a hard cap. When a change materially grows an already-large file, decompose first, then add.
- **One change** = one self-contained modification, with its tests, leaving the system functional. Large changes are fine for full-file deletions and automated refactors, where the reviewer verifies intent, not every line.
- **To split:** stack dependent changes; group by reviewer for cross-cutting work; go horizontal (shared code and stubs first, then consumers) in layered architectures; go vertical (small full-stack slices) for features.
- **Separate refactoring from feature work** — that's two changes. Small cleanups (renames) can ride along at reviewer discretion.

## Change Descriptions

- **First line:** short, imperative, standalone — "Delete the FizzBuzz RPC," not "Deleting the FizzBuzz RPC." Informative enough to understand without the diff.
- **Body:** what is changing and why — context, decisions, reasoning not visible in code. Link bugs, benchmarks, design docs. Acknowledge shortcomings.
- **Anti-patterns:** "Fix bug," "Fix build," "Moving code from A to B," "Phase 1."

## Review Process

1. **Understand the context** — what is this trying to accomplish, which spec does it implement, what behavior changes?
2. **Review the tests first** — do they exist, test behavior rather than implementation, cover edge cases, and would they catch a regression?
3. **Review the implementation** — walk each changed file through the five axes.
4. **Categorize findings** by severity (below).
5. **Verify the verification** — what tests were run, did the build pass, was it checked manually, are there screenshots or before/after numbers for UI and perf claims?

Severity prefixes, so authors don't treat every comment as mandatory: **Critical:** blocks merge (security, data loss, broken functionality); *no prefix* is a required change; **Optional:** / **Consider:** is a suggestion; **Nit:** is style the author may ignore; **FYI** needs no action.

**Lead with what matters.** Order by leverage: correctness and security, then structural regressions and missed simplifications, then everything else. A few high-conviction comments beat a long list — if you have one structural problem and ten nits, the structural problem *is* the review.

**Use a second model.** Have a different model review code an agent wrote; models have different blind spots. Ask it to check correctness, security, and conventions against the spec, labeled by the severities above. The human makes the final call.

## Dead Code Hygiene

After any refactor, identify code that is now unreachable, list it explicitly, and **ask before deleting**: "Should I remove `formatLegacyDate()` in `src/utils/date.ts` (replaced by `formatDate()`) and `LEGACY_API_URL` in `src/config.ts` (no references)?" Don't leave dead code around; don't silently delete what you're unsure of.

## Review Speed and Disagreements

Slow reviews block teams — the cost of context-switching to review is less than the waiting cost you impose. Respond within one business day (the maximum, not the target); a typical change should complete several rounds in a day. Prioritize fast individual responses over quick final approval, and ask for a split rather than reviewing a massive changeset.

Resolve disputes by hierarchy: technical facts and data over opinion; the style guide is the authority on style; design is judged on engineering principles, not preference; codebase consistency wins when it doesn't degrade health. **Don't accept "I'll clean it up later"** — deferred cleanup rarely happens. Require it before submission, or require a filed, self-assigned bug.

## Honesty in Review

- **Don't rubber-stamp.** "LGTM" without evidence of review helps no one. And don't soften real issues — calling a production bug "a minor concern" is dishonest.
- **Quantify.** "This N+1 adds ~50ms per list item" beats "this could be slow."
- **Push back on flawed approaches** and propose alternatives — sycophancy is a review failure mode. **Accept override gracefully.** Defer to an author with full context. Comment on code, not people.

## Dependency Discipline

**Before adding one:** can the existing stack do this (it often can)? How large is it? Is it maintained? Known vulnerabilities (`npm audit`)? Compatible license? Every dependency is a liability — prefer stdlib and existing utilities.

**Upgrading** is a code change like any other, and bulk "bump deps" PRs are the riskiest:

1. **Read the changelog, not the version number.** Semver is a promise the maintainer may not have kept.
2. **One dependency per change** — a bulk bump hides which package broke the build.
3. **Let the tests decide.** Green before *and* after; thin coverage around the dependency is itself the finding.
4. **Mind the transitive graph.** Review the lockfile diff, not just `package.json` — and never hand-edit it. For triaging audit findings and supply-chain risk, use `security-and-hardening`; this covers the upgrade *workflow*.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "It works, that's good enough" | Working code that's unreadable, insecure, or architecturally wrong compounds into debt. |
| "I wrote it, so I know it's correct" | Authors are blind to their own assumptions. |
| "We'll clean it up later" | Later never comes. The review is the gate — require cleanup before merge. |
| "AI-generated code is probably fine" | It needs more scrutiny, not less: confident and plausible even when wrong. |
| "The tests pass, so it's good" | Tests don't catch architecture, security, or readability problems. |
| "The refactor makes it cleaner" | Relocating complexity isn't reducing it. Look for the version where branches disappear. |
| "It's only a small addition" | Small diffs still push files past a healthy size and bolt branches onto unrelated flows. |

## Red Flags

- PRs merged without review, or "LGTM" with no evidence of one
- Security-sensitive changes with no security-focused review; bug-fix PRs with no regression test
- PRs "too big to review properly," or comments with no severity labels
- A refactor that moves code without reducing the concepts a reader must hold, or a change that grows an already-large file instead of decomposing it
- New conditionals scattered into unrelated code paths (a missing abstraction)
- A bespoke duplicate of a canonical helper, or feature logic in a shared module
- A bulk dependency bump with no changelog review, or a hand-edited lockfile

## Verification

- [ ] All Critical issues resolved; all Required changes resolved or deferred with justification
- [ ] Tests pass, the build succeeds, and the verification story is documented (what changed, how it was verified)
- [ ] Dependency upgrades: changelog read, isolated per package, green suite, lockfile diff reviewed

**Presumptive blockers** — surface these and propose the simpler design; escalate to Required only when the change actively makes structure worse: a refactor that relocates complexity; a file pushed past the size boundary with no decomposition; feature logic in a shared module; a near-duplicate of a canonical helper; a silent fallback hiding an unclear invariant.
