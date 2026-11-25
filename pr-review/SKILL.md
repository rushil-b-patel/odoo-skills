---
name: pr-review
description: Review a commit, branch, or PR diff (code plus commit message) deeply enough to merge in one pass, reporting findings as severity + why + suggestion. Use this whenever the user asks to review a PR, MR, commit, or diff, asks whether a change is ready to merge or ship, asks for feedback on code they just wrote or are about to push, asks to check a commit message, or mentions code review, self-review, or pre-push check — even if they don't say "PR review".
---

# PR review

Aim for a single-pass merge: one review complete enough that after the author
applies it, a second round surfaces nothing new. Two failures break that goal
equally — missing a real defect, and reporting a fake one. So verify every
finding against code you actually opened, and delete every "might" before you
write the report.

Report findings as **severity + why + suggestion**, in plain words the author
reads once and acts on. Nothing else earns space: they already know what their
diff says, they need to know what's wrong, why it matters, and what to type
instead. Step 8 is how to write that; it is not optional polish.

## Target

Review `$ARGUMENTS`. If empty, review the last commit on the current branch
(`git show HEAD`). For a branch, diff against the merge base with the main
branch (`git merge-base`). For a PR ref, fetch the diff plus every commit
message on the branch.

## 0. Team checklist first — always

Before reading any code, pull the team's codified review rules from the
`reviewdoo` MCP. These are distilled from past PR feedback, so a violation is a
finding you cite instead of argue.

- Get the changed file list (`git diff --name-only <base>..<head>`, or
  `git show --name-only --format= HEAD` for a single commit).
- Call `mcp__reviewdoo__get_review_checklists` with `files` set to that list
  and `version` set to the target Odoo branch. The branch name usually carries
  it (`18.0-fix-...`, `saas-19.3-...`, `master-imp-...`) and the tool accepts
  the feature branch name as-is. Omit `version` only when nothing tells you the
  target; then check each item's version range yourself before reporting it.
- If the tool is deferred, load it with ToolSearch first. If it is missing or
  errors, say so on the `Tests:` line of the report — never skip it silently.
- Keep the returned list beside you for the rest of the review: code/comment
  items feed step 3, `[commit-message]` items feed step 5, general items apply
  to both. Every item is either cleared or flagged by the step 7 sweep.

## 1. Frame it — context before judgment

Forming opinions before this step is done is how reviews invent problems that
the surrounding code already solves.

- **Intent, one sentence**: "This change exists because X." Read the PR/task
  description and commit messages. Judge against that intent, not your guess
  of it.
- **Pick the lane** from the commit tag, corrected by the diff when they
  disagree (the mismatch is itself a finding):
  - `[FIX]` → **bug-fix lane**. Establish the failure first: wrong behavior
    under what input? Then `git blame` / `git log -S` the touched lines for the
    root cause. A fix that leaves the root cause alive, or masks the symptom,
    is a blocker. Missing regression test is a major.
  - `[IMP]` `[ADD]` `[REF]` `[REM]` → **improvement lane**. Design and
    backward compatibility carry the weight here; refactors and removals break
    callers.
- **Read whole files**, not hunks. What breaks is usually just outside the
  diff context.
- **Read 2-3 siblings** for local convention (naming, error handling, logging,
  test layout). Deviating from a convention this codebase applies consistently
  is a finding; deviating from your personal taste is not.
- **Map the blast radius**: grep every caller of changed functions and every
  consumer of changed exports, configs, and schemas. Keep the list — step 3
  checks each one.

## 2. Tests: read them, don't run them

Runbot runs the full suite on push, so running it locally buys nothing. Spend
the effort where runbot is blind:

- `git diff --check` for whitespace damage and conflict markers.
- Do the tests exercise the changed behavior, or merely call the function? For
  a bug fix, would a test fail without the fix? Runbot green proves the old
  tests still pass, not that the new behavior is covered.
- Trace the changed path by hand with real data rather than assuming coverage.
- Existing tests asserting the OLD behavior now mislead — flag them.

## 3. Review the diff

Read `references/code-checklist.md` and work its three passes: **A** does it
work (correctness, performance, security), **B** should it work this way
(convention, simplicity, reuse, compatibility), **C** is it clean (dead code,
defensive bloat, tests, description match). Check every reviewdoo code item
from step 0 against the hunks it applies to, alongside the passes.

Scale the effort to the diff. When it spans many files or several independent
modules, dispatch one subagent per pass — or per module — each with the file
list and that pass's section, then merge and dedupe their findings here.
Subagents return findings; the verdict is always assembled by you. For a small
diff, run the passes inline; agent overhead isn't worth it.

## 4. The invisible diff

Most second review rounds are caused by what the change forgot, so hunt for it
explicitly:

- Callers, docs, comments, README, config samples, and env examples the change
  just made stale.
- The same pattern elsewhere: grep the codebase for the bug that was just
  fixed — it rarely lives in one place. Copies outside the diff are out of
  scope per step 6; mention them only in the one `Pre-existing:` line.
- Symmetry: if `open` / `create` / `subscribe` / `serialize` changed, check
  `close` / `unlink` / `unsubscribe` / `deserialize`.
- Migrations, seed data, fixtures, i18n strings, feature flags, and changelog
  entries this repo conventionally requires.

## 5. Commit message

Read `references/commit-message.md` and apply it to every commit on the branch,
checking against the actual diff rather than the PR title or the author's
summary. Apply the reviewdoo `[commit-message]` items from step 0 too.

## 6. Filter false positives

Walk the candidate list and keep a finding only if every answer is yes:

1. In scope — caused by this diff, or made necessary by it? In scope is: a line
   the change added or modified, a caller / test / doc / translation the change
   broke or left stale, and code the change now newly depends on being correct.
   Out of scope is everything else the surrounding files happen to get wrong,
   however real: a diff is not a licence to review its neighbourhood, and the
   author cannot act on those without growing the change they came to merge.
2. Verified against the actual code, with a concrete failing scenario or cost
   you can state? (Not "might", "could potentially", "consider".)
3. Not already handled by the type system, an upstream check, a framework
   guarantee, or a pattern you saw in step 1?
4. Not simply this codebase's established convention?
5. Would fixing it genuinely change behavior, risk, or readability?

Failing 1 means drop it. The one exception: a pre-existing defect that
threatens *this* change's correctness earns a single `Pre-existing:` line at
the end of the code section, with no severity and no bearing on the verdict —
never a `blocker` / `major` / `minor` / `nit`. Cap that at one line total, and
say nothing when there is nothing to say; the author asked what is wrong with
their diff, not for an audit of the file.

Failing 2-4 means drop it, or downgrade it to one `Question:` line for the
author — speculation forces a pointless second round exactly like a missed bug
does. Failing only 5 because the sole gain is readability is not a drop; report
it as a `nit`. The trap to avoid is silently clearing a line because it works:
correct isn't the bar, necessary is.

## 7. Sweep

Re-read the raw diff top to bottom with your finding list beside it. Every hunk
is either consciously cleared or flagged — no skimmed hunks. Same for the
reviewdoo checklist: every item is cleared or flagged. Then check whether
each fix you're proposing creates follow-on work (signature change, test
update, doc update) and fold that into the suggestion so the author isn't
surprised mid-fix.

## 8. Say it plainly

The review is only worth what the author understands on first read. Two rules
carry all of that.

**Explain from first principles, not from authority.** Never name a rule and
stop. Give the chain: what the code does → what goes wrong → who pays for it.
The author should be able to re-derive the finding on their own next time,
without having read the same books you did.

```
BAD   Violates single responsibility; consider extracting a service.
BAD   Potential N+1 query pattern.
GOOD  The loop reads product_id one line at a time, so a 500-line order
      fires 500 queries and the page times out.
```

Same rule for design and convention findings. "Doesn't follow the pattern" is
authority; "every other model here computes this in `_compute_*`, so yours
won't recompute when the field changes" is first principles.

**Use small words.** Write it the way you'd say it out loud, standing at the
author's desk. Keep technical terms only where the code genuinely needs them —
deadlock, race, index scan. Everything else gets the plain word:

| not this | this |
| --- | --- |
| instantiate, construct | create, make |
| leverage, utilize | use |
| prior to, subsequent to | before, after |
| in the event that | if |
| non-trivial, significant | say the real number |
| it is recommended that you | just write the fix |
| this may potentially cause | this causes / drop the finding |

Length is part of clarity. `Why:` is at most two sentences — if it needs three,
the finding is really two findings or you haven't understood it yet. No
finding runs past four lines including the fix.

Ban these words outright: consider, ensure, robust, leverage, arguably,
potentially, it appears, one might. Most of them are hedging, and step 6
already deleted everything you were hedging about.

## Output

Severity rubric:

| severity | meaning |
| --- | --- |
| `blocker` | breaks correctness, security, or compatibility; cannot merge |
| `major` | real defect, missing test for changed behavior, or compat risk |
| `minor` | should fix, won't block the merge |
| `nit` | style or preference; state once, never argue |

Verdict follows mechanically from the worst severity, so it never needs
debating: any `blocker` or `major` → **request changes**; only `minor`/`nit` →
**approve with nits**; nothing → **approve**. A missing regression test stays
`major` even on a two-line correct fix — the bug can silently return, and that
risk doesn't shrink with diff size.

Use this shape. Verdict first, worst finding first, code before commit
message, nits collapsed at the end:

~~~
**request changes** — <the single worst thing, in one clause>
<lane> · <N> files · <count by severity>
Tests: runbot on push. <coverage gap, or "changed path covered">. <commit message state>

### Code

**blocker** · `sale/models/sale_order.py:142`
Why: <what the code does, what goes wrong, who pays — two sentences max>
Fix:
```python
<exact replacement code>
```

**major** · `sale/models/sale_order.py:88`
Why: <...>
Fix: <exact code, or one sentence when the change is obvious>

### Nits
- `sale/models/sale_order.py:31` — `tmp2` holds the taxed total; name it that.
- `sale/views/order.xml:12` — trailing whitespace.

### Commit message
**major** · <rule name>
Why: <...>
Fix: rewritten message below.
~~~

If any commit message finding exists, end with the rewritten message, ready to
paste:

```
[TAG] module: <imperative title, under 72 chars>

<Why paragraph: problem, user impact, or constraint. Wrapped at 72.>

Task: TASK-1234
```

Keep it tight:

- No praise, no restating the diff, no "Issue:" line separate from `Why:` —
  the why states the problem.
- Nits are one line each under `### Nits`, location then the point. They never
  get a `Why:`/`Fix:` pair — a nit that needs a paragraph was never a nit.
- Drop a section with no findings rather than printing an empty heading. A
  clean commit message is worth one clause on the `Tests:` line so the author
  knows step 5 actually ran.
- One location per finding. If the same defect repeats, report it once and list
  the other lines in `Fix:`.
- No summary section. The verdict line at the top is the summary.
- Zero findings is a valid, short review: verdict line, `Tests:` line, done.
