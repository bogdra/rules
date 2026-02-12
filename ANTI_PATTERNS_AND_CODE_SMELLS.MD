# ========================================
# ANTI-PATTERNS & CODE SMELLS
# ========================================

---
description: "Avoid feature envy - move operations close to data"
alwaysApply: true
---

When a method extensively uses another object's data, move the operations to that object's class.

**Example:**
```php
// Bad: Feature envy - OrderService knows too much about User internals
class OrderService {
    public function calculateDiscount(User $user, Order $order): float {
        if ($user->getRegistrationDate() < new DateTime('-1 year')
            && $user->getTotalOrders() > 10
            && $user->getStatus() === 'premium') {
            return $order->getTotal() * 0.15;
        }
        return 0;
    }
}

// Good: Move logic to User class
class User {
    public function isEligibleForLoyaltyDiscount(): bool {
        return $this->registrationDate < new DateTime('-1 year')
            && $this->totalOrders > 10
            && $this->status === 'premium';
    }

    public function getLoyaltyDiscountRate(): float {
        return $this->isEligibleForLoyaltyDiscount() ? 0.15 : 0;
    }
}

class OrderService {
    public function calculateDiscount(User $user, Order $order): float {
        return $order->getTotal() * $user->getLoyaltyDiscountRate();
    }
}
```

---
description: "Replace static calls with dependency injection"
alwaysApply: true
---

Static calls create hidden dependencies that are hard to test. Use injected services instead.

**Example:**
```php
// Bad: Static dependencies
class OrderProcessor {
    public function process(): void {
        $user = UserHelper::getCurrentUser();        // Hidden dependency
        Logger::log('Processing order');             // Hidden dependency
        EmailSender::send($user->email, 'Order confirmation');
    }
}

// Good: Explicit dependencies
class OrderProcessor {
    public function __construct(
        private UserProviderInterface $userProvider,
        private LoggerInterface $logger,
        private EmailServiceInterface $emailService,
    ) {}

    public function process(): void {
        $user = $this->userProvider->getCurrentUser();
        $this->logger->log('Processing order');
        $this->emailService->send($user->email, 'Order confirmation');
    }
}
```

---
description: "Avoid business logic in controllers - use service layer"
alwaysApply: true
---

Controllers should only handle HTTP concerns, not business logic.

**Example:**
```php
// Bad: Business logic in controller
class UserController {
    public function register(Request $request): Response {
        $email = $request->get('email');
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {  // Validation logic
            return new Response('Invalid email', 400);
        }

        if (strlen($request->get('password')) < 8) {        // Business logic
            return new Response('Password too short', 400);
        }

        // Database operations, email sending, etc...
    }
}

// Good: Delegate to service layer
class UserController {
    public function __construct(
        private UserRegistrationService $registrationService,
    ) {}

    public function register(Request $request): Response {
        try {
            $user = $this->registrationService->register(
                $request->get('email'),
                $request->get('password')
            );

            return new Response(['user_id' => $user->getId()], 201);
        } catch (ValidationException $e) {
            return new Response(['error' => $e->getMessage()], 400);
        }
    }
}
```

---
description: "Use QueryBuilder instead of raw SQL when possible"
alwaysApply: true
---

Prefer Doctrine QueryBuilder for type safety and maintainability over raw SQL.

**Example:**
```php
// Bad: Raw SQL
public function findActiveUsersByRole(string $role): array {
    $sql = "SELECT * FROM users WHERE role = ? AND status = 'active' ORDER BY created_at DESC";
    return $this->connection->fetchAllAssociative($sql, [$role]);
}

// Good: QueryBuilder
public function findActiveUsersByRole(string $role): array {
    return $this->createQueryBuilder('u')
        ->where('u.role = :role')
        ->andWhere('u.status = :status')
        ->orderBy('u.createdAt', 'DESC')
        ->setParameter('role', $role)
        ->setParameter('status', 'active')
        ->getQuery()
        ->getResult();
}
```

---
description: "Avoid redundant documentation - don't document the obvious"
alwaysApply: true
---

Don't add docblock information that's already clear from type declarations.

**Example:**
```php
// Bad: Redundant documentation
/**
 * Gets the user ID
 * @return int The user ID
 */
public function getUserId(): int {
    return $this->userId;
}

// Good: Only document non-obvious behavior
/**
 * Calculates shipping cost based on weight, distance and delivery speed.
 * Uses external shipping API when weight exceeds 50kg.
 */
public function calculateShippingCost(Package $package, Address $destination): Money {
    // Implementation
}

// Good: Simple method needs no documentation
public function getUserId(): int {
    return $this->userId;
}
```

## Guard Clauses with `continue`

description: "In loops, handle the negative case first with early `continue`. Keep the happy path at lower nesting. Avoid deep `if/else` blocks inside loops:"
alwaysApply: true

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

