# ========================================
# TESTING & QUALITY
# ========================================

---
description: "Design for testability - avoid tight coupling"
alwaysApply: true
---

If a class is hard to unit test in isolation, it likely has too many dependencies or responsibilities.

**Example:**
```php
// Bad: Hard to test due to tight coupling
class OrderProcessor {
    public function process(int $orderId): void {
        $user = (new UserRepository())->find($orderId);           // Hard-coded dependency
        $email = new EmailService();                              // Hard-coded dependency
        $payment = PaymentGateway::charge($user->getCard());      // Static call

        if ($payment->isSuccessful()) {
            $email->send($user->getEmail(), 'Order confirmed');
            Logger::log('Order processed');                      // Static call
        }
    }
}

// Good: Easy to test with injected dependencies
class OrderProcessor {
    public function __construct(
        private UserRepositoryInterface $userRepository,
        private EmailServiceInterface $emailService,
        private PaymentGatewayInterface $paymentGateway,
        private LoggerInterface $logger,
    ) {}

    public function process(int $orderId): void {
        $user = $this->userRepository->find($orderId);
        $payment = $this->paymentGateway->charge($user->getCard());

        if ($payment->isSuccessful()) {
            $this->emailService->send($user->getEmail(), 'Order confirmed');
            $this->logger->log('Order processed');
        }
    }
}

// Easy to test with mocks
public function testOrderProcessing(): void {
    $userRepo = $this->createMock(UserRepositoryInterface::class);
    $emailService = $this->createMock(EmailServiceInterface::class);
    // ... etc

    $processor = new OrderProcessor($userRepo, $emailService, $paymentGateway, $logger);
    $processor->process(123);

    // Verify interactions
}
```

---
description: "Exception handling with appropriate responses"
alwaysApply: true
---

Handle exceptions gracefully and return meaningful responses to users.

**Example:**
```php
// Good: Comprehensive exception handling
class UserService {
    public function createUser(array $userData): User {
        try {
            $email = new Email($userData['email'] ?? '');

            if ($this->userRepository->existsByEmail($email)) {
                throw new UserAlreadyExistsException("User with email {$email->value} already exists");
            }

            return $this->userRepository->save(new User($email, $userData['name']));

        } catch (InvalidArgumentException $e) {
            throw new ValidationException("Invalid user data: " . $e->getMessage());
        } catch (DatabaseException $e) {
            $this->logger->error('Database error during user creation', ['error' => $e->getMessage()]);
            throw new ServiceUnavailableException('Unable to create user at this time');
        }
    }
}

// Controller handles service exceptions appropriately
public function createUser(Request $request): Response {
    try {
        $user = $this->userService->createUser($request->getParsedBody());
        return new Response(['user_id' => $user->getId()], 201);

    } catch (ValidationException $e) {
        return new Response(['error' => $e->getMessage()], 400);
    } catch (UserAlreadyExistsException $e) {
        return new Response(['error' => $e->getMessage()], 409);
    } catch (ServiceUnavailableException $e) {
        return new Response(['error' => $e->getMessage()], 503);
    }
}
```
