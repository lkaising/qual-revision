# Review Council

A reusable, eight-seat review council for judging and reconvening on a selected
section of the system card (or any precise technical document). Convene it on one
`{SECTION}` at a time. The roster is fixed except **Seat 1, the Domain Expert,
which swaps to match the section** (robotics for Sections 5 to 7, machine learning
for Section 3, clinical or statistics for Section 8).

Each seat has one obsession, a defined scope, a checklist, and a paste-ready lens
line for the subagent prompt. The seats are built with deliberate edge-overlap, so
the most important defects get caught by more than one lens.

## The eight seats

### 1. The Domain Expert (swappable: roboticist / ML scientist / clinician)
*"Is this actually correct and buildable in the real world?"*

- Owns the section's technical substance.
- Checks: are equations and mechanisms physically right; is it feasible; does it
  match real constraints; does it answer the questions a practitioner in that
  field would ask.
- Lens line: "You are a [robotics control / ML / clinical] expert. Find every
  technical error, infeasibility, or under-specification in {SECTION}. Be
  rigorous; do not rubber-stamp."

### 2. The Newcomer (the "simple person")
*"I don't know this project. Can I follow it?"*

- Reads as an outside engineer with no prior context.
- Checks: undefined acronyms or terms used before definition; sentences too dense
  to parse; missing "what is this and why"; anything that needs context the reader
  does not have.
- Lens line: "You have never seen this project. Flag every term, symbol, or
  sentence you cannot understand on first read, and name what is missing."

### 3. The Formalist (the "math expressions lover")
*"Is every symbol defined, every equation right, every range stated?"*

- Owns notation and math.
- Checks: a symbol inventory (each defined at or before first use, with range and
  units); notation collisions; dimensional correctness; equations that are
  decorative versus load-bearing; one symbol per concept.
- Lens line: "Build a symbol inventory for {SECTION}. Flag every undefined symbol,
  collision, dimension error, missing range, or equation that only restates prose."

### 4. The Systems Architect (cohesion and integration)
*"How does this piece connect to the whole?"*

- Owns interconnection.
- Checks: does the section name its place in the overall structure; are
  cross-references correct; do shared objects and symbols thread consistently in
  and out; does the loop or flow close.
- Lens line: "Judge only how {SECTION} connects to the rest of the document. Flag
  broken cross-references, orphaned concepts, and places the throughline is unclear."

### 5. The Skeptic (adversarial examiner)
*"Where is this overclaiming, and how would I attack it?"*

- Owns the argument (distinct from Seat 8, which attacks behavior).
- Checks: claims stronger than the evidence supports; hidden assumptions; missing
  falsifiability; weasel words; the question an examiner would use to corner the
  author.
- Lens line: "Attack {SECTION} as a hostile examiner. Find every overclaim,
  unstated assumption, and place a reviewer would push back."

### 6. The Source Keeper (faithfulness and scope discipline)
*"Does this match the source, and respect what is locked or deferred?"*

- Owns fidelity.
- Checks: nothing load-bearing dropped from the source; nothing unsupported added;
  genuinely-open decisions flagged rather than silently invented; locked sections
  not contradicted; lock-the-interface-defer-the-mechanism honored.
- Lens line: "Compare {SECTION} against the source documents. Flag dropped
  substance, unsupported additions, invented 'settled' decisions, and
  contradictions with locked material."

### 7. The Editor (concision and style)
*"Can this be shorter and cleaner without losing meaning?"*

- Owns prose discipline.
- Checks: redundancy and restatement; run-on sentences; banned conventions (for
  example, em-dashes); voice consistency; opening with the answer; one question
  per section.
- Lens line: "Tighten {SECTION}. Flag redundancy, run-ons, convention violations,
  and any sentence that buries its point. Give the shorter rewrite."

### 8. The Safety / Failure-Mode Officer (robustness; generalizes to "what breaks?")
*"What is the worst case, and is it bounded?"*

- Owns failure behavior (distinct from Seat 5, which attacks claims; this attacks
  behavior).
- Checks: unhandled inputs or states; unbounded actions; missing fallbacks;
  graceful degradation; who has authority when things go wrong.
- Lens line: "Enumerate how the behavior in {SECTION} can fail or run away. Flag
  any unbounded action, unhandled state, or missing fallback."

## Quick reference

| # | Seat | Obsession |
| --- | --- | --- |
| 1 | Domain Expert (swap) | technically correct and buildable |
| 2 | Newcomer | understandable cold |
| 3 | Formalist | symbols, math, ranges |
| 4 | Systems Architect | connects to the whole |
| 5 | Skeptic | overclaims and attacks |
| 6 | Source Keeper | faithful and in scope |
| 7 | Editor | concise and clean |
| 8 | Safety Officer | bounded failure |

## How to run it

**Inputs every seat receives.** The target `{SECTION}`; the full document for
context; the source or authority documents; the style standard; and a note on what
is locked or out of scope.

**Output contract (identical for all seats).** A prioritized list, most important
first, in the form:

> [severity high/med/low] | location (quoted phrase) | the problem | a concrete fix

Each seat stays silent on what is already fine and ends with a one-line verdict.

**Chairing (the orchestrator).** Dispatch all eight in parallel, then:

1. Dedupe and cluster the findings.
2. Reconcile conflicts (for example, the Editor wanting to cut what the Architect
   wants kept for cohesion).
3. Apply the fixes.
4. **Reconvene** the relevant seats on the revised section to confirm resolution
   and catch regressions.

A section is done when the reconvened seats return go or no-go with only
low-severity polish remaining.

**Why eight with deliberate edge-overlap.** The most important defects are caught
by two lenses at once (the Formalist and the Domain Expert both hit a wrong
equation; the Newcomer and the Editor both hit dense jargon). That redundancy is
the point.
