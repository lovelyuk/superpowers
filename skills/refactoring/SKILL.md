---
name: refactoring
description: Use when improving code structure without changing behavior — before touching production code for cleanup
---

# Refactoring

## Overview

Refactoring without discipline breaks things. The goal is to change code structure while keeping behavior identical.

**Core principle:** Tests must pass before, during, and after every refactoring step. If tests don't exist, write them first.

**Announce at start:** "I'm using the refactoring skill."

## The Iron Law

```
BEHAVIOR MUST NOT CHANGE. TESTS PROVE IT.
```

If you don't have tests covering the code you're refactoring, write characterization tests first.

## When to Use

- Code is hard to understand or change
- Function or class is doing too many things
- Duplication exists that's causing maintenance pain
- Names don't reflect what the code actually does
- Code structure makes adding a new feature unnecessarily difficult
- Tech debt is slowing down the team

**Do NOT refactor when:**
- You're also adding new features (split into two PRs)
- Tests are failing (fix them first)
- You're under time pressure to ship a feature
- You don't understand what the code does yet

## Phase 1: Safety Net

**Before any changes:**

1. **Identify what needs tests**
   - Run coverage on the code you plan to touch
   - Any uncovered path is a potential silent breakage

2. **Write characterization tests** (if tests are missing)
   - Capture current behavior, not desired behavior
   - These tests are your safety net, not a spec

   ```python
   # Characterization test — document what the code DOES, not what it SHOULD do
   def test_process_order_current_behavior():
       result = process_order({"items": [], "user_id": 42})
       # Run it, see what it returns, lock that in
       assert result == {"status": "empty", "total": 0}
   ```

3. **Run the full test suite**
   - All tests must pass before you start
   - Note any flaky tests so you don't blame them on your changes

4. **Commit the tests separately**
   ```bash
   git add tests/
   git commit -m "test: add characterization tests before refactoring"
   ```

## Phase 2: Identify the Refactoring

**Name what you're doing.** Use standard patterns:

| Pattern | When to use |
|---------|------------|
| Extract Function | Code block that needs a name; used in multiple places |
| Extract Class | Object has too many responsibilities |
| Rename | Name doesn't reflect purpose |
| Inline | Abstraction isn't earning its keep |
| Move Function/Method | Function belongs in a different module/class |
| Replace Conditional with Polymorphism | Long if/switch chains on type |
| Extract Variable | Complex expression needs a name |
| Replace Magic Number | Literal value needs a constant |
| Introduce Parameter Object | Function takes too many related parameters |
| Replace Loop with Pipeline | Loop logic is easier to read as map/filter/reduce |

**Scope the work:**
- One refactoring at a time
- Changes should fit in a single focused commit
- If it's a big refactoring, break it into steps and commit each

## Phase 3: Execute in Small Steps

**Each step:**
1. Make one change
2. Run tests immediately
3. If tests pass → commit
4. If tests fail → revert immediately (don't debug, don't stack changes)

```bash
# Tight loop: change → test → commit or revert
git add -p          # stage just the change
pytest              # tests must pass
git commit -m "refactor: extract validate_user_input function"
```

**Step size guide:**
- Rename a variable → one commit
- Extract a function → one commit
- Move a function to another module → one commit
- Never bundle multiple refactorings into one commit

## Common Refactorings

### Extract Function

```python
# Before
def process_order(order):
    # validate
    if not order.get("user_id"):
        raise ValueError("missing user_id")
    if not order.get("items"):
        raise ValueError("empty order")
    # ... 40 more lines

# After
def validate_order(order):
    if not order.get("user_id"):
        raise ValueError("missing user_id")
    if not order.get("items"):
        raise ValueError("empty order")

def process_order(order):
    validate_order(order)
    # ... 40 lines, now readable
```

### Rename for Clarity

```python
# Before — what does d mean?
def calc(d, r):
    return d * r

# After
def calculate_revenue(daily_users, revenue_per_user):
    return daily_users * revenue_per_user
```

### Replace Magic Numbers

```python
# Before
if response.status_code == 429:
    time.sleep(60)

# After
HTTP_TOO_MANY_REQUESTS = 429
RATE_LIMIT_BACKOFF_SECONDS = 60

if response.status_code == HTTP_TOO_MANY_REQUESTS:
    time.sleep(RATE_LIMIT_BACKOFF_SECONDS)
```

### Replace Duplication

```python
# Before — same logic in two places
def send_welcome_email(user):
    subject = f"Welcome, {user.name}!"
    body = render_template("welcome.html", user=user)
    smtp.send(user.email, subject, body)

def send_password_reset_email(user):
    subject = "Reset your password"
    body = render_template("reset.html", user=user)
    smtp.send(user.email, subject, body)

# After
def send_email(user, subject, template):
    body = render_template(template, user=user)
    smtp.send(user.email, subject, body)

def send_welcome_email(user):
    send_email(user, f"Welcome, {user.name}!", "welcome.html")

def send_password_reset_email(user):
    send_email(user, "Reset your password", "reset.html")
```

## Red Flags — STOP

- Tests are failing after your change → revert immediately, don't stack more changes
- You're fixing bugs while refactoring → separate PR
- The refactoring is getting bigger than expected → stop, reconsider scope
- You've been at it for hours with no commits → steps are too large
- You're changing behavior "while you're in there" → that's a feature, not a refactoring

## Phase 4: Review

After completing the refactoring:

1. **Read the diff with fresh eyes**
   - Is the code actually clearer now?
   - Did you introduce any accidental behavior changes?

2. **Run the full test suite one final time**

3. **Delete the characterization tests** that were only there for safety (if they're testing internal implementation rather than behavior)
   - Or keep them if they document meaningful behavior

4. **Update the PR description**
   - List what was refactored and why
   - "No behavior changes" should be provable from the tests

## Commit Message Convention

```
refactor: <what changed structurally>

Not: what it does differently (it doesn't)
Yes: what's structurally different and why it's better

Examples:
  refactor: extract validate_payment function from checkout flow
  refactor: rename d/r params to daily_users/revenue_per_user
  refactor: replace magic status codes with named constants
  refactor: move email helpers to notifications module
```

## Related Skills

- **superpowers:test-driven-development** — write tests before refactoring if none exist
- **superpowers:systematic-debugging** — if refactoring reveals a bug
- **superpowers:verification-before-completion** — confirm behavior is unchanged before merging
- **superpowers:writing-plans** — plan large refactors as multi-step projects
