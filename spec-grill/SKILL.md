---
name: "spec-grill"
description: "Grill an unclear or low-quality feature spec before development: five isolated lenses (user, developer, tester, PM, product engineer) report anonymously to a blind judge for a functional verdict."
---

# Spec Grill

A pre-development review for feature specs, tasks and tickets. It is based on the LLM Council pattern (karpathy/llm-council): independent opinions, anonymized findings, and a final judge.

Use it when the user says "spec-grill", "grill this spec", "review this spec" or "is this task worth building", or hands over a vague or low-quality spec before starting work.

The question is **"should we build this, and in what shape?"**, not "how do we build it?". All output stays **functional**: users, flows, behavior and value. Members may read the codebase or the product to find evidence (is this already there, does it clash with something), but they report what they find in product terms. No implementation details.

## Why it's built this way
- **Different lenses, isolated.** Members running the same model make correlated mistakes. Distinct roles, mixed models (opus and sonnet), and no shared context in Step 1 give the council real diversity.
- **A blind judge.** The main session assigns the lenses, so it knows who said what. The judge must be a fresh subagent that only sees anonymous findings.
- **Evidence beats votes.** One well-evidenced "this already exists" outweighs four members who liked the idea.

## Step 0: Spec Brief (you)
1. Collect the spec: pasted text, a file, or a task link. Collect the codebase path or product area if there is one.
2. Restate the spec as a **Spec Brief** covering: the feature in one line, the intended user, the stated problem, the stated behavior, and anything unclear or missing.
3. Ask the user at most 1 to 3 questions, and only if members would otherwise review different features. Gaps in the spec are not a reason to ask. Members report those as findings.

## Step 1: Five lenses (parallel, isolated)
Launch all five in a **single message** as `general-purpose` subagents. They are read-only.

| Lens | Model | Mandate |
|---|---|---|
| End User | sonnet | Walk the flow as a real, non-expert user. Would I find it, understand it, want it? Where would I get stuck? |
| Developer | opus | Does this already exist, fully or partly, in the product? Does it conflict with existing behavior? Which parts are disproportionately complex for their value? Explain in functional terms. |
| Tester | sonnet | Unclear or contradictory behavior. Missing states: empty, error, permission, mobile, undo. Edge cases the spec doesn't define. |
| Product Manager | opus | Is it worth doing? Who benefits and how much? Does it fit the product? What is the smallest version that delivers the value? |
| Product Engineer | opus | Bridges user and system. Is there a simpler flow to the same outcome? Are there related small additions with a big gain in ease or accessibility? |

Use this prompt for each member:
```
You are on a pre-development spec review council. Your lens: <LENS + mandate>.
Work independently; others review in parallel and you cannot see them.
This is a FUNCTIONAL review: talk about users, flows, behavior and value, not code or implementation.
You MAY read the codebase/product (<path>) to find evidence; cite it briefly as a reference, then describe it in product terms. Do NOT edit files.

<SPEC BRIEF>

Return exactly (max ~500 words):
## Stance
Build as-is | Build with changes | Rework spec first | Don't build, plus a one-line reason
## 1. Already exists / conflicts
- <existing feature or behavior it duplicates or clashes with, with evidence>, or "none found (what you checked)"
## 2. User value & ease of use
- Who benefits, the job they're doing, before -> after
- Friction points in the proposed flow
## 3. Worth adding
- <addition>: <user benefit>, rated Must | Should | Nice
## 4. Cut, simplify, or shift
- <part>: <why it's low value / redundant / too complex> -> <better alternative>
## 5. Open questions for the spec owner
- <question>, and why the answer changes what gets built
## Confidence
low | medium | high, plus a reason
```

## Step 2: Anonymous peer review (optional, only when the user passes `review`)
1. Remove lens names, model names, and self-references. Label the findings **A to E**. Shuffle the order for each reviewer, and keep the mapping private.
2. Send each member all of the findings, including its own, without saying which one is its own. Launch them all in one message. Ask each member to give, for every finding, its strongest point and its biggest flaw, then end with the line `FINAL RANKING: X > Y > Z ...`.
3. Work out the average rank of each finding (1 = best).

## Step 3: Blind judge
The judge is a **fresh subagent** with `model: "opus"`. It is not you.
1. Remove lens names, model names, and self-references from every finding. Label them **Finding A to E** in shuffled order, and keep the mapping private.
2. Send the judge the Spec Brief, the anonymous findings, and the Step 2 rankings if Step 2 ran:
```
You are the judge of a spec review council. You receive anonymous findings; weigh them on evidence and user impact, not on how many findings repeat a point. A single well-evidenced objection (e.g., "this already exists") can outweigh a majority.
Stay functional: no implementation details.

<SPEC BRIEF>
<Finding A>...</Finding A> ... <Finding E>...</Finding E>

Return the Spec Verdict in exactly this format:

# Spec Verdict: <feature>
**Verdict:** Build as-is | Build with changes | Rework spec first | Don't build
<2-3 sentence reason>

## Already exists / conflicts
- ...
## User value
Who, the job, before -> after. Ease-of-use verdict.
## Refined spec (functional)
- **Must-have scope:** ...
- **User flow:** 1. ... 2. ...
- **Acceptance criteria (user terms):** When <user does X>, they see/can <Y> ...
## Recommended additions (prioritized)
1. <addition>: <benefit>, rated Must | Should | Nice
## Cut, defer, or replace
- <part> -> <alternative>, because ...
## Open questions for the spec owner (blocking first)
1. ...
## Strongest dissent
- <the best argument against the verdict>
```
3. Pass the judge's verdict on to the user unchanged. Then add a short **Lens map** of your own: which lens each finding came from, and where the lenses converged or split. For example: "Tester and End User independently flagged the missing empty state." Reveal this only after the verdict.

## Options
- `members=3`: lite mode, with End User, Developer and PM only. Use it for small specs.
- `review`: add Step 2. This costs more and suits big or contested specs.
- `path=<dir>`: the codebase or product area that members should check for existing or conflicting behavior.

## Rules
- Members and the judge are read-only. Nobody edits files or starts implementing.
- Never reveal identities to reviewers or to the judge.
- If findings come back nearly identical, say so: the council added little diversity on this spec.
- Be honest about cost. The default run is 6 calls (5 lenses + judge), and 11 calls with `review`. If the spec is plainly fine, say so instead of grilling it.
- Without a codebase path, the "already exists / conflicts" check is weak. Say so in the verdict.