---
name: test-driven-development
description: "TDD: enforce RED-GREEN-REFACTOR, tests before code."
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [testing, tdd, development, quality, red-green-refactor]
    related_skills: [systematic-debugging, plan, subagent-driven-development]
---

# Test-Driven Development (TDD)

## Overview

Write the test first. Watch it fail. Write minimal code to pass.

**Core principle:** If you didn't watch the test fail, you don't know if it tests the right thing.

**Violating the letter of the rules is violating the spirit of the rules.**

## When to Use

**Always:**
- New features
- Bug fixes
- Refactoring
- Behavior changes

**Exceptions (ask the user first):**
- Throwaway prototypes
- Generated code
- Configuration files

Thinking "skip TDD just this once"? Stop. That's rationalization.

## The Iron Law

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

Write code before the test? Delete it. Start over.

**No exceptions:**
- Don't keep it as "reference"
- Don't "adapt" it while writing tests
- Don't look at it
- Delete means delete

Implement fresh from tests. Period.

## Red-Green-Refactor Cycle

### RED — Write Failing Test

Write one minimal test showing what should happen.

**Good test:**
```python
def test_retries_failed_operations_3_times():
    attempts = 0
    def operation():
        nonlocal attempts
        attempts += 1
        if attempts < 3:
            raise Exception('fail')
        return 'success'

    result = retry_operation(operation)

    assert result == 'success'
    assert attempts == 3
```
Clear name, tests real behavior, one thing.

**Bad test:**
```python
def test_retry_works():
    mock = MagicMock()
    mock.side_effect = [Exception(), Exception(), 'success']
    result = retry_operation(mock)
    assert result == 'success'  # What about retry count? Timing?
```
Vague name, tests mock not real code.

**Requirements:**
- One behavior per test
- Clear descriptive name ("and" in name? Split it)
- Real code, not mocks (unless truly unavoidable)
- Name describes behavior, not implementation

### Verify RED — Watch It Fail

**MANDATORY. Never skip.**

```bash
# Use terminal tool to run the specific test
pytest tests/test_feature.py::test_specific_behavior -v
```

Confirm:
- Test fails (not errors from typos)
- Failure message is expected
- Fails because the feature is missing

**Test passes immediately?** You're testing existing behavior. Fix the test.

**Test errors?** Fix the error, re-run until it fails correctly.

### GREEN — Minimal Code

Write the simplest code to pass the test. Nothing more.

**Good:**
```python
def add(a, b):
    return a + b  # Nothing extra
```

**Bad:**
```python
def add(a, b):
    result = a + b
    logging.info(f"Adding {a} + {b} = {result}")  # Extra!
    return result
```

Don't add features, refactor other code, or "improve" beyond the test.

**Cheating is OK in GREEN:**
- Hardcode return values
- Copy-paste
- Duplicate code
- Skip edge cases

We'll fix it in REFACTOR.

### Verify GREEN — Watch It Pass

**MANDATORY.**

```bash
# Run the specific test
pytest tests/test_feature.py::test_name -v

# Then run ALL tests to check for regressions
pytest tests/ -q
```

Confirm:
- Test passes
- Other tests still pass
- Output pristine (no errors, warnings)

For web/app UI changes: passing tests are necessary but not sufficient. After tests pass, verify the actual running target the user will inspect (Docker container, staging/production URL, reverse proxy port, mobile viewport). Rebuild/restart/deploy if needed, fetch the served page/assets, then use a browser to click the changed path and confirm the visible behavior and console state before saying done. See `references/deployed-web-ui-verification.md` for the concrete checklist.

**Test fails?** Fix the code, not the test.

**Other tests fail?** Fix regressions now.

### REFACTOR — Clean Up

After green only:
- Remove duplication
- Improve names
- Extract helpers
- Simplify expressions

Keep tests green throughout. Don't add behavior.

**If tests fail during refactor:** Undo immediately. Take smaller steps.

### VERIFY RUNTIME — Check the Real Thing

For changes to a live website, containerized app, bot, scheduled job, or service, tests are necessary but not enough. Verify the exact runtime surface the user will judge:

1. Identify the actual target: deployed URL, reverse-proxy port, Docker Compose service, production/staging host, or scheduler storage.
2. Rebuild/restart/reload that target if needed; do not assume source edits are live.
3. Probe the target directly (`curl`, health endpoint, served HTML/static asset version, scheduler list, etc.).
4. Use browser automation for visible UI behavior: click the actual button, open the modal, confirm map/form/content appears, and inspect console errors.
5. Report the exact target verified and the real output observed.

**Pitfall:** Passing tests on a local dev server does not prove the Docker/reverse-proxied/deployed website changed. If the user checks the public/running site and sees old behavior, the verification failed.

### Repeat

Next failing test for next behavior. One cycle at a time.

## Why Order Matters

**"I'll write tests after to verify it works"**

Tests written after code pass immediately. Passing immediately proves nothing:
- Might test the wrong thing
- Might test implementation, not behavior
- Might miss edge cases you forgot
- You never saw it catch the bug

Test-first forces you to see the test fail, proving it actually tests something.

**"I already manually tested all the edge cases"**

Manual testing is ad-hoc. You think you tested everything but:
- No record of what you tested
- Can't re-run when code changes
- Easy to forget cases under pressure
- "It worked when I tried it" ≠ comprehensive

Automated tests are systematic. They run the same way every time.

**"Deleting X hours of work is wasteful"**

Sunk cost fallacy. The time is already gone. Your choice now:
- Delete and rewrite with TDD (high confidence)
- Keep it and add tests after (low confidence, likely bugs)

The "waste" is keeping code you can't trust.

**"TDD is dogmatic, being pragmatic means adapting"**

TDD IS pragmatic:
- Finds bugs before commit (faster than debugging after)
- Prevents regressions (tests catch breaks immediately)
- Documents behavior (tests show how to use code)
- Enables refactoring (change freely, tests catch breaks)

"Pragmatic" shortcuts = debugging in production = slower.

**"Tests after achieve the same goals — it's spirit not ritual"**

No. Tests-after answer "What does this do?" Tests-first answer "What should this do?"

Tests-after are biased by your implementation. You test what you built, not what's required. Tests-first force edge case discovery before implementing.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Too simple to test" | Simple code breaks. Test takes 30 seconds. |
| "I'll test after" | Tests passing immediately prove nothing. |
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |
| "Already manually tested" | Ad-hoc ≠ systematic. No record, can't re-run. |
| "Deleting X hours is wasteful" | Sunk cost fallacy. Keeping unverified code is technical debt. |
| "Keep as reference, write tests first" | You'll adapt it. That's testing after. Delete means delete. |
| "Need to explore first" | Fine. Throw away exploration, start with TDD. |
| "Test hard = design unclear" | Listen to the test. Hard to test = hard to use. |
| "TDD will slow me down" | TDD faster than debugging. Pragmatic = test-first. |
| "Manual test faster" | Manual doesn't prove edge cases. You'll re-test every change. |
| "Existing code has no tests" | You're improving it. Add tests for the code you touch. |

## Red Flags — STOP and Start Over

If you catch yourself doing any of these, delete the code and restart with TDD:

- Code before test
- Test after implementation
- Test passes immediately on first run
- Can't explain why test failed
- Tests added "later"
- Rationalizing "just this once"
- "I already manually tested it"
- "Tests after achieve the same purpose"
- "Keep as reference" or "adapt existing code"
- "Already spent X hours, deleting is wasteful"
- "TDD is dogmatic, I'm being pragmatic"
- "This is different because..."

**All of these mean: Delete code. Start over with TDD.**

## Verification Checklist

Before marking work complete:

- [ ] Every new function/method has a test
- [ ] Watched each test fail before implementing
- [ ] Each test failed for expected reason (feature missing, not typo)
- [ ] Wrote minimal code to pass each test
- [ ] All tests pass
- [ ] Output pristine (no errors, warnings)
- [ ] Tests use real code (mocks only if unavoidable)
- [ ] Edge cases and errors covered
- [ ] If the change affects a running website/service, verified the **actual running target** the user will see (container, deployed port, reverse-proxy URL, or production/staging URL) — not only a local dev server
- [ ] If static assets changed, verified the running target serves the new asset version/cache-buster and the browser loads that version
- [ ] If UI behavior changed, used browser interaction/DOM checks against the running target to confirm visible behavior, not just HTML snapshots or unit tests

Can't check all boxes? You skipped TDD/verification. Start over or explicitly report the blocker.
- [ ] Browser interaction verified on the real target, including console check

Can't check all boxes? You skipped TDD. Start over.

## When Stuck

| Problem | Solution |
|---------|----------|
| Don't know how to test | Write the wished-for API. Write the assertion first. Ask the user. |
| Test too complicated | Design too complicated. Simplify the interface. |
| Must mock everything | Code too coupled. Use dependency injection. |
| Test setup huge | Extract helpers. Still complex? Simplify the design. |

## Hermes Agent Integration

### Django template/UI changes

For Django-rendered UI changes, follow the same RED-GREEN-REFACTOR cycle with `TestCase` assertions against real rendered pages before editing templates. Assert visible copy, DOM hooks (`data-*` attributes), removed old blocks, and ordering of key sections. After tests pass, perform browser verification for JavaScript interactions and static assets. See `references/django-template-ui-tdd.md` for a compact checklist and examples.

When a requested UI change is a space-saving refactor, do **not** flatten the design into plain text unless the user explicitly asks for plain text. Preserve the original visual affordance/class of component where possible: if the old UI used pill/button-like chips with icons/checkmarks, the compact version should still read as designed chips, and the test should assert durable hooks for those affordances (for example `amenity-token`, icon/checkmark spans, and the expand/collapse button) rather than only asserting text and `data-*` toggles. Browser verification should include a visual check/screenshot of the exact section, not only DOM state.

When adding scroll-triggered visual interaction (listing cards, category chips, landing sections), test durable animation hooks first, implement IntersectionObserver + reduced-motion fallbacks, bump static cache-busters, then verify the running site by scrolling in a mobile browser and checking both screenshots and computed animation state. Do not treat “animation fired” as sufficient: visually judge whether the motion fits the site’s design system. For marketplace/travel listing pages, avoid gimmicky 3D/blur/fly-at-user effects unless explicitly desired; prefer subtle fade + upward lift/glide. See `references/scroll-animation-ui-verification.md` for the checklist and pitfalls.

### Running Tests

Use the `terminal` tool to run tests at each step:

```python
# RED — verify failure
terminal("pytest tests/test_feature.py::test_name -v")

# GREEN — verify pass
terminal("pytest tests/test_feature.py::test_name -v")

# Full suite — verify no regressions
terminal("pytest tests/ -q")
```

### Django model/form/admin changes

For Django feature work that touches models, forms, admin, or templates, keep the RED-GREEN loop concrete and end-to-end:

1. Write a failing `django.test.TestCase` first for the user-visible/admin-visible behavior, not just the model field. Good assertions include: required form fields, saved model values, admin checklist fields, template guidance text, and validation rules.
2. Run the narrow test and verify it fails for the missing behavior.
3. Implement the minimum model/form/template/admin changes.
4. Run the narrow test until green.
5. Generate/check migrations with `python manage.py makemigrations <app> --noinput` when model fields changed.
6. Run the full suite plus `python manage.py check` before reporting completion.

Pitfall: do not treat adding model fields as complete until the public form, admin review workflow, migration, and tests all agree. For anti-fraud / approval workflows, tests should assert both host-facing prerequisites and admin-only checklist fields so the approval process cannot silently become documentation-only.

### With delegate_task

When dispatching subagents for implementation, enforce TDD in the goal:

```python
delegate_task(
    goal="Implement [feature] using strict TDD",
    context="""
    Follow test-driven-development skill:
    1. Write failing test FIRST
    2. Run test to verify it fails
    3. Write minimal code to pass
    4. Run test to verify it passes
    5. Refactor if needed
    6. Commit

    Project test command: pytest tests/ -q
    Project structure: [describe relevant files]
    """,
    toolsets=['terminal', 'file']
)
```

### With systematic-debugging

Bug found? Write failing test reproducing it. Follow TDD cycle. The test proves the fix and prevents regression.

Never fix bugs without a test.

## Testing Anti-Patterns

- **Testing mock behavior instead of real behavior** — mocks should verify interactions, not replace the system under test
- **Testing implementation details** — test behavior/results, not internal method calls
- **Happy path only** — always test edge cases, errors, and boundaries
- **Brittle tests** — tests should verify behavior, not structure; refactoring shouldn't break them

## Final Rule

```
Production code → test exists and failed first
Otherwise → not TDD
```

No exceptions without the user's explicit permission.
