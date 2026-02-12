# ========================================
# CODE STYLE & FORMATTING
# ========================================

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
