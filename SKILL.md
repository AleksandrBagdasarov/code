---
name: code
description: Core software-engineering principles that hold for every kind of program, in any language, including CLIs, libraries, services, scripts, data pipelines, embedded and systems code, games, mobile and desktop apps. Covers how to work (think before coding, simplicity first, surgical changes, goal-driven execution with verifiable success criteria) and what good code is (correctness and invariants, state and data, boundaries and error handling, abstraction and coupling, naming, comments, testing, performance, concurrency, safe change). Use this skill whenever writing, modifying, refactoring, debugging or reviewing code, planning an implementation, or judging code quality, even when the user doesn't ask for "best practices". Language- and framework-specific skills add detail on top of it; this is the shared floor beneath them.
---

# Engineering core

These principles apply whatever the program is: a 30-line script, a kernel driver, a game loop, a payment service, a compiler, a mobile app. Frameworks and languages change how a principle is *expressed*, never whether it applies. When a more specific skill (a language, framework or database) also applies, use both: that skill says *how* in its ecosystem, this one says *what matters and why*.

Treat everything here as judgment, not law. Each rule exists to reduce one of three costs: **bugs** (the program does the wrong thing), **change cost** (the next person, often you, can't safely modify it), and **waste** (code, time or resources spent on things nobody needed). When a rule doesn't reduce any of those in the situation at hand, it doesn't apply.

---

## Part 1 — How to work

These four habits matter more than any code-level rule, because most bad code comes from solving the wrong problem, or solving the right one with too much.

### 1. Think before coding

Understand the problem before producing a solution.

- Restate the goal in one or two sentences. If you can't, you don't understand it yet.
- Read the surrounding code first: how does this codebase already solve similar problems, name things, handle errors, test? Fitting in beats being locally "better".
- State your assumptions explicitly. When a requirement is ambiguous and the choice matters (data loss, public API shape, security, cost), ask instead of guessing. When it doesn't matter, pick the conventional option and say so.
- When there are real trade-offs between approaches, name them briefly and recommend one. Push back if a simpler approach than the one requested would meet the actual need.
- Find the edges early: what inputs, sizes, failure modes and concurrent uses must this handle?

### 2. Simplicity first

Write the least code that fully solves the stated problem.

- No speculative features, options, hooks or configurability that nobody asked for. Each one is code to read, test and maintain forever (YAGNI).
- No abstraction for a single use. An interface with one implementation, a factory that builds one thing, or a base class with one subclass is usually just indirection.
- Don't add handling for situations that cannot happen; do handle the ones that can (see Part 2, Boundaries).
- Prefer boring, well-known solutions and the standard library over clever ones and new dependencies.
- If a solution feels big, step back: 200 lines that could be 50 should be rewritten as 50. Simple is not the same as easy: finding the simple design often takes more thought, not less.

### 3. Surgical changes

Change only what the task requires, and leave everything else as you found it.

- Every changed line should trace back to the request. Unrelated renames, reformatting, refactors and "while I'm here" fixes bloat the diff, hide the real change from reviewers and add risk.
- Match the existing style and patterns of the file and project, even where you'd personally choose differently.
- Clean up what *your* change made obsolete (unused imports, dead helpers, stale comments). Pre-existing problems you notice but weren't asked to fix: mention them, don't silently change them.
- Keep public interfaces, data formats and behavior stable unless changing them is the point. When you must change them, say so explicitly.

### 4. Goal-driven execution

Turn the task into something you can verify, then verify it.

- Define success before starting: "fix the bug" becomes "a test reproduces the bug, and it passes after the fix without breaking others"; "make it faster" becomes "p95 of X drops below Y on this input".
- For multi-step work, write a short plan where each step has its own check (compiles, tests pass, output matches, benchmark improves).
- Run the check. Don't report success on code you haven't executed, compiled or tested when you could have; if you couldn't, say exactly what is unverified.
- When a check fails, find the cause before changing more code. Guess-and-patch loops create the very complexity Part 2 warns against.

---

## Part 2 — What good code is

### Correctness comes first
A fast, elegant program that is wrong is worthless.

- Know the **invariants**: what must always be true about the data and state (a balance never negative, a list always sorted, a handle always closed). Write them down in types, assertions or tests.
- **Make illegal states unrepresentable** where the language allows it: an enum/sum type instead of several booleans that can contradict each other, a non-empty type instead of a list plus a comment, a parsed `Email` instead of a raw string passed around.
- Walk the edge cases every time: empty, one, many, maximum; zero and negative; duplicates; very large input; overflow and precision (never use binary floats for money); text encoding and Unicode; time zones, DST and clock skew; missing or null values; partial failure halfway through.
- Prefer code that is **obviously correct** over code that is cleverly correct. If the correctness argument needs a paragraph, the code probably needs restructuring.

### State and data
Most complexity is state: what can change, who changes it, and when.

- Design the **data structures first**; good data shapes make the code that uses them simple. Bad ones force every function to compensate.
- Keep **one source of truth** for each fact. Derived or cached copies must have a clear owner and a clear invalidation rule.
- Minimize **shared mutable state**. Mutation that is local and owned (a buffer inside one function, a builder before it's published) is fine and often the efficient choice. Mutation of data that other code also holds is where bugs live; prefer passing values or new copies across those boundaries.
- Narrow scope and lifetime: declare variables close to use, keep globals and singletons rare, release resources deterministically (`with`, `defer`, RAII, `try/finally`, `using`).

### Boundaries and error handling
A program meets the outside world at its edges: user input, files, network, other processes, hardware, the clock, other modules' APIs.

- **Validate and parse at the boundary, then trust inside.** Convert untrusted input into well-typed internal values once; don't re-check the same thing in every layer.
- **Fail fast and loudly on bugs.** A violated invariant should stop the operation with a clear message (assertion, panic, exception), not limp on with corrupted state.
- **Distinguish expected failures from bugs.** "File not found", "user not authorized", "item out of stock" are normal outcomes to handle and report. A null where none can exist is a bug to surface.
- **Never swallow errors.** Every caught error is handled (recovered, retried, translated) or propagated with added context (what was being attempted, with which identifiers). An empty `catch`/`except` or an ignored error return is almost always wrong.
- Handle errors at the level that can actually do something about them; lower levels add context and pass them up.
- Anything that can be retried (network calls, message handlers, jobs) should be **idempotent** or guarded, and anything that waits should have a **timeout**.
- Treat all external input as hostile: validate sizes and formats, never build code, queries, shell commands or paths by string concatenation of untrusted data, keep secrets out of code and logs, and grant the least privilege that works.

### Abstraction and coupling
Abstractions are for hiding complexity, not for showing structure.

- Good modules are **deep**: a small, simple interface hiding substantial implementation. Many thin layers that just pass calls through add cost without hiding anything.
- **Hide decisions likely to change** (storage format, third-party library, algorithm) behind an interface owned by the code that uses it. Depend on stable things; isolate volatile ones.
- **Separate decisions from effects**: keep the logic that decides what to do (pure, easy to test) apart from the code that does I/O, talks to hardware or mutates the world ("functional core, imperative shell").
- **DRY means one source of truth for each piece of knowledge**, not "no two similar-looking lines". Duplication is cheaper than the wrong abstraction: wait until a pattern appears about three times and you understand what really varies before extracting it.
- Prefer composition over inheritance; keep inheritance hierarchies shallow.
- SOLID is a set of heuristics for object-oriented designs: single responsibility (one reason to change), extend rather than edit stable code where variation is expected, subtypes must honor their parent's contract, small focused interfaces, depend on abstractions at volatile seams. Apply them when they reduce coupling, not as a checklist that multiplies classes.

### Names and readability
Code is read far more often than it is written, by people (and models) without your context.

- Names say **what something is or does in the problem's terms**: `retry_delay_ms`, `pending_orders`, `is_expired`, `parse_header()`. Units and meaning go in the name when the type doesn't carry them.
- Functions are verbs (`compute_total`), booleans read as predicates (`has_access`), collections are plurals. Length matches scope: `i` in a 3-line loop is fine; a module-level `data` is not.
- Use the vocabulary of the domain and of the existing codebase consistently: one concept, one name.
- Keep control flow flat: guard clauses and early returns instead of deep nesting; one level of abstraction per function.
- Replace magic numbers and strings with named constants that explain their meaning.
- Follow the language's idioms and the project's formatter and linter instead of hand-styling.

### Comments and documentation
- Comment **why**, not what: the reason for a non-obvious choice, a workaround and its issue link, an invariant, a performance or safety constraint. The code already says what it does.
- Document **contracts** at public boundaries: what a function expects, returns, raises, and any side effects, ownership or thread-safety rules.
- Delete comments that have become false; a wrong comment is worse than none.

### Testing
Tests are the executable form of "goal-driven execution".

- Test **behavior through public interfaces**, not implementation details, so refactors don't break tests that should still pass.
- For every bug fix, first write a test that reproduces the bug, then fix it.
- Keep tests **deterministic and fast**: inject time, randomness, I/O and network rather than depending on them; no sleeps to wait for things.
- Structure each test as arrange → act → assert, with a name that states the scenario and expected result ("returns empty list when no orders match").
- Cover the edge cases from the correctness section; most bugs live at boundaries, not in the happy path.
- Match the test level to the risk: many small unit tests for logic, fewer integration tests for the seams where components and real dependencies meet, a handful of end-to-end checks of critical paths.

### Performance
- **Measure before optimizing.** Profile to find the real hot spot; intuition about where time goes is usually wrong.
- Choose the right algorithm and data structure first; complexity class beats micro-tuning (a quadratic loop over 10⁶ items will not be saved by caching).
- Avoid gratuitous pessimization even without profiling: repeated work inside loops, one query or call per item instead of a batch (N+1), loading everything when you need a page, allocating in tight loops, unbounded growth of caches and queues.
- Every optimization that complicates code should be justified by a measurement and explained in a comment.

### Concurrency
- Shared mutable state plus concurrency is the main source of hard bugs. Prefer, in order: no sharing (each task owns its data), message passing, immutable shared data, and only then locks.
- When locking, keep critical sections small, take multiple locks in a consistent order, and never do slow I/O while holding a lock.
- Check-then-act sequences ("if not exists, create") are races unless made atomic.
- Every concurrent operation needs a story for cancellation, timeouts and partial failure.

### Changing code safely
- Make changes in small, reviewable, reversible steps, each leaving the program working.
- Separate refactoring from behavior change: refactor first (tests stay green), then change behavior.
- Keep backward compatibility for anything others depend on (public APIs, file and wire formats, CLI flags, stored data), or version it and migrate deliberately.
- Make the running program observable: meaningful logs with context, errors with enough detail to diagnose, metrics where operations matter.
- Add dependencies deliberately: each one is code you don't control, must update, and must trust.

---

## How the principles bend by program type

The principles are constant; the trade-offs shift with context. Recognize which context you're in.

| Context | What weighs more | Typical adjustment |
|---|---|---|
| Throwaway script / one-off tool | speed of writing, simplicity | minimal structure is fine; still validate inputs that could destroy data |
| Library / SDK / public API | interface stability, clear contracts | invest in naming, docs, compatibility; avoid leaking internal types |
| Long-running service | failure handling, observability | timeouts, retries with idempotency, graceful shutdown, structured logs |
| Data pipeline / batch job | reproducibility, idempotency | re-runnable steps, checkpoints, validate schemas at ingestion |
| Embedded / systems / real-time | determinism, resources | bounded memory, no allocation in hot paths, in-place mutation is normal |
| Games / graphics / simulations | frame-time performance | data-oriented layouts, mutation per frame, profile constantly |
| UI (web, mobile, desktop) | responsiveness, state consistency | single source of truth for UI state, never block the main/UI thread |
| CLI tools | predictable interface | clear exit codes, stdout for data and stderr for messages, scriptable output |
| Security-sensitive code | correctness over convenience | use vetted libraries, never roll your own crypto, least privilege |

---

## Review checklist

Use this when reviewing or finishing any change, your own or someone else's.

1. **Goal**: does it solve the stated problem, and only that? Is there evidence (test, run, output) that it works?
2. **Correctness**: edge cases, invariants, error paths, partial failure, concurrency handled?
3. **Simplicity**: anything speculative, over-abstracted or longer than it needs to be?
4. **Scope**: any unrelated changes mixed in? Does it match the codebase's existing style?
5. **Boundaries**: untrusted input validated once at the edge? Errors handled or propagated with context, never swallowed? Resources released?
6. **State**: one source of truth? Shared mutable state minimized?
7. **Readability**: would a newcomer understand the names and flow without asking? Do comments explain why?
8. **Tests**: behavior covered, deterministic, including the bug being fixed?
9. **Performance**: no obvious pessimizations; any optimization backed by measurement?
10. **Compatibility and safety**: public interfaces and data formats preserved or deliberately migrated? No secrets, no injection paths?

When reporting review findings, lead with what can cause wrong behavior (bugs, data loss, security), then what raises change cost, and only then style. Point to the specific line and explain the concrete consequence, not just the rule it breaks.

## Common smells and what they usually mean

| Smell | Usually means | Typical fix |
|---|---|---|
| Function that needs scrolling or does "A and B and C" | several responsibilities | extract steps with intention-revealing names |
| Deep nesting | mixed validation and logic | guard clauses, early return, extract |
| Several booleans describing one state | illegal states are representable | enum / sum type / state machine |
| Same fact stored in two places | no single source of truth | derive one from the other |
| Flag argument changing behavior (`process(x, True)`) | two functions in one | split into two named functions |
| Empty catch / ignored error return | hidden failures | handle, or propagate with context |
| Long parameter lists, data clumps | a missing type | introduce a small value type |
| Comment explaining *what* a block does | unclear code | rename or extract so the code says it |
| Interface with one implementation, layers that only forward calls | premature abstraction | inline until a second real use appears |
| Change requires edits in many unrelated places | scattered knowledge, tight coupling | gather the knowledge behind one module |
