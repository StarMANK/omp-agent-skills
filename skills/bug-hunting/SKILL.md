---
name: bug-hunting
description: Use when diagnosing a crash, exception, wrong output, flaky test, race, regression, or any gap between observed and expected behavior. Reproduce first, minimize the case, localize with LSP and bisect, test one hypothesis at a time, fix the violated invariant rather than the symptom, then guard it with a test that fails before the fix.
allowed-tools: read, bash, edit, write, grep, glob, find, eval, task, todo, lsp, debug, web_search
---

# Bug hunting

Diagnose by evidence, never by plausibility. A theory you have not falsified is a guess with better prose.

## 0. The two rules

1. No fix before a reproduction. Without a failing command you cannot prove the fix, so you will ship a guess.
2. No claim without output. Every "it works now" must be the same command that failed before, showing it no longer fails.

## 1. Capture the failure verbatim

Before anything else, record the raw evidence:

- The exact command with its flags, the working directory, the environment that matters (OS, runtime version, env vars, feature flags, database/fixtures).
- The full error text, stack trace, and surrounding log lines. Save it: `cmd 2>&1 | tee /tmp/repro.log`.
- The expected behavior as an executable assertion, not a sentence: the exit code, the output, the field value.

Then find the error's origin. Grep the literal message across the tree — `grep "invalid frame size"` — and follow it to the raise site. Most "mysterious" failures are emitted by the project's own code, and the raise site names the invariant that broke. Only if no literal match exists is the message foreign (dependency, libc, OS).

## 2. Reproduce deterministically

Run the failing case and note the rate: always / N times out of M / never again.

- If it reproduces, lock the repro command down as the gate for every later step. Re-run it before and after each change.
- If it does not, go to §8. Do not start editing on the strength of one screenshot.
- If the repro is a test, run it in isolation and in its normal context: `pytest path::test -x` and the full file. The gap between the two is itself a finding (shared state, ordering, fixtures).

## 3. Localize

Narrow which code path is responsible before reading code line by line.

- Unknown location → `find` with a plain-language description of the behavior plus `grep_keywords`. If `find` reports the judge model is unavailable, fall back to `grep` for identifiers, table names, config keys, message strings.
- Known symbol → `lsp` with `action: definition`, then `action: references` to see every caller, `action: implementation` for interface dispatch. A bug in a shared helper is a blast-radius bug; `references` tells you whether the fix belongs in the helper or in one caller.
- Unknown which layer lies → print/return the value at each layer boundary from the outside in. One binary search over the layers beats ten minutes inside one layer.
- Known regression window → `git log --oneline -- path/` then `git log -S'<offending literal>' -- .`, then bisect:
  ```sh
  git bisect start <bad> <good>
  git bisect run ./repro.sh   # exit 0 = good, nonzero = bad
  git bisect reset
  ```
  `repro.sh` must be non-interactive and cheap; a 30-second repro makes bisect a 20-minute job, a 10-second one makes it 5 minutes.
- Check the obvious first: uncommitted changes (`git status`), stale build artifacts, a dependency version that moved (`git diff` on the lockfile), environment drift between where it works and where it fails.

## 4. Minimize

Shrink the repro to the smallest case that still fails. Cut input size, delete unrelated calls, stub the collaborators, drop one field at a time. The minimal repro is the artifact that makes the root cause obvious, and it becomes the regression test. If a 500-line input fails but a 3-line one passes, the diff between them is the bug's address.

## 5. Hypothesize, then falsify

Write the hypothesis as a falsifiable sentence: "because X is nil when Y has not run, the loop skips the retry." Then design the **cheapest experiment that could disprove it** — an assert, a log line, an `evaluate` in the debugger, a unit call with the suspect input. Change one variable per run. If the experiment confirms the hypothesis, the root cause is still not established: ask why X is nil, and again, until you reach the invariant that should have held and did not.

Discard the whole "maybe it's" pattern: if two hypotheses are equally cheap to test, test the one that is falsifiable without new code.

## 6. Inspect live state instead of guessing at it

For anything involving state, control flow, or timing, prefer the `debug` tool to adding print statements you later have to delete:

- `action: launch` with `program` (plus `args`, `cwd`), or `action: attach` with `pid`/`port`.
- `action: set_breakpoint` with `file`+`line`, or `function` for a named entry point; add `condition` to stop only on the failing iteration rather than the first one.
- At the stop: `stack_trace`, `scopes`, `variables`, `evaluate` to read the actual values and call functions.
- "Who writes this field?" → `data_breakpoint_info` with `name`, then `set_data_breakpoint` with the returned `data_id`. That answers in one hit what twenty log lines would.
- Step with `continue`, `step_over`, `step_in`, `step_out`; read captured stdout with `output`; always `terminate` when done.

Print statements remain right when the target cannot be debugged (production, a container without adapters, a language with no adapter installed) or when the observation is "how often", not "what value".

## 7. Fix the invariant, not the symptom

Name the invariant that was violated — "this map always has an entry for every id returned by `query()`" — and restore it where it was broken.

Symptom fixes are the failure mode here, and they all look like:

- a null check added where the value should never have been null;
- a `try/catch` that swallows the error instead of handling the condition;
- a special case for the input in the bug report;
- a retry, a `sleep`, or a widened timeout;
- a fix in the caller that leaves the helper's contract broken for the next caller.

If the correct fix is upstream of where the exception was thrown, fix it upstream and let the invariant violation propagate loudly. Suppressing it converts a crash into silent corruption.

Before editing, run `lsp action: references` on whatever you are changing. A fix that satisfies one caller and breaks three is the second most common way to close a bug with a new one.

## 8. When you cannot reproduce

Work the ladder, in order, and stop as soon as one rung yields a lead:

1. Reproduce in the reporter's exact environment — same OS, runtime version, flags, data. Most "cannot reproduce" cases are environment differences.
2. Diff the environments: versions, lockfiles, feature flags, locale, timezone, file casing, clock.
3. Instrument for the next occurrence: add structured logging at the invariant boundary with the data needed to diagnose it, behind a flag you can enable in production. Ship the instrumentation, not a guess.
4. Add an assertion for the invariant you believe is violated, plus the surrounding values in the message.
5. Only then declare it unreproducible — with what you ruled out, and what evidence would settle it.

Never "fixed" a bug you could not reproduce by changing nearby code.

## 9. Flaky, racy, and Heisenbugs

- Establish the rate first: `for i in $(seq 50); do ./repro.sh || echo "FAIL $i"; done` and read the seeds/order. "Roughly 1 in 10" is data; "sometimes fails" is not.
- Randomize the order and the timing deliberately: run the suite reversed, with a fixed seed captured so a failure is replayable.
- Suspect the three usual causes in this order: shared mutable state across tests, real timing dependence (wall clock, timeouts, I/O, thread/worker scheduling), and unordered concurrency (two writes, lost close, missing await).
- For concurrency: capture the interleaving, do not just the outcome. Timestamped logs of the events that matter turn a race into a checkable sequence. Then make it deterministic — a barrier, an injected delay, or a single-threaded scheduler — and fix it against that deterministic repro.
- A bug that vanishes under the debugger is a timing or memory-lifetime bug you have now perturbed; keep the non-debug repro as the gate.
- Never commit a retry loop or a `sleep` to make a flaky test green. Fix the synchronization or make the test wait on the actual condition.

## 10. Performance regressions

Read `skill://optimization` instead of improvising: measure first, attribute the bottleneck, macro-optimize before micro. Its protocol is better than ad-hoc timing.

## 11. Close it out

Before reporting done, all of these must be true:

- The original repro command now passes. Paste its output.
- The narrowest existing test covering the touched code passes: `pytest path/to/test_file.py`, `go test ./pkg/...`, `cargo test -p crate`. Then the project's standard check for the files you edited (`lsp action: diagnostics`, or the build/typecheck command).
- A regression test exists that fails on the pre-fix code and passes on the post-fix code. Verify the first half by stashing or reverting the fix — a test that passes both ways guards nothing. Put it in the existing test file and style for that module; if the repo has no test harness for that path, say so explicitly in the report rather than inventing one.
- The report states: root cause, the invariant that was broken, the fix, why the fix is at that layer, and what the new test asserts.

## Guardrails

- One bug at a time. Unrelated smells get noted, not fixed, so the diff stays reviewable and the causal claim stays clean.
- Do not edit the test to match new behavior unless the test encoded the bug. If it did, say so and explain why the new expectation is the contract.
- Do not fix more than you can prove. Two proven changes beat five speculative ones.
- A fix that needs a comment explaining why the guard is safe is usually the wrong guard.
- If a fix lands and the bug returns, your root cause was wrong; restart at §3 rather than stacking a second patch on the first.
