# Architecture Principles -- Detailed Spec

> WHY: By default, Claude ignores layer boundaries and generates code along the shortest path.
> It executes SQL directly from controllers, or has domain objects call HTTP clients.
> Without this rule, layer violations accumulate until unit test isolation becomes impossible,
> circular references emerge, and replacing infrastructure requires a full rewrite.

## SOURCE

| Evidence ID | Basis | Type | Source |
|-------------|-------|------|--------|
| E-IS-001 | Layer separation and unidirectional dependency flow | INDUSTRY_STD | Robert C. Martin, Clean Architecture (2017) |
| E-IS-002 | Dependency Inversion Principle -- higher modules must not depend on lower implementations | INDUSTRY_STD | SOLID Principles -- Robert C. Martin |
| E-IS-003 | Acyclic Dependencies Principle -- circular references between modules are forbidden | INDUSTRY_STD | Acyclic Dependencies Principle (ADP) |

## Scope

- paths: all source code (`src/**`, `lib/**`, `app/**`, etc.)
- target: any code work involving new module creation, import additions, or changes to inter-layer call paths

## 1. Layer Responsibility Isolation

The presentation, business, and infrastructure layers must each perform only their own responsibilities and must not encroach on other layers.

Replacing or refactoring one layer must not propagate changes to other layers.
Layer boundaries are physically separated via directories or packages.

| Layer | Does | Does Not Do |
|-------|------|-------------|
| Presentation | Request parsing, response formatting, routing | SQL execution, domain rule decisions |
| Business (Domain/Application) | Domain rules, use-case flow control, validation | DB access, HTTP communication, file I/O |
| Infrastructure | DB queries, external API communication, message queues, filesystem | UI rendering, business policy decisions |

### CONSTRAINT

- If the presentation layer calls the DB directly: handler tests require a running DB -- the millisecond feedback loop degrades to seconds
- If SQL lives inside the business layer: all domain logic must be modified when swapping ORMs or running DB migrations

### Good

```python
# Presentation layer -- receives request, delegates to use case, formats result
def handle_create_order(request):
    result = create_order_usecase.execute(request.user_id, request.items)
    return json_response(result)

# Business layer -- applies domain rules only; accesses the repository through an interface
def execute(user_id, items):
    user = self.user_repo.get(user_id)
    if user.is_suspended():
        raise DomainError("Suspended user")
    order = Order.create(user, items)
    return self.order_repo.save(order)

# Infrastructure layer -- handles DB queries only
def save(order):
    return self.db.insert("orders", order.to_dict())
```

### Bad

```python
# Presentation layer handles DB queries and business rules all at once
def handle_create_order(request):
    user = db.query("SELECT * FROM users WHERE id = ?", [request.user_id])
    if user["status"] != "active":
        return {"error": "Suspended user"}
    db.execute("INSERT INTO orders (user_id, items) VALUES (?, ?)",
               [request.user_id, json.dumps(request.items)])
    return {"status": "created"}
```

-- Routing, SQL, and domain rules are mixed in one function. None of them can be independently tested or replaced.

## 2. Unidirectional Dependency Flow

Dependencies always flow from the outside (infrastructure) to the inside (domain). Reverse dependencies, where the domain layer references infrastructure, are prohibited.

External capabilities needed by the domain layer are declared as interfaces (abstractions), and the infrastructure layer implements them.
This is the Dependency Inversion Principle (DIP).

```
[Presentation] ──→ [Business/Domain] ←── [Infrastructure]
                         ↑
                   Interface definitions
```

### CONSTRAINT

- If the domain imports a specific DB driver directly: replacing that DB propagates changes throughout all domain logic
- If concrete classes are referenced directly: tests cannot swap in lightweight in-memory implementations -- they always depend on real infrastructure, producing slow tests

### Good

```python
# Domain layer declares the interface only
class OrderRepository:
    def save(self, order: Order) -> Order: ...
    def find_by_id(self, order_id: str) -> Order: ...

# Infrastructure layer provides the implementation
class PostgresOrderRepository(OrderRepository):
    def __init__(self, connection_pool):
        self._pool = connection_pool

    def save(self, order: Order) -> Order:
        self._pool.execute("INSERT INTO orders ...", order.to_row())
        return order
```

### Bad

```python
# Domain layer directly imports a specific infrastructure library
import psycopg2

class OrderService:
    def __init__(self):
        self.conn = psycopg2.connect("host=db.prod dbname=orders")

    def create(self, user_id, items):
        cursor = self.conn.cursor()
        cursor.execute("INSERT INTO orders ...")
```

-- OrderService is directly coupled to psycopg2 and the production DB address. Switching to MySQL or using SQLite in tests requires rewriting OrderService itself.

## 3. Inter-Module Communication Through Public Interfaces

Inter-module calls use only the public entry points each module exposes. Internal implementations (private functions, internal packages, private data structures) must not be referenced from outside.

Only what a module exports via `index` or `__init__` is its public API.

### CONSTRAINT

- Referencing internal implementations directly: calling code breaks simultaneously when that module is refactored internally
- Coupling modules without a public contract: static analysis cannot trace the impact of changes, so a single edit produces unpredictable ripple effects

### Good

```python
# Use the payment module's public interface
from payment import PaymentProcessor

processor = PaymentProcessor()
receipt = processor.charge(order)
```

### Bad

```python
# Reference the payment module's private internals directly
from payment._adapters.stripe_raw import StripeRawClient
from payment._utils.currency import convert_to_cents

client = StripeRawClient()
amount = convert_to_cents(order.total)
client.create_charge(amount)
```

-- `_adapters` and `_utils` are private internals. If the payment module switches from Stripe to another provider, all calling code breaks.

## 4. Preventing Circular References

Circular loops, where module A references module B while module B also references module A, are forbidden.

When a cycle is detected, break the reverse dependency by extracting shared interfaces into a separate module or by using event/callback patterns.

### CONSTRAINT

- When a cycle occurs: some languages fail the build; others experience tangled module initialization order that triggers runtime errors
- As cycle depth increases: a single module change propagates along the entire cycle path, expanding the impact radius exponentially

### Good

```python
# Extract a shared events module to break the cycle
# shared/events.py
class OrderCompletedEvent:
    order_id: str
    total: int

# order/ -- depends only on shared
from shared.events import OrderCompletedEvent

def complete_order(order):
    publish(OrderCompletedEvent(order_id=order.id, total=order.total))

# notification/ -- depends only on shared (does not reference order directly)
from shared.events import OrderCompletedEvent

def on_order_completed(event: OrderCompletedEvent):
    send_email(event.order_id)
```

### Bad

```python
# order/service.py
from notification.sender import NotificationSender  # order -> notification

# notification/sender.py
from order.service import OrderService  # notification -> order (cycle!)
```

-- order and notification import each other directly. Modifying one breaks the other, and neither can be tested independently.

## 5. Project CLAUDE.md Takes Precedence

If the project's `CLAUDE.md` specifies an architecture, that definition takes precedence over this rule.

Apply sections 1-4 of this rule as default guidelines only when `CLAUDE.md` has no architecture section.

### CONSTRAINT

- Ignoring the CLAUDE.md architecture when generating code: conflicts with the team-agreed structure -- review rejection and rework
- Introducing an independent architecture without a CLAUDE.md definition: adds a structure not agreed on by the team, destroying consistency

### Good

```markdown
# When CLAUDE.md defines Hexagonal Architecture
# -> Follow the port/adapter structure from CLAUDE.md as-is

# When CLAUDE.md has no architecture definition
# -> Apply sections 1-4 of this rule as general principles
# -> Propose adding an architecture section to the team
```

### Bad

```markdown
# Even though CLAUDE.md specifies "Hexagonal Architecture"
# -> Ignore it and generate code with the MVC pattern
```

-- Ignoring the team-agreed architecture leads to review rejection and destroys overall structural consistency.

## Exceptions

- Prototype/PoC code: layer separation may be relaxed. Mark its temporariness with a `MIGRATION` prefix comment
- One-off scripts (`scripts/`, `tools/`): single-purpose utilities are exempt from architecture rules
- Configuration files (`*.config.*`, `*.json`, `*.yaml`): not subject to layer rules
- Test code: internal access to the test target is permitted when unavoidable. Prefer testing through public APIs

## DEPENDS

- `dari-standards:claude-md-design.md` -- Section 5 only works effectively when CLAUDE.md defines an architecture
- `testing.md` -- Layer separation is a prerequisite for setting up test isolation and Mock boundaries
