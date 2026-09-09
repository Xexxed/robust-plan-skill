---
name: robust-plan
description: Design deep, evidence-based implementation and testing plans for complex features and fixes in the "robust plan" style — invariants, route selection, staged rollout, acceptance criteria, fallback tables, isolated testing. Use when the user asks for an implementation plan, a testing plan, "plan the implementation", "how should we implement X", "design a plan for X", "make a plan for this feature", "write a plan", is planning a complex/robust feature, or wants a detailed plan before coding.
---

# Robust Implementation Planning

Produce implementation plans in the style proven by two exemplars: a pipeline reliability fix plan (`FIX_IMPLEMENTATION_PLAN.md`) and a swarm-download architecture plan (`instant-swarm-download-plan.md`). Both share one method. Use this skill when the user wants a plan for a complex feature or fix — as a document alone, or as a document followed by implementation.

The style's non-negotiable commitments:

- **Evidence before design.** Every claim about current code cites `file:line`. No planning from memory.
- **Invariants before implementation.** Numbered absolute rules the design must never violate.
- **Honesty throughout.** The plan states what is NOT known, and never claims work is done or tested when it is not. Verification lists read "the following are required tests, not tests already executed."
- **Staged delivery.** Risk-ordered stages: protect data first, concurrency/scaling last, everything risky behind default-off toggles.
- **Nothing destructive.** Fallbacks escalate from graceful to bounded retry to quarantine-and-report — never broad destructive action.

## Method

Follow the phases in order. Do not write the plan document before Phase 2 and Phase 3 are resolved.

### Phase 1 — Gather evidence

- Read every file the feature touches, plus its callers and configuration. Note exact line numbers for every relevant behavior.
- If the repo has `AGENTS.md` or `CONTEXT.md`, read them first.
- Record what the review did NOT establish: unreproduced environments, unverified assumptions, runtime behavior not observed. This becomes the honesty backbone of the plan.

### Phase 2 — Resolve ambiguity with the user

Ask the user (via the question tool, options with a recommended default first) when:

- Scope is ambiguous (which surfaces, hosts, platforms, entry points are covered?).
- Two routes materially differ in outcome or reversibility.
- A decision is destructive or hard to reverse.
- Defaults must be chosen (sizes, limits, on/off, precedence between modes).
- Something in the codebase or the request is genuinely confusing.

Do NOT ask when the answer is discoverable from the code, or the choice is trivially reversible. Never silently invent an answer to a question you should have asked.

Record every resolved answer in the plan under **Decisions (resolved with the user)**. Anything left unresolved goes under **Non-goals and shortcomings** or as an explicit open question — visibly.

### Phase 3 — Choose the route

- Enumerate 2–3 candidate implementation routes (not micro-variants — genuinely different architectures or strategies).
- Compare them in a small trade-off table keyed to the goals and invariants.
- Pick the best route and record: why it wins, and why each rejected route loses. If the choice is close or materially affects the user, make it a Phase 2 question.

### Phase 4 — Write the plan

Write the plan to a file: `<FEATURE>_IMPLEMENTATION_PLAN.md` at the repo root (or `.kilo/plans/` if the project keeps plans there). Summarize briefly in chat; the file is the deliverable. Use this exact section order:

```
# <Feature Name>: <one-line intended end state>

## Scope and evidence
## Goals
## Decisions (resolved with the user)        <- omit only if there were none
## Route selection
## Target architecture (or Target design)
### Invariants
## 1..N  <one section per concern area>
## Failure modes and fallbacks
## Smaller but definite fixes / considerations
## Non-goals and shortcomings
## Implementation sequence
## Required verification
## Post-implementation additions
## Rollout
## Final acceptance criterion
```

Section-by-section rules:

- **Scope and evidence.** Files reviewed (with branch), what the review established and explicitly did not. End with one honesty sentence: "This document is an implementation plan, not a claim that it is implemented or tested." (Adjust the wording for the actual situation — but never omit the disclaimer.)
- **Goals.** Concrete, verifiable bullets. Include preservation goals ("with the toggle off, behavior is byte-for-byte identical to today") alongside new capabilities.
- **Decisions.** Numbered. Each states the decision and a one-line rationale. Mark which were user-confirmed defaults.
- **Route selection.** Trade-off table, chosen route, why, why the others were rejected. Short.
- **Target architecture.** A fenced text diagram of the intended lifecycle/flow, then numbered **Invariants** — absolute rules using "Never", "at most one", "Every X must Y". Invariants are where the plan earns its robustness: cover ownership, cleanup, state consistency, bounded terminal states, and untouched-by-default behavior.
- **Per-concern sections** (numbered, grouped by subsystem or concern — not by file):
  - **Files.** Exact paths this section touches.
  - **Findings** / **Current constraints.** What the code does today, with `file:line` citations. Include risky coupling, not just bugs.
  - **Changes.** Numbered imperative directives ("Use X", "Do not Y", "Never Z"). Specific enough to implement without re-deciding: exact function/state/file names, exact precedence rules, exact validation steps. Reuse existing helpers explicitly ("resolve via `X.resolve_filename` — reuse, do not duplicate").
  - **Why.** 1–3 sentences tying the changes back to the invariants.
  - **Acceptance criteria.** Checkbox list; each item independently verifiable.
- **Failure modes and fallbacks.** Table: `Situation | Procedure`. Must cover: crash mid-operation, user cancellation, timeout, partial success, stale/invalid state, and ownership-uncertain cases. Escalation order: graceful path → bounded retry → quarantine/report → never broad destructive action.
- **Smaller but definite fixes / considerations.** Table: `Location | Problem | Required change` (or `Consideration | Required handling`). Small items that would otherwise get lost.
- **Non-goals and shortcomings.** What the plan deliberately does not do; known limitations; open questions. Honest, visible.
- **Implementation sequence.** Staged checkbox lists, risk-ordered: (1) protect data / fix immediate crashes, (2) core architecture behind default-off toggles, (3) integrations, (4) UIs, (5) concurrency and scaling only after the serialized/single-path version is verified. Mark stages that must deploy together. Give each stage a rollback criterion ("roll back to N if X regresses").
- **Required verification.** Open with: "The following are required tests, not tests already executed." Mocked/fake-server tests first, real integration second. Grouped checkbox lists per area. Include regression tests proving untouched paths stay untouched. Apply the testing isolation rules below, and state them in the plan.
- **Post-implementation additions.** Checklist of everything required after code lands, beyond the code itself: version bumps, rebuild/reinstall steps, verifying the running version, docs/README updates, `.env.example` entries, config persistence keys, data migrations, log lines, rollback instructions.
- **Rollout.** Numbered operational steps: back up state, ship behind default-off flags, single smoke test on real inputs, compare against baseline, then widen.
- **Final acceptance criterion.** One paragraph the whole plan converges on — the ultimate invariant stated as observable behavior.

### Phase 5 — Quality checklist

Before presenting the plan, verify:

- [ ] Every code claim has a `file:line` citation.
- [ ] Every numbered section ends with acceptance criteria.
- [ ] Invariants cover ownership, cleanup, state consistency, and never-destructive behavior.
- [ ] The failure table covers crash, cancel, timeout, partial success, and stale state.
- [ ] Verification says "required tests, not tests already executed", mocked before real.
- [ ] Testing isolation and env-restore rules appear in the verification section.
- [ ] Stages are risk-ordered, default-off toggles present, rollback criteria present.
- [ ] Post-implementation additions include version/packaging/docs/config items.
- [ ] Non-goals and shortcomings are listed honestly.
- [ ] The final acceptance criterion is one paragraph.

## Testing isolation rules

Apply to every plan's verification section and to every test actually run:

- Tests run in isolation: temp directories, mock servers, in-memory fakes, injected config. Never mutate real env values, real profiles, real queues, or user data.
- If a test MUST change an env var or config value: snapshot the original value first, restore it in `finally`/teardown, and verify restoration — including on failure paths. Never leave the environment modified.
- Prefer reading configuration through injectable objects over mutating `os.environ` or rewriting config files.
- Never point tests at production URLs, shared state, or anything a crash could corrupt.
- A test that cannot clean up after itself must be redesigned, not excused.

## From plan to implementation

When the user asks to implement the plan:

- Follow the stages in order. Tick a checkbox only when its acceptance criterion is actually verified.
- If reality diverges from the plan (a finding was wrong, an interface differs, a route no longer fits), stop, note the divergence as a shortcoming, update the plan file, and tell the user before continuing.
- The plan file is the living source of truth for what was decided and why — keep it current as work progresses.
- After implementation, execute the **Post-implementation additions** checklist.

## When not to use

Single-file tweaks, typo fixes, small bug fixes, and exploratory one-liners do not need this skill — just do the work.
