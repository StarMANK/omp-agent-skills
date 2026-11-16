---
name: feature-development
description: Use when adding a feature, new endpoint, CLI command, module, integration, or any behavior that did not exist before. Reconnoiter existing patterns first, map every touchpoint with LSP references, build a thin end-to-end walking skeleton before deepening, handle edge cases explicitly, cut over cleanly with no compatibility shims, and prove it by exercising the real surface.
allowed-tools: read, edit, write, bash, grep, glob, eval, task, todo, lsp, web_search, ask
---

# Feature development

Ship the smallest correct thing that is complete. Half an interface is worse than none: it has all the coupling and none of the guarantee.

## 1. Recon before writing a line

The dominant failure is inventing a second way to do something this codebase already does. Read first, and read enough that the answer is boring.

- Unknown location → `find` with a description of the behavior you are adding and `grep_keywords` of the identifiers you expect (`find.enabled` is `auto`; when it reports the judge model unavailable, use `grep`).
- `glob` for sibling files: a new handler goes where the other handlers live, named the way they are named.
- `lsp action: symbols` with `file: "*"` and a `query` for the workspace symbol list; `lsp action: implementation` on an interface to enumerate every type that already satisfies it.
- `task` with `agent: scout` for an unfamiliar subsystem — one scout per distinct area, in a single batch, while you read the files you already know. Ask it for the pattern, the registration points, and the test layout, not a summary.

Pick the closest existing analogue and copy, in this order of importance: file placement and naming, error-handling shape, how it is registered/wired/discovered, logging and diagnostics, test structure. Only deviate where the new behavior genuinely differs, and be able to say which existing file you imitated.

Two competing patterns in the repo? Prefer the one with more recent commits touching it, then the one the adjacent module uses. Say which you copied and which you did not.

## 2. Map the touchpoints before you write

Enumerate every place the new concept must be known. A feature that works in the happy path but is missing from one of these is a bug report in waiting:

- Callers and implementors of anything you change: `lsp action: references` on the exported symbol. This is the only correct way to enumerate them; do not text-search and hope.
- Exhaustive matches: `switch`/`match` statements, enums, discriminated unions, tagged types, sealed classes.
- Boundaries: (de)serialization, wire/JSON/protobuf field names, database schema and migrations, cache keys, CLI flag tables, config keys and their defaults and docs, permission/authz tables, metrics and audit events.
- Registration: plugin/module manifests, dependency-injection wiring, index/barrel files, router tables, command registries, i18n catalogs.
- Anything that enumerates the concept: list screens, filter dropdowns, validation allowlists, help text.

Write these as the `todo` phases before implementing, so no step can be silently dropped.

## 3. Shape the data before the code

Decide, and state in one short comment or docstring at the type, the load-bearing invariants: which type is the truth, who owns it, who may mutate it, what is always non-null, what is derived and must never be persisted. Then encode them: parse at the boundary into a typed value, keep the invalid state unrepresentable where the language makes that cheap. If the request has a schema, generate or reuse the existing validator rather than hand-rolling checks in three places.

## 4. Walking skeleton first

Build one thin path from the entry point to the effect and back, for the simplest interesting case, before writing any deep layer:

- CLI: flag parsed → command dispatched → prints the right thing.
- HTTP: route registered → handler → response serialized, tested with a real request.
- UI: event → state change → render.
- Library: public function → real I/O → returns a real value.

Wire it for real at every layer. The point is to discover integration surprises — auth, serialization, lifecycle, configuration, how things actually get registered — while there is still almost nothing to change. Building the deepest, purest layer first and bolting on the interface last is the expensive order.

## 5. Deepen in dependency order

Extend outward from the skeleton, one behavior per step, keeping the build and the existing tests green after each step. Order: parse/validate → core transform → persistence → presentation → error paths → concurrency and caching only if a measurement or the spec demands it. Cache, retry, and generalization are earned, not assumed.

Stop when the requested behavior is complete. Extra knobs, extra configuration, and speculative abstraction are scope creep: they cost review effort and carry no requirement.

## 6. Edge-case pass

Walk each new path and give an explicit answer — handle it, or state why it cannot happen:

- empty / missing / zero / very large input
- duplicate keys, concurrent invocation of the same operation, two callers writing the same record
- partial failure: the write succeeded and the side effect failed; the process died mid-operation
- permission denied, resource not found, upstream timeout, malformed data from a source you do not control
- cancellation and cleanup: does a half-started operation leave anything behind?

Fail loudly with a message that names the thing that was missing and where it was looked for. "Something went wrong" costs the next engineer an hour.

## 7. Clean cutover

When the feature replaces existing behavior:

- Migrate every caller in the same change. Use `lsp action: rename` (previews with `apply: false`) or `action: rename_file` so the language server updates references and sends the correct LSP rename notifications — safer than text replacement.
- Delete the code your change made obsolete: old functions, unused branches, compatibility aliases, re-exports kept "just in case", now-dead fields. Code that is obsolete because of your change is yours to remove.
- Update contract tests whose stated behavior changed. Do not re-pin an assertion to whatever the new code emits: change it only where the contract itself moved, and keep behavior/wording assertions out of existence rather than re-asserting them.
- No `// TODO: remove after migration`, no dual-write flag, no `legacy_` shim unless the user asked for a staged rollout.

Unrelated code you did not make obsolete stays untouched.

## 8. Prove it works

Type-check and the existing suite prove you broke nothing; they do not prove the feature works. Exercise the real surface:

- Start the program or service and drive the actual entry point: real request, real CLI invocation, real query against a real fixture.
- Concrete input → concrete expected output, written down before you run it.
- Check the negative path at least once: bad input → the error message you intended.
- For a UI or TUI, look at the rendered result; no assertion substitutes for it.
- No runtime for the changed surface: a throwaway script that calls the real function and prints the result. Report the limitation rather than claiming coverage you did not get.

Add a new test only where behavior is genuinely uncertain (edge cases, the tricky transform, a bug path), in the module's existing test file and harness. Skip tests that restate the type system or echo the source. Delete throwaway scripts once the proof is captured, unless they earn a permanent place.

## 9. Reporting

End with: the files changed and why, the invariant comment you added if any, the exact command and output that proves the feature, what you deliberately did not build, and any place the repo's existing pattern was a poor fit (so the next change can decide differently with evidence).

## Guardrails

- No stubs, placeholders, mocked internals, or `TODO`s left in the shipped path. If a slice truly cannot land, say so plainly rather than narrowing the feature silently.
- Never invent a second convention for something this repo already does; if the existing one is wrong, name it and propose the fix rather than quietly forking it.
- Prefer boring: the standard library, the existing helper, the existing storage, the pattern already in the adjacent file.
- Reuse beats new code — a one-line "no equivalent utility exists here" is the only justification for writing the third variant of a date parser.
- If a step takes more than about 30 minutes without a runnable result, you built the wrong slice; cut it smaller and get the skeleton moving again.
- Never claim a feature works on the strength of a compile or a typecheck.
