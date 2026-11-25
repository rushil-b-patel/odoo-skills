# Commit message checklist

Read this during step 5. Apply every rule to each commit on the branch, judged
against the actual diff — the PR title and the author's summary can both be
wrong about what the commit does.

Report message findings with the same severity + why + suggestion shape as code
findings, naming the rule. If anything is wrong, end the review with the full
rewritten message so the author can `git commit --amend -F -` it rather than
reassemble your notes.

## 1. Line length

Title and every body line within 72 characters, hard. Wrap manually; don't rely
on the Git client.

## 2. Title completes "This commit will ..."

The title after any `[TAG] module:` prefix must finish that sentence
grammatically, which forces imperative mood: add, fix, remove, link, prevent,
allow — never adds, added, should, is, was.

- GOOD: `[FIX] website: link menus with spaceful URLs to new pages`
  → "This commit will link menus with spaceful URLs to new pages"
- BAD: `[IMP] website: menus should be linked with or without space`
  → "This commit will menus should be linked..." (broken)

## 3. Plain language over jargon

Write for the next engineer reading `git log`, not for the compiler. Technical
terms are fine when the change genuinely needs them (race condition, N+1 query,
MTU mismatch); flag jargon that doesn't earn its place.

## 4. Cover the key changes

The body surfaces every meaningful change in the diff — three things touched,
three things acknowledged, even briefly. Flag scope creep buried in the diff (a
refactor smuggled into a bug fix).

## 5. Justify why, not what

The diff already shows what changed; the message explains why it had to. Lead
with the problem, the user impact, or the constraint that forced the change.
Describe the what only where the code can't (a subtle ordering fix, a
workaround for a third-party bug).

## 6. Capture design decisions

If a non-obvious choice was made ("X over Y because Z"), it belongs here.
Future readers shouldn't need Jira, Slack, or PR comments to reconstruct the
reasoning. Flag missing rationale for any decision that isn't self-evident from
the code.

## 7. Sell the change

The reader should finish feeling the change was worth making: what's faster,
safer, clearer, fixed, or unlocked. No filler like "minor improvements" or
"cleanup" without specifics.

## 8. Structure

```
[TAG] module: <imperative summary, 72 chars max>

<Body wrapped at 72. Prose paragraphs preferred over bullet soup; bullets
are fine when the change is genuinely a list of independent items.>

Fixes: #1234
Task: TASK-1234
```

Trailers (`Fixes:`, `Closes:`, `Co-authored-by:`, task IDs) go at the bottom
after a blank line.
