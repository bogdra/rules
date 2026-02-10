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
