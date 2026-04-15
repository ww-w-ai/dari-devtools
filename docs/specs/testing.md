# Testing Principles -- Detailed Spec

> WHY: By default, when Claude is asked to write tests, it produces meaningless tests
> that verify internal method call counts, replace every dependency with a Mock,
> and only push coverage numbers up.
> Without this rule, every refactoring breaks the entire test suite, reaching
> a state where "test maintenance cost > cost of having no tests."

## SOURCE

| Evidence ID | Basis | Type | Source |
|-------------|-------|------|--------|
| E-IS-004 | Behavior-driven test design and Fresh Fixture pattern | INDUSTRY_STD | Gerard Meszaros, xUnit Test Patterns (2007) |
| E-IS-005 | Mock external dependencies only -- use real implementations for internal collaborators | INDUSTRY_STD | Martin Fowler, Mocks Aren't Stubs (2007) |
| E-IS-006 | Behavior-driven naming in Given-When-Then structure | INDUSTRY_STD | BDD (Behavior-Driven Development) |

## Scope

- paths: test files (`*.test.*`, `*.spec.*`, `test_*`, `*_test.*`)
- target: test code authoring, test strategy design, Mock scope decisions

## 1. Test Names Must Reveal the Cause of Failure

Test names include "what, under what condition, and the expected outcome."

You should be able to infer the cause of failure from the test name alone.
Use Given-When-Then as the default structure, adjusted to language/framework conventions.

### CONSTRAINT

- Vague test names: a `test_1 failed` notification cannot tell you which scenario broke -- debugging time spikes
- Implementation details in the name: all test names must change when the internal method name or call pattern changes

### Good

```python
# Condition + behavior + expected outcome included in the name
test_expired_token_returns_401_unauthorized
test_empty_cart_prevents_checkout
test_duplicate_email_raises_validation_error

# Group + scenario form
describe("OrderService.cancel")
  it("fully refunds orders before shipping starts")
  it("rejects cancellation for orders already in transit")
```

### Bad

```python
test_1
test_order
test_it_works
test_cancel_order_calls_refund_service_once  # Call count included in name
```

-- When the failure report shows `test_1 FAILED`, the informational value about which business scenario broke is zero.

## 2. Verify Behavior Through Public Interfaces

Instead of calling internal methods directly or counting calls, verify observable results through the public API.

Test **"given this input to this function, what output comes out"** or **"what observable side effect occurs."**

### CONSTRAINT

- Testing internal implementation: method extraction, algorithm replacement, or caching -- behavior stays the same, but every refactor breaks the test
- Verifying call counts: when batch processing is introduced for performance, the result is the same but the test turns red

### Good

```python
# Result value verification -- behavior test
def test_apply_discount_reduces_total():
    cart = Cart(items=[Item(price=10000), Item(price=20000)])
    cart.apply_discount(percent=10)
    assert cart.total == 27000

# Observable side-effect verification
def test_place_order_sends_confirmation_email():
    order_service.place_order(user_id=1, items=[item_a])
    assert email_outbox.last_recipient == "user@example.com"
```

### Bad

```python
# Verify existence and count of internal method calls
def test_apply_discount():
    cart = Cart(items=[Item(price=10000)])
    cart.apply_discount(percent=10)
    assert cart._recalculate_subtotal.call_count == 1  # internal call count
    assert cart._apply_tax.called                       # internal call existence
```

-- Even if `_recalculate_subtotal` and `_apply_tax` are merged into one, the `cart.total` result is identical -- yet the test breaks.

## 3. Mock Only External Boundaries

Mock only out-of-process dependencies: DB, external APIs, filesystem, system clock, etc.

Use real implementations for classes and functions within the same project.
If a single test exceeds 3 Mocks, review the design of the test target first.

### CONSTRAINT

- Mocking internal code: Mock behavior drifts from the real implementation, producing "test is green but production explodes"
- 4+ Mocks: the test precisely mirrors the production code structure -- refactoring requires full test rewrites

### Good

```python
# Mock only the external payment API; internal validation/calculation uses real code
def test_order_confirmed_when_payment_succeeds():
    fake_gateway = FakePaymentGateway(always_approve=True)
    service = OrderService(payment_gateway=fake_gateway)

    order = service.create_order(user_id=1, items=[item_a, item_b])

    assert order.status == "CONFIRMED"
    assert order.total == item_a.price + item_b.price  # internal calculation actually runs
```

### Bad

```python
# Mock every internal collaborator
def test_order_creation():
    mock_validator = Mock()
    mock_price_calc = Mock(return_value=30000)
    mock_formatter = Mock()
    mock_logger = Mock()
    mock_repo = Mock()
    mock_payment = Mock()

    service = OrderService(
        mock_validator, mock_price_calc, mock_formatter,
        mock_logger, mock_repo, mock_payment
    )
```

-- Six Mocks. The test runs in a fake world disconnected from real behavior. Refactoring requires a full test rewrite.

## 4. Do Not Share State Between Tests

Every test runs independently. It must not depend on execution order or share global state.

Each test sets up the state it needs (setup) and cleans up after (teardown).

### CONSTRAINT

- Sharing global state across tests: failure in test A cascades into failures in test B -- root cause becomes unidentifiable
- Depending on execution order: single-test execution is impossible, parallel execution is impossible, and flaky tests appear in CI

### Good

```python
# Each test creates its own independent state
def test_add_item_increases_count():
    cart = Cart()
    cart.add(Item(name="A", price=1000))
    assert cart.item_count == 1

def test_remove_item_decreases_count():
    cart = Cart()
    cart.add(Item(name="A", price=1000))
    cart.remove("A")
    assert cart.item_count == 0
```

### Bad

```python
# State shared across tests
shared_cart = Cart()

def test_step1_add():
    shared_cart.add(Item(name="A", price=1000))
    assert shared_cart.item_count == 1

def test_step2_add_more():  # depends on step1's result
    shared_cart.add(Item(name="B", price=2000))
    assert shared_cart.item_count == 2  # breaks if step1 fails
```

-- If test_step1 fails, test_step2 fails too. Running it alone yields a different result.

## 5. Separate Unit Tests from Integration Tests

Unit tests verify business logic quickly at millisecond speed; integration tests verify interactions with external systems.

Separate the two into distinct execution paths -- do not mix them.

| Criterion | Unit Test | Integration Test |
|-----------|-----------|------------------|
| Target | Domain rules, pure functions, business flow | DB, external APIs, message queue integration |
| Execution speed | Milliseconds | Seconds |
| External dependencies | Mock or in-memory | Real instance (test DB, etc.) |
| Execution timing | Every commit | PR or nightly build |
| Meaning of failure | Business logic error | Integration config, schema mismatch |

### CONSTRAINT

- Mixing external dependencies into unit tests: running tests requires a DB, slowing the feedback loop; impossible to run in environments without the DB
- Only unit tests without integration tests: misses the "individual parts work but do not assemble" problem

### Good

```python
# Unit test -- pure logic, no external dependencies
def test_free_shipping_over_50000():
    assert calculate_shipping(subtotal=60000) == 0

def test_standard_shipping_under_50000():
    assert calculate_shipping(subtotal=30000) == 3000

# Integration test -- verifies real DB interaction
def test_order_persists_and_retrieves():
    order = Order(user_id=1, items=[item_a])
    repo.save(order)
    loaded = repo.find_by_id(order.id)
    assert loaded.user_id == 1
    assert loaded.items == [item_a]
```

### Bad

```python
# Unit test that depends on a real DB
def test_calculate_discount():
    db.insert(User(id=1, tier="VIP"))   # DB required!
    user = db.find(1)                    # DB required!
    discount = calculate_discount(user.tier)
    assert discount == 0.2
```

-- Verifying pure discount calculation requires a DB. Impossible to run without the DB, and execution is tens to hundreds of times slower.

## 6. Priority for Fixing Test Failures

When a test fails, **inspect the test code first**. Modify production code only after an actual logic error is confirmed.

### Order

1. **Inspect test code** -- check whether the test itself has wrong expected values, wrong setup, or wrong assumptions
2. **Inspect test environment** -- check environment issues such as missing dependencies, timing issues, or ordering dependencies
3. **Modify production code** -- only if the above two steps show no issues and a real logic error is confirmed

### CONSTRAINT

- Modifying production code immediately on test failure: breaks correctly working code and introduces new bugs
- Ignoring a real bug and only making the test pass: production defects get hidden

### Good

```
Test failure -> verify the expected value reflects the latest spec
  -> Expected value is wrong -> fix the test
  -> Expected value is correct -> bug in production code -> fix production code
```

### Bad

```
Test failure -> (without verification) modify production code
  -> Other tests break -> cascade fixes -> affects previously working features
```

## Exceptions

- Auto-generated code tests: the scope guaranteed by the generation tool is exempt from tests
- E2E tests: use scenario-based tests instead of isolation/mock policy -- exempt from sections 3 and 4
- Performance/load tests: exempt from behavior-test rules. Write around metrics such as throughput and response time
- Legacy code tests: implementation-detail tests are temporarily allowed as "change-detection tests" until refactoring completes -- convert to behavior tests after refactoring

## DEPENDS

- `architecture.md` -- Layer separation is a prerequisite for setting up test isolation and Mock boundaries
