# Testing Conventions

This file defines testing expectations for all projects. The essentials are summarized in `CONVENTIONS_CORE.md` (always loaded); load this full file when a task involves writing or changing tests and needs the complete TDD cycle, isolation, and organization detail.

---

## Philosophy

- **TDD always:** write tests first → confirm red → implement → confirm green → commit. No exceptions for functional changes.
- **Failing tests are victories** — they catch bugs before users do. Never treat a test failure as a problem to paper over.
- **Never claim false success** — if a test fails, report it immediately and completely.
- **Run tests as part of the TDD cycle** — after implementing, run the scoped suite automatically. Reserve full-suite runs for explicit user requests.
- **Every bug fix starts with a failing test** — reproduce the bug in a test first → confirm red → fix → confirm green. The test stays as a permanent guard so the bug can never silently return. This is the single highest-leverage habit for stability. It applies with most force to a production incident, where the test is what closes it out (`incident-conventions.md`).
- **Prove a new guard fails.** Confirming red is habitual for unit tests and routinely skipped for everything else — schema checks, CI gates, smoke tests, monitors. Feed it the exact defect it exists to catch and watch it fail; a guard only ever seen passing is indistinguishable from one wired to nothing. Often another check fires first and the new one never runs, which is itself the finding.
- **The harder case is a guard that is wired and still cannot fail** — it runs, it asserts, and it measures something adjacent to the defect. A UI check that ends a drag off the element it was testing, so the click never lands there; a value chosen where the distortion is symmetrical; a synthetic gesture whose step size clears the threshold under test on its first event; a correctness step followed by a safety margin wide enough to absorb the error that dropping it causes, so the assertion holds on code that never does the work. A cheaper version of the same failure is a guard that **filters for a set and then asserts over it**: when the filter matches nothing the loop body never runs, so the check is green precisely when the thing it guards has gone missing. The commonest instance wherever output is formatted is an assertion that a number is *present* where the contract is that it is *formatted*: `/price \d/` passes against `price 12.8399999`, so the rendering the rule exists to pin is exactly what goes unchecked. Anchor the format itself — `/\d+\.\d km/` — not a digit somewhere after the label. These pass against *deliberately broken* code, which is the only way to find them: break the specific behaviour, and if the assertion stays green, it was never evidence. Budget this for the checks you would cite when closing the work — not for every test.
- **A guard on state written now and read later cannot be broken from inside one interaction.** Mutate away the condition deciding whether to *record* something — a flag, a handoff, a cached target — and the test covering that interaction stays green, because nothing reads the value on that path; the leak surfaces on the next interaction of a different kind, one action away from the cause. The mutation then reads as proof the guard was unnecessary when it is merely deferred, which is worse than no mutation at all. A test for such a guard has to span from the write to the read.
- **A helper that encodes a domain assumption goes stale when the domain changes, and it fails as a plausible assertion rather than as a broken helper.** A fixture that picks "an empty point", "an unused id", "a free port", "a quiet hour" is asserting something about the system, and a later change can quietly make it false. The test then fails on the assertion it was aimed at — sending you to debug the code under test, which is fine — while the lie is in the fixture. So when a change redefines a region, a namespace or a schedule, audit the helpers that *select against* it in the same pass: they are inside the change's blast radius, not collateral to it.
- **A helper that *establishes* a precondition must witness the input, never the system's response.** Retrying a gesture until the camera glided, or polling until the queue drained, asks the thing under test whether it worked — so a genuine defect is reported as a harness failure that never reaches the assertion, and the mutation that should have exposed it comes back green. Check the input the system was actually given, and fail loudly in the harness's own terms when you could not give it; never skip, because a test that cannot be driven has to go red rather than quietly pass.
- **A fixture that reaches a state by a route the user cannot take is a lie that stays green until something moves.** Writing a *finished* record and opening the app to land on its summary screen, or seeding a session the login flow would reject, sets up a state the product never produces — it works only because some other code path is currently permissive enough not to notice. The test reads as covering the real thing and does not, and it fails later during an unrelated change, where it looks like that change's regression. Reach the state the way the user does, even when it costs a few more steps; if that is genuinely too slow, say in the fixture which route it is skipping and why it is equivalent.
- **Match the fake input's shape to the real input's, or say which shape you tested.** Synthetic drags, uploads, clocks and payloads default to convenient values — large steps, round sizes, whole seconds — and a defect that only appears at the awkward end of the range survives every run. The runner's own timing is part of this, on both sides: a paced loop that stretches under load can silently move the input outside the window the code reads, and a test that asks *has it moved yet?* across a wall-clock duration can span no tick of the thing it is watching at all — an animation frame, a poll, a scheduler pass — so a still-running system reads as finished. Measure motion in the unit the system advances in, not in milliseconds.
- **A timeout raised because an operation is inherently expensive belongs to the operation, not to one test of it.** Write it as a shared exported bound next to the slow thing, so every caller inherits it; a private constant in one test file leaves the sibling suite that exercises the same operation on the runner's default, failing at random under full-suite load. The same holds for any tolerance an operation earns — retries, poll intervals, size limits.

## Test Levels

Aim for a pyramid: **many unit, some integration, few E2E.** Each level catches what the ones below it can't; using a higher level to test logic a lower one covers is the main source of slow, brittle suites.

- **Unit** — one module in isolation, dependencies mocked. Fast and deterministic; the bulk of coverage. This is where business logic, conditionals, and edge cases get exercised.
- **Integration** — the seams between units: a service against a real (test) database, a module against its adapter, an API route through to persistence. Catches what unit tests structurally cannot — wiring, contracts, serialization, migrations, transaction boundaries. Do **not** mock the boundary under test; mock only what lies beyond it (third-party network, clock, randomness). Migrations and backfills are tested at this level — see `migration-conventions.md`.
- **E2E** — a handful of critical paths exercised end to end through the real entry point: a UI journey in a browser, a full CLI invocation, an API request flowing through to persistence and back (e.g. sign in → perform the core action → confirm the result). Slow and brittle by nature, so reserve them for paths whose breakage is unacceptable. Not a substitute for unit or integration coverage — if an E2E test is checking logic a unit test could, drop it down a level.

## Where Tests Run

The levels above describe what a test *is*. Where it runs is a separate question, and each place catches a different class of failure:

- **Local** — the TDD cycle. Every level runs here; this is where you get the fast feedback that makes TDD work. Always required.
- **CI** — the same suite, somewhere it can't be skipped or run against a dirty working tree. See "The CI Gate" below.
- **Against a deployed staging environment** — where E2E tests earn their keep most, because they exercise the real build, real infrastructure, real migrations, and real integration wiring. A suite that is green locally and green in CI can still fail here, and that failure is the one users would have hit.

A green suite is **not** the same as a verified change. Once a project has real users, code is also verified in a production-like environment before promotion — how much, and when that becomes required, is in `environment-conventions.md`.

## Running Tests

- **Scope to the change** — run the minimum suite that validates the change; full-suite runs are expensive and rarely needed.
- **Green phase:** check exit code and failure lines only — don't review every line of output.
- **One run at a time.** Two concurrent runs of the same suite contend for the same fixed ports, databases, fixtures and browser workers, and the failures that produces are indistinguishable from real ones. The cost is not the wasted run — it is every conclusion drawn while both were in flight, including the diagnosis of whatever you were actually investigating. It is easy to do by accident: one run left in the background, a second started in the foreground. Repeated runs go in sequence. Starting a second run *deliberately*, to reproduce contention, is an experiment whose result is about the runner rather than the code — never a way to run the suite. **If you remove the contention by making the shared resource per-run — a port, a database name, a scratch directory — memoise the choice somewhere every process of that run reads, such as an environment variable set before the workers fork.** A runner re-evaluates its config in *each* worker process, so a value drawn per evaluation is drawn again per worker and the workers address a resource nobody created: the entire suite fails at once, at a different address each time, which reads as a broken application rather than a broken config.

## Test Output

- **Lean by default** — configure test runners and reporters for minimal output: no per-test pass lines, no full stack traces on success, summary only. Verbosity should not increase token usage when everything is green.
- **Dig deeper on demand** — when a failure needs investigation, temporarily increase verbosity (e.g. `--reporter=verbose`, `--silent=false`, added `console.log`) to get the signal needed, then revert to lean defaults once resolved.
- **No test-internal logging by default** — do not add `console.log` or debug output inside tests unless actively troubleshooting. Remove it before committing.

## Background Process Cleanup

- **Stop what you started** — any long-running process spun up for testing (dev/app server, API server, database or emulator, container, headless browser) should be stopped when done, so it doesn't hold a port or resource on the next run.
- **How:** stop it in its own terminal (`Ctrl+C`), or kill it by match — `pkill -f "<the command you launched>"` on macOS/Linux, `Stop-Process` / `taskkill` on Windows. Match the specific command you started, not the whole runtime (kill `pkill -f "myapp serve"`, not every `node`/`python`/`java` process on the machine).
- **Why:** prevents "address already in use" / port-conflict and stale-state errors when starting the process again.

## Test Isolation

- **Isolate at the boundary** — tests must not depend on external state or running services. Mock or stub all dependencies beyond the unit or module under test.
- **The boundary moves with the level** — the rule above describes a *unit* test's boundary. An integration test's boundary sits further out: it keeps the seam it exists to verify (e.g. the real test database) and mocks only what lies beyond that. Never mock the thing you're trying to test.
- **Independent tests** — each test sets up its own state and cleans up after itself. No test should rely on execution order.
- **Never test against production.** A test suite pointed at a production database or a live third-party account will eventually write to it. Tests run against local or staging targets with synthetic data and sandbox credentials only (`environment-conventions.md`).
- **Flagged code is tested in both states.** A release flag's on-path and off-path each get coverage, and the off-path stays covered until the flag is deleted. You are not obliged to test every *combination* of flags — if a combination matters, that's the signal there are too many (`progressive-delivery-conventions.md`).

## Test Organization Principles

- **DO test:** business logic, complex conditionals, state management, CRUD operations, error handling, edge cases, win/loss conditions.
- **DON'T test:** simple DOM manipulation, pure delegating functions, static data constants, external library wrappers, animation/timer functions.
- **Adversarial mindset:** after writing happy-path tests, actively try to break your own code — boundary conditions (zero, null, max), invalid states, race conditions, failure modes.

## Coverage & Reliability

- **Coverage is a diagnostic, not a target** — use it to find untested branches and error paths, never as a number to chase. 100% coverage of trivial code is overkill; 70% that skips the failure modes is a gap. Judge by risk, not percentage.
- **Zero tolerance for flaky tests** — a test that passes and fails without a code change is worse than no test: it trains everyone to ignore red. Fix the nondeterminism or quarantine the test the moment it's spotted; never just re-run until green.
- **A different test failing each run, each passing alone, means resource contention** — usually the runner's worker count, which defaults to the machine's cores and says nothing about what one local database or container stack can serve. Cap the workers; an oversubscribed suite is contending rather than working, so this often costs no wall-clock time. Fix it in config, never with retries.
- **Reconstructing a failure is not observing it.** Rebuilding a failing scenario in a script — replaying a spec's steps against the library it drives, recomputing what a page *should* show — models your understanding of the system, and your understanding is what is currently wrong. It returns confident numbers that can point away from the cause. Ask the running system what it actually contains first: dump the rendered text, the response body, the row. Reconstruct only once you know what you are explaining.
- **Diagnose flakiness before attributing it.** A failure appearing right after your change is not evidence your change caused it. Re-run alone, reset shared state, then vary one factor at a time — otherwise a real pre-existing defect gets buried under an unrelated change.

## The CI Gate

- **Green before merge** — once a project has CI, the suite passing is an un-skippable gate, not a courtesy. In collaborative mode no change merges red; solo, the same gate catches the mistake you'd otherwise push. The gate itself and how it's enforced by mode live in `cicd-conventions.md` — this file owns what the suite *is*, that file owns when it *blocks*.

## Advanced Techniques — Reference, Not Rule

These are **not** part of the default expectation. They're tools to reach for when a specific project crosses a threshold that makes them worth the cost — the Tier 3 "level up when the codebase matures" trigger in `coding-conventions.md`. Adopt one when its trigger below is real; don't add them speculatively (YAGNI applies to test infrastructure too).

- **Property-based testing** — when a function's correctness is a rule over a large input space (parsers, serializers, math, encoders), generate cases instead of hand-picking them. Trigger: you keep finding edge cases the examples missed.
- **Contract testing** — when independently deployed services must agree on a shape, pin the contract so one side can't break the other silently. Trigger: a real service boundary with separate release cadences.
- **Mutation testing** — when you need to know whether the *tests* are any good, not just whether they pass. It measures whether your suite actually catches injected faults. Trigger: coverage looks high but bugs still escape.
- **Fuzzing** — when untrusted input hits a parser or decoder and a crash is a security event. Trigger: you're handling adversarial input at a trust boundary (`security-conventions.md`).
- **Load / performance testing** — when latency or throughput is itself a requirement. Trigger: real-user scale, or an SLA. Lives with `observability-conventions.md`.

## Project Overrides

Any deviation from these defaults is declared in that project's CLAUDE.md and takes precedence.
