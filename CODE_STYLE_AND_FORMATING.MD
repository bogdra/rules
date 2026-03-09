# ========================================
# CODE STYLE & FORMATTING
# ========================================

---
description: Enforce correct Cursor rule format when creating or editing rules (per cursor.com/docs/rules)
globs: ".cursor/rules/**"
alwaysApply: false
---

# Cursor Rule Format

When asked to create a new rule or when editing files in `.cursor/rules/`, follow the [Cursor Rules documentation](https://cursor.com/docs/rules).

## File location and extension

- **Path:** `.cursor/rules/` (optionally in subfolders, e.g. `frontend/components.md`).
- **Extension:** `.md` (plain markdown) or `.mdc` (markdown with frontmatter for `description` and `globs`). Use `.mdc` when you need path-based or always-apply control.

## Frontmatter (for .mdc rules)

YAML between `---` at the top of the file:

```yaml
---
description: "Brief description (shown in rule picker; used for Apply Intelligently)"
globs: "**/*.ts"   # optional; for Apply to Specific Files
alwaysApply: false # true = Apply to every chat session
---
```

| Rule type (dropdown)     | Frontmatter |
|--------------------------|-------------|
| Always Apply             | `alwaysApply: true` |
| Apply Intelligently      | `alwaysApply: false`, set `description` |
| Apply to Specific Files  | `alwaysApply: false`, set `globs` (e.g. `**/*.ts`, `**/*.md`) |
| Apply Manually           | `alwaysApply: false`; rule is applied when @-mentioned in chat |

If `alwaysApply` is false and no `globs` is set, the rule is applied when the Agent decides it is relevant (using `description`) or when invoked manually.

## Body (content)

- **Title:** One H1 at the top (e.g. `# Rule Title`).
- **Length:** Keep under 500 lines; split into multiple rules if larger.
- **Focus:** One main concern per rule; composable rules over one huge rule.
- **Actionable:** Write like clear internal docs. Avoid vague guidance.
- **Examples:** Provide concrete examples or reference files with `@filename.ext` instead of pasting large blocks.
- **References:** Prefer pointing to canonical examples in the codebase over copying code into the rule.

## Best practices (from docs)

- Encode domain-specific knowledge or project-specific workflows.
- Standardize style or architecture decisions.
- Include concrete examples or referenced files.
- Reuse rules when repeating prompts in chat.

## Avoid in rules

- Copying entire style guides (use a linter instead).
- Documenting every possible command (Agent knows common tools).
- Instructions for rare edge cases; keep rules focused on frequent patterns.
- Duplicating what is already in the codebase; point to canonical examples.

## Checklist when creating a new rule

- [ ] File is in `.cursor/rules/` with extension `.md` or `.mdc`.
- [ ] If `.mdc`: frontmatter has `description` and `alwaysApply`; add `globs` for file-scoped application.
- [ ] Body has one H1 title and is focused and actionable (under 500 lines).
- [ ] Concrete examples or `@file` references included where they help.
- [ ] Rule is checked into git so the team can use it.

---
description: Mermaid diagram syntax to avoid UnknownDiagramError and parser issues
alwaysApply: false
---

# Mermaid diagrams in Markdown

## Diagram type declaration

- **Use `graph` for flowcharts**, not `flowchart`. Some renderers (e.g. Mermaid in Cursor/IDE) can misparse `flowchart` and report "UnknownDiagramError: No diagram type detected... for text: lowchart TB" because the leading `f` is dropped. The keyword `graph` is the valid Mermaid alias and avoids this.
- **First line inside the code block must be the diagram declaration only**: no leading space, no BOM, no invisible characters. Start the line with the diagram type.
- **Valid diagram types**: `graph`, `flowchart` (prefer `graph`), `sequenceDiagram`, `classDiagram`, `stateDiagram`, `erDiagram`, `journey`, `gantt`, `pie`, etc. Spell them exactly.
- **Use `graph` for flowcharts**, not `flowchart`. Some renderers drop the leading "f" and report "UnknownDiagramError... for text: lowchart TB". The keyword `graph` is the valid Mermaid alias.
- **First line inside the code block**: diagram declaration only. No leading newline, space, or BOM.
- **Valid types**: `graph`, `sequenceDiagram`, `classDiagram`, `stateDiagram`, `erDiagram`, `journey`, `gantt`, `pie`. For flowcharts use `graph TB` or `graph LR`.

## Correct pattern

```mermaid
graph TB
    A --> B
```

First line after the mermaid fence: `graph TB` then your nodes/edges.

## Avoid

```markdown
```mermaid
flowchart TB
```
```
(Can trigger UnknownDiagramError in some viewers.)

```markdown
```mermaid

flowchart TB
```
```
(Leading newline or space can break detection.)
```

## Summary

- Prefer **`graph TB`** (or `graph LR`, etc.) instead of `flowchart TB` for flowcharts.
- Keep the first line of the diagram as a single declaration: `graph TB`, `sequenceDiagram`, etc., with no leading whitespace.

---
description: "PHP coding style to avoid PHP_CodeSniffer violations (PSR-12)"
alwaysApply: false
---

- Newline after opening brace `{`; closing brace `}` on its own line.
- No blank line immediately after opening brace of control structure.
- At least one space before and after `.` (concatenation) and `??` (null coalescing).
- `case` keyword must be followed by a single space.
- Multi-line function calls: continuation lines must align with opening parenthesis or first argument; use 4 spaces per indent level.
- Header blocks: single blank line between `use` statements or docblocks, not multiple.

---
description: "Use class constants instead of magic strings for domain values"
alwaysApply: true
---

When working with delivery methods, delivery modes, or any domain-specific string values, use the defined constants from the appropriate transformer or service class instead of hardcoded strings.

**Example:**
```php
// Bad — uses magic strings
$subcartDeliveryMethods[$vendorKey] = 'electronic';
$subcartDeliveryMethods['pickup'] = $method;
$subcartDeliveryMethods[$vendorKey] = 'courier';

// Good — uses constants
$subcartDeliveryMethods[$vendorKey] = DeliveryMethodTransformer::DELIVERY_MODE_ELECTRONIC;
$subcartDeliveryMethods[DeliveryMethodTransformer::DELIVERY_MODE_PICKUP] = $method;
$subcartDeliveryMethods[$vendorKey] = DeliveryMethodTransformer::DELIVERY_MODE_HOME;
```

---
description: "Omit exception variable in catch when unused (PHP 8.0+ non-capturing catches)"
alwaysApply: true
---

When the exception object is not used, omit the variable name (PHP 8.0+).

**Example:**
```php
// Good
catch (Throwable) {
    Log::error("Something failed");
}

// Bad
catch (Throwable $e) {
    Log::error("Something failed");
}
```

---
description: "Use match expressions for complex conditionals"
alwaysApply: true
---

Replace complex if-elseif chains with match expressions for better readability.

**Example:**
```php
// Good: Using match
$status = match($orderState) {
    'pending' => OrderStatus::PENDING,
    'processing' => OrderStatus::PROCESSING,
    'shipped' => OrderStatus::SHIPPED,
    'delivered' => OrderStatus::DELIVERED,
    default => throw new InvalidArgumentException("Unknown state: $orderState")
};

// Avoid: Long if-elseif chains
if ($orderState === 'pending') {
    $status = OrderStatus::PENDING;
} elseif ($orderState === 'processing') {
    $status = OrderStatus::PROCESSING;
} // ... etc
```

---
description: "Explicit type declarations for all methods and constants"
alwaysApply: true
---

Always declare return types for methods and data types for constants.

**Example:**
```php
// Good: Explicit types
private const string TARGETED_VCR_TEST_VARIANT = 'B';
private const int MAX_RETRY_ATTEMPTS = 3;
private const array ALLOWED_STATUSES = ['active', 'pending'];

public function calculateTotal(array $items): float {
    return array_sum(array_column($items, 'price'));
}

public function findUser(int $id): ?User {
    return $this->repository->find($id);
}
```

---
description: "PHP 8.3+ features and modern syntax"
alwaysApply: true
---

Use modern PHP features including typed properties, union types, and readonly classes.

**Example:**
```php
// Good: Modern PHP syntax
readonly class UserRegistrationRequest {
    public function __construct(
        public string $email,
        public string $password,
        public ?string $firstName = null,
        public DateTimeInterface $createdAt = new DateTime(),
    ) {}
}

// Union types
public function process(string|int $identifier): User|null {
    return is_string($identifier)
        ? $this->findByEmail($identifier)
        : $this->findById($identifier);
}
```

---
description: "Naming conventions - PascalCase classes, camelCase methods, snake_case DB columns"
alwaysApply: true
---

Consistent naming across all code elements.

**Example:**
```php
// Good: Consistent naming
class UserRegistrationService {           // PascalCase for classes
    public function registerNewUser(): void // camelCase for methods
    {
        // Database columns use snake_case: user_email, created_at
    }

    private function validateEmailAddress(): bool // camelCase for methods
    {
        return true;
    }
}
```

---
description: "Trailing commas in method parameters and array elements"
alwaysApply: true
---

Use trailing commas for better version control diffs.

**Example:**
```php
// Good: Trailing commas
public function createOrder(
    User $user,
    array $items,
    PaymentMethod $payment,    // <- trailing comma
): Order {
    return new Order([
        'user_id' => $user->getId(),
        'items' => $items,
        'payment_method' => $payment->getType(),    // <- trailing comma
    ]);
}
```

---
description: "Space after negation operators"
alwaysApply: true
---

Always add space between negation operator and operand for readability.

**Example:**
```php
// Good: Space after !
if (! $user->isActive()) {
    throw new InactiveUserException();
}

if (! isset($data['email'])) {
    $errors[] = 'Email is required';
}

// Bad: No space
if (!$user->isActive()) { }
if (!isset($data['email'])) { }
```

---
description: "Null Coalescing for Defaults"
alwaysApply: true
---

Prefer `??` over `isset()` ternaries. Use type-appropriate defaults:
- `?? null` for optional objects
- `?? ''` for strings
- `?? 0` for integers
- `?? []` for arrays

**Example:**
```php
// Good
$method = $deliveryMethods[$vendorId] ?? null;
$name = $productsInfo['vendors'][$id] ?? '';

// Bad
$method = isset($deliveryMethods[$vendorId]) ? $deliveryMethods[$vendorId] : null;
```

---
description: "Naming: camelCase Variables, snake_case Array Keys"
alwaysApply: true
---

Variables use camelCase. Array keys use snake_case:
**Example:**
```php
// Good
$deliveryVendorId = 123;
$line['vendor_name'] = 'Acme';

// Bad
$delivery_vendor_id = 123;
$line['vendorName'] = 'Acme';
```

---
description: "Non-Capturing Catches"
alwaysApply: true
---

When the exception object is not used, omit the variable name (PHP 8.0+):
**Example:**
```php
// Good
catch (Throwable) {
    Log::error("Something failed");
}

// Bad
catch (Throwable $e) {
    Log::error("Something failed");
}
```

---
description: "Guard Clauses with `continue`"
alwaysApply: true
---

In loops, handle the negative case first with early `continue`. Keep the happy path at lower nesting. Avoid deep `if/else` blocks inside loops:
**Example:**
```php
// Good
foreach ($items as $key => $item) {
    if (! $item->isValid()) {
        continue;
    }

    $this->process($item);
}

// Bad
foreach ($items as $key => $item) {
    if ($item->isValid()) {
        $this->process($item);
    }
}
```

## Prefer array_map / array_filter over foreach

Use `array_map` and `array_filter` for transformations and filtering. Reserve `foreach` for side effects or complex multi-output logic.

```php
// Good: array_map for transformation
$orderTotals = array_map(
    fn(Order $order) => $order->getTotal(),
    $orders,
);

// Bad: foreach just to build a transformed array
$orderTotals = [];
foreach ($orders as $order) {
    $orderTotals[] = $order->getTotal();
}
```

```php
// Good: array_filter for selection
$activeUsers = array_filter(
    $users,
    fn(User $user) => $user->isActive(),
);

// Bad: foreach just to filter
$activeUsers = [];
foreach ($users as $user) {
    if ($user->isActive()) {
        $activeUsers[] = $user;
    }
}
```

`foreach` is still appropriate for side effects (persisting, logging) or when the loop body produces multiple outputs or requires early exit.
