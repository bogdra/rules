# ========================================
# ARCHITECTURE & DESIGN PATTERNS
# ========================================

---
description: "Single Responsibility Principle - One class, one purpose"
alwaysApply: true
---

Each class should have only one reason to change.

**Example:**
```php
// Bad: Multiple responsibilities
class UserManager {
    public function createUser($data) { /* user creation */ }
    public function sendWelcomeEmail($user) { /* email logic */ }
    public function logActivity($action) { /* logging logic */ }
}

// Good: Separated responsibilities
class UserCreator {
    public function create(UserData $data): User { /* user creation only */ }
}

class EmailService {
    public function sendWelcome(User $user): void { /* email logic only */ }
}
```

---
description: "Dependency Inversion - Depend on abstractions, not implementations"
alwaysApply: true
---

High-level modules should not depend on low-level modules. Both should depend on abstractions.

**Example:**
```php
// Bad: Direct dependency on implementation
class OrderService {
    public function process(): void {
        $repo = new MySqlUserRepository();  // Tight coupling
        $user = $repo->find(1);
    }
}

// Good: Depend on abstraction
class OrderService {
    public function __construct(
        private UserRepositoryInterface $userRepository,
    ) {}

    public function process(): void {
        $user = $this->userRepository->find(1);
    }
}
```

---
description: "Interface Segregation - Small, specific interfaces"
alwaysApply: true
---

Create smaller, focused interfaces rather than large monolithic ones.

**Example:**
```php
// Bad: Fat interface
interface DeviceInterface {
    public function turnOn(): void;
    public function turnOff(): void;
    public function setTimer(int $minutes): void;
    public function playMusic(): void;
}

// Good: Segregated interfaces
interface Switchable {
    public function turnOn(): void;
    public function turnOff(): void;
}

interface Timer {
    public function setTimer(int $minutes): void;
}

interface MediaPlayer {
    public function playMusic(): void;
}
```

---
description: "Open/Closed Principle - Open for extension, closed for modification"
alwaysApply: true
---

Add new functionality through extension, not by modifying existing code.

**Example:**
```php
// Good: Extensible payment system
abstract class Payment {
    protected string $title;
    protected int $balance = 1000;

    public function __construct(string $title, string $accountNumber) {
        $this->title = $title;
        $this->accountNumber = $accountNumber;
    }

    abstract public function pay(int $amount): void;
}

class CreditCard extends Payment {
    private int $charges = 50;

    public function pay(int $amount): void {
        $this->balance -= ($amount + $this->charges);
        // Implementation specific to credit cards
    }
}
```

---
description: "Value Objects for data validation and business rules"
alwaysApply: true
---

Encapsulate data with validation rules in value objects instead of passing primitives.

**Example:**
```php
// Bad: Raw primitive validation everywhere
function sendEmail(string $email) {
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        throw new InvalidArgumentException('Invalid email');
    }
    // send email logic
}

// Good: Encapsulated validation
final readonly class Email {
    public function __construct(public string $value) {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }
    }
}

function sendEmail(Email $email): void {
    // email is guaranteed to be valid
}
```

---
description: "Composition over inheritance for complex behavior"
alwaysApply: true
---

Build complex functionality by combining simple objects rather than deep inheritance chains.

**Example:**
```php
// Good: Composition approach
class OrderProcessor {
    public function __construct(
        private PriceCalculator $calculator,
        private InventoryChecker $inventory,
        private NotificationSender $notifier,
    ) {}

    public function process(Order $order): void {
        $price = $this->calculator->calculate($order);
        $this->inventory->reserve($order->getItems());
        $this->notifier->sendConfirmation($order);
    }
}

// Avoid: Deep inheritance
class OrderProcessor extends BaseProcessor extends AbstractHandler { }
```

---
description: "Isolate external dependencies behind interfaces"
alwaysApply: true
---

Wrap third-party libraries behind your own interfaces to protect against changes.

**Example:**
```php
interface PaymentGatewayInterface {
    public function charge(Money $amount, string $token): PaymentResult;
}

class StripePaymentGateway implements PaymentGatewayInterface {
    public function charge(Money $amount, string $token): PaymentResult {
        // Stripe-specific implementation hidden behind interface
        return $this->stripeClient->createCharge([
            'amount' => $amount->getCents(),
            'source' => $token,
        ]);
    }
}
```

---
description: "Bounded contexts with explicit interfaces"
alwaysApply: true
---

Group related functionality together with clear boundaries. Cross-boundary communication must use explicit interfaces.

**Example:**
```php
// Order Context
namespace Order;
interface OrderRepositoryInterface {
    public function save(Order $order): void;
}

// User Context
namespace User;
interface UserServiceInterface {
    public function findById(UserId $id): User;
}

// Cross-context communication via DTOs
class OrderService {
    public function __construct(
        private OrderRepositoryInterface $orders,
        private UserServiceInterface $users, // Interface, not direct dependency
    ) {}
}
```
