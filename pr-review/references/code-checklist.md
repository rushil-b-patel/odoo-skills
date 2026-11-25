# Code checklist — three passes over the diff

Read this during step 3 of the review. Reference the codebase on every point;
hold the change to the bar "would a senior engineer on this team be proud to
ship this exact diff?"

Each pass answers a different question, which is why they're separate: a diff
can be correct and still be the wrong design, and a well-designed diff can
still carry dead code. Running them as one pass reliably loses B and C.

- [Pass A — does it work?](#pass-a--does-it-work-correctness-and-safety)
- [Pass B — should it work this way?](#pass-b--should-it-work-this-way-design)
- [Pass C — is it clean?](#pass-c--is-it-clean-hygiene)

## Pass A — does it work? (correctness and safety)

- **Data flow**: trace every changed function — where each input comes from
  (the call sites mapped in step 1) and where each output goes. Mismatches
  between those two ends are where real bugs live.
- **Edge cases**: empty input, boundary values, large input, concurrent calls,
  partial failure, retries.
- **Mechanics**: right data structure and algorithm for the problem; resource
  cleanup (cursors, connections, listeners, timers, file handles); idempotency
  where it matters; concurrency safety (races, shared mutable state); correct
  async/await; deterministic behavior; graceful failure modes.
- **Performance**: N+1 queries, loops that could be a single query, blocking
  I/O on hot paths, unbounded fetches, missing indexes, leaks.
- **Security**: input validation at trust boundaries, auth/authz checks,
  secrets in code, SQL injection, XSS, SSRF, unsafe deserialization, sensitive
  data in logs. Flag concrete issues with file and line — never a vague
  "consider security".

## Pass B — should it work this way? (design)

- **Codebase alignment**: naming, file structure, error handling, logging,
  imports follow established patterns. Names convey purpose without the reader
  inspecting the implementation. An unjustified deviation from a consistent
  local pattern is a finding.
- **Simplicity from first principles**: is this the simplest solution to the
  actual problem? Flag premature abstraction, unnecessary design patterns,
  speculative generality. Ask what problem each line solves; flag anything that
  doesn't answer cleanly.
- **Reuse sweep** — do this deliberately, it's recall-bound and correctness
  review never surfaces it. For every unit the diff adds, search for an
  existing base class or sibling that already does most of the job. Could it
  extend that base and override one hook instead? Do two units (new+new or
  new+existing) differ only in a value or message, so they collapse into one
  parametrized unit?
- **Backward compatibility**: check function signatures, return types, error
  shapes, public exports, config keys, and schemas against the caller list from
  step 1. If breakage is intentional, is it documented with a migration path?

## Pass C — is it clean? (hygiene)

- **No unnecessary code**: dead code, unused variables, redundant comments,
  leftover debug statements, commented-out blocks, verbose implementations
  where the codebase style is concise. This includes needless intermediate
  variables and single-use aliases — if inlining is equally clear and shorter,
  it's a nit.
- **No defensive bloat**: optional chaining, null checks, try/except, or guard
  clauses where the value is guaranteed by the type system or an upstream
  contract. Defensive code earns its place at trust boundaries (external input,
  network, user data); inside trusted internal flow it's noise.
- **Engineering practices**: single responsibility, appropriate abstraction
  level, error handling matching local conventions, no magic numbers or strings
  where the codebase uses named constants, logs and metrics consistent with the
  surrounding code.
- **Tests**: added for new functionality or the fixed bug, and actually
  exercising the changed behavior rather than just calling the function. Flag
  missing tests for non-trivial logic; if tests aren't feasible, the PR should
  say why.
- **Description match**: does the PR description describe this diff? Flag scope
  creep (does more than stated), undersell (does less or differently), and
  missing context reviewers need.
