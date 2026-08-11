---
id: qa-process-new-symbols-without-a-consumer
domain: qa
category: process
applies_to: [general]
confidence: verified
sources:
  - https://github.com/import-js/eslint-plugin-import/blob/main/docs/rules/no-unused-modules.md
  - https://github.com/jendrikseipp/vulture
  - https://knip.dev/explanations/why-use-knip
last_verified: 2026-08-11
related:
  [
    backend-common-change-impact-call-site-enumeration,
    qa-process-scope-purity-checks,
    testing-quality-tests-that-cannot-fail,
    infrastructure-agent-orchestration-worktree-isolated-workers,
  ]
---

# Finding the New Symbol Nothing Consumes

## When this applies

Work was split across parallel tasks, branches, or sessions where one side
builds a function/endpoint/export and another side is meant to call it, and you
are deciding whether the split is done. Also when a feature that passed every
gate renders nothing in production, or a completed endpoint has no caller.

Enumerating callers of a symbol whose contract you are *changing* →
[backend-common-change-impact-call-site-enumeration].

## Do this

1. **Build the census from the diff, programmatically.** List the public symbols
   the change adds, then for each count references in production files **outside
   its defining module**, excluding tests. Cross-module usage is the criterion
   the established tools use: `import/no-unused-modules` reports "exports without
   any static usage within other modules".

2. **Classify each zero before reporting it.** A zero is a finding only when a
   written artifact claims a cross-module consumer:

| The zero-consumer symbol | Read it as |
|--------------------------|------------|
| Its docstring, the task plan, or the API spec names a caller in another module | Integration gap — wire it, or reopen a task that will |
| Referenced only inside its own module | Internal helper — no action |
| Referenced only by its own tests, with no consumer claim | Not yet a finding; recheck after the consuming task lands |
| Reached by name at runtime (registry, `getattr`, entry point, DI container, route table, decorator registration) | Census blind spot — confirm by searching the name as a string, and record that this class is not statically enumerable |
| Exported for a consumer outside this repository (published package, plugin API) | Out of scope for this census |

3. **Run the census on the merged tree, not per branch.** Before merge the
   consumer lives in a sibling branch and every producer looks unconsumed; the
   count is only meaningful once all branch tips are in one tree.

4. **For each confirmed gap, name an owner and a time.** When the consuming task
   already finished, the wiring has no owner and stays unwritten — open it as its
   own task rather than appending it to a closed one
   ([infrastructure-agent-orchestration-worktree-isolated-workers]).

5. **Record the method beside the count** — "9 public symbols added, 14 with zero
   cross-module production references, 1 with a documented consumer" is
   reviewable; "no wiring gaps" is not.

## Edge cases

| Case | Then |
|------|------|
| The symbol is consumed through a re-export or facade | Count the facade's consumers too; the importing site names the facade, not the defining module |
| A framework decorator registers the function (HTTP route, CLI command, event handler) | The decorator is the consumer — the symbol is wired even at zero direct references |
| The language has no module-level export marker (Python `_`-prefix convention, Go capitalization) | Take the convention as the public set and say which convention you used; the census scope is otherwise unstated |
| The census tool reports far more zeros than the diff added | It is scanning the whole repo, not the diff — restrict it to the added symbols, or the one real gap is buried |
| The producer landed but its consumer task is still open | Not a gap yet; re-run the census when that task closes, and keep the symbol on the list until then |
| The consumer is a test fixture that stands in for production wiring | Still a gap — a fixture calling the symbol is what makes the suite green while production never reaches it ([testing-quality-tests-that-cannot-fail]) |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Take a green type-check, green suite, and clean merge as proof the new code is reachable | Run the cross-module consumer census on the merged tree | An unconsumed symbol type-checks, and its own tests call it, so every gate reports it as used |
| Report every zero-consumer symbol as a finding | Classify each zero against a written consumer claim first | Measured on one module: 14 zeros, 1 real gap — an unfiltered list buries the one that matters |
| Delete the zero-consumer symbols as dead code | Classify first, and apply the dynamic-dispatch row | Vulture states "code that is only called implicitly may be reported as unused", and rates functions at 60% confidence |
| Split parallel work by file ownership and leave the cross-file wiring unassigned | Name, at split time, which task writes each boundary-crossing call | A boundary call outside every task's file set is inside no task's definition of done |

## Sources

- https://github.com/import-js/eslint-plugin-import/blob/main/docs/rules/no-unused-modules.md — the rule reports "individual exports not being statically `import`ed or `require`ed from other modules in the same project"; `unusedExports` identifies "exports without any static usage within other modules" — cross-module reference, not any reference, is the criterion
- https://github.com/jendrikseipp/vulture — "Due to Python's dynamic nature, static code analyzers like Vulture are likely to miss some dead code. Also, code that is only called implicitly may be reported as unused"; confidence is 60% for functions, methods, classes and variables, which is why a raw zero is a candidate rather than a verdict
- https://knip.dev/explanations/why-use-knip — "Unused files, exports and dependencies pile up as projects grow"; in large or legacy projects "Knip may report false positives and require some configuration", so the census needs the classification step rather than a bare list
- Field measurement 2026-08-11 (Python module set built across parallel tasks): a full census of public functions found 14 with zero cross-module production references; filtering to those whose docstring or plan named an external caller left exactly 1 real integration gap — a signed-URL builder whose only documented consumer had already finished and never wired it
