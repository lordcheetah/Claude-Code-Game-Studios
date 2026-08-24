# Coding Standards

- All game code must include doc comments on public APIs
- Every system must have a corresponding architecture decision record in `docs/architecture/`
- Gameplay values must be data-driven (external config), never hardcoded
- All public methods must be unit-testable (dependency injection over singletons)
- Commits must reference the relevant design document or task ID
- **Commit messages**: Use Conventional Commits format — `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`. Reference the story or task ID in the body (e.g., `Story: EPIC-001-S02`).
- **Verification-driven development**: Write tests first when adding gameplay systems.
  For UI changes, verify with screenshots. Compare expected output to actual output
  before marking work complete. Every implementation should have a way to prove it works.

# Design Document Standards

- All design docs use Markdown
- Each mechanic has a dedicated document in `design/gdd/`
- Documents must include these 8 required sections:
  1. **Overview** -- one-paragraph summary
  2. **Player Fantasy** -- intended feeling and experience
  3. **Detailed Rules** -- unambiguous mechanics
  4. **Formulas** -- all math defined with variables
  5. **Edge Cases** -- unusual situations handled
  6. **Dependencies** -- other systems listed
  7. **Tuning Knobs** -- configurable values identified
  8. **Acceptance Criteria** -- testable success conditions
- Balance values must link to their source formula or rationale

# Testing Standards

## Test Evidence by Story Type

All stories must have appropriate test evidence before they can be marked Done:

| Story Type | Required Evidence | Location | Gate Level |
|---|---|---|---|
| **Logic** (formulas, AI, state machines) | Automated unit test — must pass | `tests/unit/[system]/` | BLOCKING |
| **Integration** (multi-system) | Integration test OR documented playtest | `tests/integration/[system]/` | BLOCKING |
| **Visual/Feel** (animation, VFX, feel) | Screenshot + lead sign-off | `production/qa/evidence/` | ADVISORY |
| **UI** (menus, HUD, screens) | Manual walkthrough doc OR interaction test | `production/qa/evidence/` | ADVISORY |
| **Config/Data** (balance tuning) | Smoke check pass | `production/qa/smoke-[date].md` | ADVISORY |

## Automated Test Rules

- **Naming**: `[system]_[feature]_test.[ext]` for files; `test_[scenario]_[expected]` for functions
- **Determinism**: Tests must produce the same result every run — no random seeds, no time-dependent assertions
- **Isolation**: Each test sets up and tears down its own state; tests must not depend on execution order
- **No hardcoded data**: Test fixtures use constant files or factory functions, not inline magic numbers
  (exception: boundary value tests where the exact number IS the point)
- **Independence**: Unit tests do not call external APIs, databases, or file I/O — use dependency injection

## What NOT to Automate

- Visual fidelity (shader output, VFX appearance, animation curves)
- "Feel" qualities (input responsiveness, perceived weight, timing)
- Platform-specific rendering (test on target hardware, not headlessly)
- Full gameplay sessions (covered by playtesting, not automation)

## CI/CD Rules

- Automated test suite runs on every push to main and every PR
- No merge if tests fail — tests are a blocking gate in CI
- Never disable or skip failing tests to make CI pass — fix the underlying issue
- **A run that executed zero tests must fail the build, never pass it.** A test
  runner that dies before executing anything — bad path, missing output directory,
  licensing failure, wrong flag — often writes no results file, or an empty one, and
  still exits 0. A CI check that only counts *failed* tests then reports green while
  nothing ran, and the suite silently stops protecting the project. Assert both that
  the results file exists and that its test count is greater than zero.
- Engine-specific CI commands:
  - **Godot**: `godot --headless --script tests/gdunit4_runner.gd`
  - **Unity**: `unity test --mode EditMode --report-format nunit,junit --timeout <seconds>`
    (Unity CLI — writes NUnit and/or JUnit XML, and supports `--filter` and `--coverage`.
    `--timeout` is disabled by default, so always set it in CI or a hung PlayMode test
    blocks the runner.) `--report-format` takes `nunit`, `junit`, or the comma pair
    `nunit,junit`; the CLI's own `--help` advertises `both`, which the parser rejects
    with exit 2. The CLI is marked *experimental* by Unity and still changing — treat
    Unity's own skill as the syntax source of truth over `--help`, and install it with
    `unity skill install claude-code --local`.

    **`unity test` presumes a machine whose Unity license is already active — it does not
    provision one.** On an ephemeral runner (GitHub-hosted, container, any fresh VM) holding
    only a Unity **Personal** license, every activation mode is closed: `--file` posts the
    .ulf to the licensing service and is rejected on hardware binding ("Machine bindings
    don't match"), `--personal` requires an interactive sign-in, `--serial` needs a serial
    Personal has none of, `--floating` needs a Pro/Enterprise license server, and Unity's
    licensing backend refuses service-account tokens for activation. Established 2026-08-24
    over five consecutive runner attempts on a real project, not inferred.

    So pick the runner to match the license: a **self-hosted** runner executing as a user
    account already signed in to Unity Hub needs no activation at all and is where
    `unity test` shines; a **hosted** runner on Personal should use
    `game-ci/unity-test-runner@v4`, which randomizes the machine ID and activates through the
    editor binary rather than the licensing endpoint. Switching a green game-ci pipeline to
    `unity test` without a self-hosted runner will break it.
  - **Unreal**: headless runner with `-nullrhi` flag
