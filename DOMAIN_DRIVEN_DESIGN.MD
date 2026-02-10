
# ========================================
# STRATEGIC DESIGN - BOUNDED CONTEXTS
# ========================================

---
description: "Define clear bounded contexts with explicit boundaries"
alwaysApply: true
---

Each bounded context should have a clear responsibility and its own ubiquitous language. Avoid sharing domain models between contexts.

**Example:**
```php
// Good: Separate contexts with clear boundaries
namespace Sales\Domain;
class Customer {
    private CustomerId $id;
    private string $name;
    private CreditLimit $creditLimit;
    
    public function placePurchaseOrder(Money $amount): PurchaseOrder {
        if ($this->creditLimit->isExceeded($amount)) {
            throw new CreditLimitExceededException();
        }
        return new PurchaseOrder($this->id, $amount);
    }
}

namespace Shipping\Domain;  
class Customer {
    private CustomerId $id;
    private DeliveryAddress $address;
    private ShippingPreferences $preferences;
    
    public function scheduleDelivery(Package $package): Delivery {
        return new Delivery($this->address, $this->preferences->getTimeWindow());
    }
}

// Bad: Shared domain model across contexts
namespace Shared\Domain;
class Customer {
    // Mixed concerns from different contexts
    private $creditLimit;     // Sales concern
    private $address;         // Shipping concern  
    private $preferences;     // Multiple contexts
}
```

---
description: "Use Anti-Corruption Layer for external system integration"
alwaysApply: true
---

Protect your domain model from external systems by translating their concepts through an Anti-Corruption Layer.

**Example:**
```php
// Good: Anti-Corruption Layer protects domain
interface PaymentServiceInterface {
    public function processPayment(Payment $domainPayment): PaymentResult;
}

class StripePaymentAdapter implements PaymentServiceInterface {
    public function __construct(private StripeClient $stripeClient) {}
    
    public function processPayment(Payment $domainPayment): PaymentResult {
        // Translate domain concepts to Stripe API
        $stripeCharge = [
            'amount' => $domainPayment->getAmount()->getCents(),
            'currency' => $domainPayment->getAmount()->getCurrency()->getCode(),
            'source' => $this->translatePaymentMethod($domainPayment->getMethod()),
        ];
        
        $result = $this->stripeClient->charges->create($stripeCharge);
        
        // Translate Stripe response back to domain
        return new PaymentResult(
            new PaymentId($result->id),
            PaymentStatus::fromString($result->status)
        );
    }
}

// Bad: Domain model dependent on external system
class Payment {
    public function process(): void {
        // Direct dependency on external service
        $stripeResult = \Stripe\Charge::create([
            'amount' => $this->amount,
            'currency' => 'usd',
        ]);
        
        $this->stripeChargeId = $stripeResult->id; // Leaking external concepts
    }
}
```

---
description: "Define Context Maps for bounded context relationships"
alwaysApply: true
---

Explicitly define how bounded contexts relate to each other using context map patterns.

**Example:**
```php
// Customer-Supplier relationship between Order and Inventory contexts
namespace Order\Domain;
class OrderService {
    public function __construct(
        private InventoryServiceInterface $inventoryService,  // Upstream supplier
    ) {}
    
    public function placeOrder(CreateOrderCommand $command): Order {
        // Order context is downstream customer of Inventory context
        $availability = $this->inventoryService->checkAvailability(
            $command->getProductIds()
        );
        
        if (!$availability->allItemsAvailable()) {
            throw new InsufficientInventoryException();
        }
        
        return Order::create($command);
    }
}

// Published Language - Inventory publishes events for downstream contexts
namespace Inventory\Domain;
class Product {
    public function reserveStock(Quantity $quantity): void {
        $this->stock = $this->stock->subtract($quantity);
        
        // Publish event for downstream contexts
        $this->recordEvent(new StockReservedEvent(
            $this->id,
            $quantity,
            $this->stock
        ));
    }
}
```

# ========================================
# TACTICAL DESIGN - BUILDING BLOCKS
# ========================================

---
description: "Entities have identity and lifecycle, Value Objects are immutable"
alwaysApply: true
---

Entities are defined by identity and can change over time. Value Objects are immutable and defined by their attributes.

**Example:**
```php
// Good: Entity with identity
class Order {  // Entity
    private OrderId $id;
    private CustomerId $customerId;
    private OrderStatus $status;
    private array $lineItems = [];
    
    public function __construct(OrderId $id, CustomerId $customerId) {
        $this->id = $id;
        $this->customerId = $customerId;
        $this->status = OrderStatus::pending();
    }
    
    public function addLineItem(Product $product, Quantity $quantity): void {
        $this->lineItems[] = new OrderLineItem($product->getId(), $quantity);
    }
    
    public function confirm(): void {
        $this->status = OrderStatus::confirmed();
    }
}

// Good: Value Object is immutable
final readonly class Money {  // Value Object
    public function __construct(
        public int $amount,
        public Currency $currency
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('Amount cannot be negative');
        }
    }
    
    public function add(Money $other): self {
        if (!$this->currency->equals($other->currency)) {
            throw new CurrencyMismatchException();
        }
        
        return new self($this->amount + $other->amount, $this->currency);
    }
    
    public function equals(Money $other): bool {
        return $this->amount === $other->amount 
            && $this->currency->equals($other->currency);
    }
}

// Bad: Treating entity as value object or vice versa
class Money {  // Wrong: Mutable value object
    public int $amount;
    public function addAmount(int $value): void {
        $this->amount += $value;  // Value objects should be immutable
    }
}
```

---
description: "Aggregates enforce business invariants and control access"
alwaysApply: true
---

Aggregates are consistency boundaries. Only the aggregate root can be accessed from outside, and all invariants are enforced within the aggregate.

**Example:**
```php
// Good: Aggregate Root controlling access and invariants
class Order {  // Aggregate Root
    private OrderId $id;
    private CustomerId $customerId;
    private array $lineItems = [];
    private OrderStatus $status;
    
    public function addProduct(Product $product, Quantity $quantity): void {
        if ($this->status->isConfirmed()) {
            throw new OrderAlreadyConfirmedException();
        }
        
        if ($quantity->isZero()) {
            throw new InvalidQuantityException();
        }
        
        // Check business invariant: max 10 different products per order
        if (count($this->lineItems) >= 10) {
            throw new MaximumProductsExceededException();
        }
        
        $this->lineItems[] = new OrderLineItem($product->getId(), $quantity);
    }
    
    public function getTotal(): Money {
        return array_reduce(
            $this->lineItems,
            fn(Money $total, OrderLineItem $item) => $total->add($item->getSubtotal()),
            Money::zero()
        );
    }
}

class OrderLineItem {  // Entity within aggregate, not accessible from outside
    private ProductId $productId;
    private Quantity $quantity;
    private Money $unitPrice;
    
    // Only accessible through Order aggregate root
    public function getSubtotal(): Money {
        return $this->unitPrice->multiply($this->quantity->getValue());
    }
}

// Bad: Direct access to aggregate internals
class OrderService {
    public function addProduct(OrderId $orderId, ProductId $productId): void {
        $order = $this->orderRepository->find($orderId);
        
        // Wrong: Bypassing aggregate root
        $lineItem = new OrderLineItem($productId, new Quantity(1));
        $order->lineItems[] = $lineItem;  // Direct manipulation of internal state
    }
}
```

---
description: "Use Domain Services for operations that don't belong to a single entity"
alwaysApply: true
---

Domain Services contain business logic that doesn't naturally fit within an entity or value object.

**Example:**
```php
// Good: Domain Service for complex business operations
class TransferService {  // Domain Service
    public function transfer(
        BankAccount $fromAccount,
        BankAccount $toAccount,
        Money $amount
    ): void {
        // Business logic that involves multiple aggregates
        if (!$fromAccount->canWithdraw($amount)) {
            throw new InsufficientFundsException();
        }
        
        if (!$toAccount->canReceive($amount)) {
            throw new AccountLimitExceededException();
        }
        
        // Coordinate the transfer
        $fromAccount->withdraw($amount);
        $toAccount->deposit($amount);
    }
}

// Good: Using domain service in application service
class TransferApplicationService {
    public function __construct(
        private BankAccountRepository $accountRepository,
        private TransferService $transferService,  // Domain service
    ) {}
    
    public function transfer(TransferCommand $command): void {
        $fromAccount = $this->accountRepository->find($command->getFromAccountId());
        $toAccount = $this->accountRepository->find($command->getToAccountId());
        
        $this->transferService->transfer($fromAccount, $toAccount, $command->getAmount());
        
        $this->accountRepository->save($fromAccount);
        $this->accountRepository->save($toAccount);
    }
}

// Bad: Business logic in application service
class TransferApplicationService {
    public function transfer(TransferCommand $command): void {
        // Wrong: Domain logic in application layer
        $fromAccount = $this->accountRepository->find($command->getFromAccountId());
        $toAccount = $this->accountRepository->find($command->getToAccountId());
        
        if ($fromAccount->getBalance() < $command->getAmount()) {  // Domain logic
            throw new InsufficientFundsException();
        }
        
        $fromAccount->setBalance($fromAccount->getBalance() - $command->getAmount());
        $toAccount->setBalance($toAccount->getBalance() + $command->getAmount());
    }
}
```

---
description: "Repositories provide collection-like interface for aggregates"
alwaysApply: true
---

Repositories hide persistence details and provide domain-oriented query methods for aggregate roots only.

**Example:**
```php
// Good: Domain-oriented repository interface
interface OrderRepository {
    public function save(Order $order): void;
    public function find(OrderId $id): ?Order;
    public function findByCustomer(CustomerId $customerId): array;
    public function findPendingOrdersOlderThan(DateTimeInterface $date): array;
    public function nextIdentity(): OrderId;
}

// Good: Repository implementation hiding persistence details
class DoctrineOrderRepository implements OrderRepository {
    public function __construct(private EntityManagerInterface $em) {}
    
    public function save(Order $order): void {
        $this->em->persist($order);
    }
    
    public function findPendingOrdersOlderThan(DateTimeInterface $date): array {
        return $this->em->createQueryBuilder()
            ->select('o')
            ->from(Order::class, 'o')
            ->where('o.status = :status')
            ->andWhere('o.createdAt < :date')
            ->setParameter('status', OrderStatus::PENDING)
            ->setParameter('date', $date)
            ->getQuery()
            ->getResult();
    }
}

// Bad: Generic CRUD repository exposing persistence details
class GenericOrderRepository {
    public function findBy(array $criteria): array { }      // Too generic
    public function findOneBy(array $criteria): ?object { } // Not domain-specific
    public function executeQuery(string $sql): array { }    // Exposes persistence details
    
    // Bad: Allows access to non-aggregate entities
    public function findOrderLineItems(OrderId $id): array {
        // Bypasses aggregate root
    }
}
```

# ========================================
# DOMAIN EVENTS & MESSAGING
# ========================================

---
description: "Use Domain Events to decouple bounded contexts and capture business events"
alwaysApply: true
---

Domain Events represent something important that happened in the domain and allow loose coupling between bounded contexts.

**Example:**
```php
// Good: Domain Event represents business occurrence
final readonly class OrderConfirmedEvent {
    public function __construct(
        public OrderId $orderId,
        public CustomerId $customerId,
        public Money $totalAmount,
        public DateTimeInterface $confirmedAt,
    ) {}
}

// Good: Aggregate publishes domain events
class Order {
    private array $events = [];
    
    public function confirm(): void {
        if ($this->status->isConfirmed()) {
            throw new OrderAlreadyConfirmedException();
        }
        
        $this->status = OrderStatus::confirmed();
        
        // Record domain event
        $this->recordEvent(new OrderConfirmedEvent(
            $this->id,
            $this->customerId,
            $this->getTotal(),
            new DateTimeImmutable()
        ));
    }
    
    public function getUncommittedEvents(): array {
        return $this->events;
    }
    
    public function markEventsAsCommitted(): void {
        $this->events = [];
    }
    
    private function recordEvent(object $event): void {
        $this->events[] = $event;
    }
}

// Good: Event handlers in different bounded contexts
class InventoryEventHandler {
    public function handle(OrderConfirmedEvent $event): void {
        // Reserve inventory when order is confirmed
        $this->inventoryService->reserveStock($event->orderId);
    }
}

class BillingEventHandler {
    public function handle(OrderConfirmedEvent $event): void {
        // Generate invoice when order is confirmed
        $this->invoiceService->generateInvoice(
            $event->customerId,
            $event->totalAmount
        );
    }
}

// Bad: Direct coupling between contexts
class Order {
    public function confirm(): void {
        $this->status = OrderStatus::confirmed();
        
        // Wrong: Direct coupling to other contexts
        $this->inventoryService->reserveStock($this->id);
        $this->billingService->generateInvoice($this->customerId);
        $this->emailService->sendConfirmation($this->customerId);
    }
}
```

---
description: "Implement Event Sourcing for audit trails and temporal queries"
alwaysApply: false
---

Use Event Sourcing when you need complete audit trails or temporal queries. Store events instead of current state.

**Example:**
```php
// Good: Event Sourced Aggregate
class BankAccount {
    private AccountId $id;
    private Money $balance;
    private array $uncommittedEvents = [];
    
    public static function open(AccountId $id, Money $initialDeposit): self {
        $account = new self();
        $account->apply(new AccountOpenedEvent($id, $initialDeposit));
        return $account;
    }
    
    public function withdraw(Money $amount): void {
        if ($this->balance->isLessThan($amount)) {
            throw new InsufficientFundsException();
        }
        
        $this->apply(new MoneyWithdrawnEvent($this->id, $amount));
    }
    
    public function deposit(Money $amount): void {
        $this->apply(new MoneyDepositedEvent($this->id, $amount));
    }
    
    private function apply(object $event): void {
        $this->when($event);
        $this->uncommittedEvents[] = $event;
    }
    
    // Rebuild state from events
    private function when(object $event): void {
        match(get_class($event)) {
            AccountOpenedEvent::class => $this->whenAccountOpened($event),
            MoneyDepositedEvent::class => $this->whenMoneyDeposited($event),
            MoneyWithdrawnEvent::class => $this->whenMoneyWithdrawn($event),
        };
    }
    
    private function whenAccountOpened(AccountOpenedEvent $event): void {
        $this->id = $event->accountId;
        $this->balance = $event->initialDeposit;
    }
    
    private function whenMoneyDeposited(MoneyDepositedEvent $event): void {
        $this->balance = $this->balance->add($event->amount);
    }
    
    private function whenMoneyWithdrawn(MoneyWithdrawnEvent $event): void {
        $this->balance = $this->balance->subtract($event->amount);
    }
}

// Event Store Repository
interface EventStore {
    public function saveEvents(string $aggregateId, array $events, int $expectedVersion): void;
    public function getEventsForAggregate(string $aggregateId): array;
}
```

# ========================================
# APPLICATION LAYER & COORDINATION
# ========================================

---
description: "Application Services orchestrate use cases without business logic"
alwaysApply: true
---

Application Services coordinate domain objects and infrastructure but contain no business logic.

**Example:**
```php
// Good: Application Service coordinates use case
class OrderApplicationService {
    public function __construct(
        private OrderRepository $orderRepository,
        private ProductRepository $productRepository,
        private DomainEventDispatcher $eventDispatcher,
    ) {}
    
    public function placeOrder(PlaceOrderCommand $command): void {
        // Coordination only, no business logic
        $customer = $this->customerRepository->find($command->getCustomerId());
        $order = $customer->createOrder();  // Business logic in domain
        
        foreach ($command->getItems() as $item) {
            $product = $this->productRepository->find($item->getProductId());
            $order->addProduct($product, $item->getQuantity());  // Business logic in domain
        }
        
        $this->orderRepository->save($order);
        $this->eventDispatcher->dispatchEvents($order->getUncommittedEvents());
    }
}

// Good: Command represents use case intent  
final readonly class PlaceOrderCommand {
    public function __construct(
        public CustomerId $customerId,
        /** @var OrderItemData[] */
        public array $items,
    ) {}
}

final readonly class OrderItemData {
    public function __construct(
        public ProductId $productId,
        public int $quantity,
    ) {}
}

// Bad: Business logic in application service
class OrderApplicationService {
    public function placeOrder(PlaceOrderCommand $command): void {
        $order = new Order($command->getCustomerId());
        
        // Wrong: Business logic in application layer
        $total = 0;
        foreach ($command->getItems() as $item) {
            $product = $this->productRepository->find($item->getProductId());
            if ($product->getStock() < $item->getQuantity()) {
                throw new InsufficientStockException();
            }
            $total += $product->getPrice() * $item->getQuantity();
        }
        
        if ($total > 1000) {  // Business rule
            throw new OrderTooLargeException();
        }
    }
}
```

---
description: "Use CQRS to separate read and write models when complexity justifies it"
alwaysApply: false
---

Separate Command (write) and Query (read) models when read and write requirements differ significantly.

**Example:**
```php
// Good: Separate command and query models
// Command Side - focused on writes
class CreateProductCommand {
    public function __construct(
        public string $name,
        public string $description,
        public float $price,
    ) {}
}

class ProductCommandHandler {
    public function handle(CreateProductCommand $command): void {
        $product = Product::create(
            $this->productRepository->nextIdentity(),
            $command->name,
            new Money($command->price * 100, Currency::USD())
        );
        
        $this->productRepository->save($product);
    }
}

// Query Side - optimized for reads
class ProductListQuery {
    public function __construct(
        public ?string $category = null,
        public ?string $searchTerm = null,
        public int $page = 1,
        public int $limit = 20,
    ) {}
}

class ProductQueryHandler {
    public function __construct(private ProductReadModelRepository $readModel) {}
    
    public function handle(ProductListQuery $query): ProductListResult {
        // Optimized read model with denormalized data
        return $this->readModel->findProductList(
            $query->category,
            $query->searchTerm,
            $query->page,
            $query->limit
        );
    }
}

// Read Model - optimized for queries
class ProductListItem {
    public function __construct(
        public string $id,
        public string $name,
        public string $formattedPrice,
        public string $categoryName,      // Denormalized
        public int $reviewCount,          // Denormalized
        public float $averageRating,      // Denormalized
    ) {}
}
```

---
description: "Use Sagas/Process Managers for long-running business processes"
alwaysApply: false
---

Implement Sagas to coordinate multi-step business processes across aggregates or bounded contexts.

**Example:**
```php
// Good: Saga coordinates order fulfillment process
class OrderFulfillmentSaga {
    private OrderId $orderId;
    private SagaState $state;
    
    public function handle(OrderPlacedEvent $event): array {
        $this->orderId = $event->orderId;
        $this->state = SagaState::PAYMENT_PENDING;
        
        return [
            new ProcessPaymentCommand($event->orderId, $event->amount)
        ];
    }
    
    public function handle(PaymentProcessedEvent $event): array {
        if ($this->state !== SagaState::PAYMENT_PENDING) {
            return [];
        }
        
        $this->state = SagaState::INVENTORY_PENDING;
        
        return [
            new ReserveInventoryCommand($event->orderId)
        ];
    }
    
    public function handle(InventoryReservedEvent $event): array {
        if ($this->state !== SagaState::INVENTORY_PENDING) {
            return [];
        }
        
        $this->state = SagaState::SHIPPING_PENDING;
        
        return [
            new ScheduleShippingCommand($event->orderId)
        ];
    }
    
    public function handle(PaymentFailedEvent $event): array {
        $this->state = SagaState::FAILED;
        
        return [
            new CancelOrderCommand($event->orderId),
            new NotifyCustomerCommand($event->customerId, 'Payment failed')
        ];
    }
}

enum SagaState {
    case PAYMENT_PENDING;
    case INVENTORY_PENDING; 
    case SHIPPING_PENDING;
    case COMPLETED;
    case FAILED;
}
```

# ========================================
# ANTI-PATTERNS & COMMON MISTAKES
# ========================================

---
description: "Avoid anemic domain models - put behavior with data"
alwaysApply: true
---

Domain models should contain both data and behavior. Avoid models that are just data containers with business logic elsewhere.

**Example:**
```php
// Bad: Anemic Domain Model - just data containers
class Order {
    public OrderId $id;
    public array $items = [];
    public OrderStatus $status;
    public Money $total;
    
    // No behavior, just getters/setters
    public function getTotal(): Money { return $this->total; }
    public function setTotal(Money $total): void { this->total = $total; }
}

class OrderService {
    // Business logic separated from data
    public function calculateTotal(Order $order): Money {
        $total = Money::zero();
        foreach ($order->items as $item) {
            $total = $total->add($item->getPrice());
        }
        return $total;
    }
    
    public function confirmOrder(Order $order): void {
        if ($order->status === OrderStatus::PENDING) {
            $order->status = OrderStatus::CONFIRMED;
        }
    }
}

// Good: Rich Domain Model - behavior with data
class Order {
    private OrderId $id;
    private array $items = [];
    private OrderStatus $status;
    
    public function addItem(Product $product, Quantity $quantity): void {
        if ($this->status->isConfirmed()) {
            throw new OrderAlreadyConfirmedException();
        }
        
        $this->items[] = new OrderItem($product, $quantity);
    }
    
    public function calculateTotal(): Money {
        return array_reduce(
            $this->items,
            fn(Money $total, OrderItem $item) => $total->add($item->getSubtotal()),
            Money::zero()
        );
    }
    
    public function confirm(): void {
        if (empty($this->items)) {
            throw new EmptyOrderException();
        }
        
        if ($this->status->isConfirmed()) {
            throw new OrderAlreadyConfirmedException();
        }
        
        $this->status = OrderStatus::confirmed();
        $this->recordEvent(new OrderConfirmedEvent($this->id));
    }
}
```

---
description: "Don't leak domain concepts to infrastructure layer"
alwaysApply: true
---

Infrastructure should translate between domain concepts and external systems, not expose external concerns to domain.

**Example:**
```php
// Bad: Domain depends on infrastructure details
class Order {
    private string $customerId;  // Should be domain CustomerId
    private array $items;
    
    public function save(): void {
        // Wrong: Domain knows about persistence
        $pdo = new PDO('mysql:host=localhost;dbname=orders');
        $stmt = $pdo->prepare('INSERT INTO orders...');
        $stmt->execute([
            'customer_id' => $this->customerId,
            'items' => json_encode($this->items)  // Infrastructure concern
        ]);
    }
}

// Good: Infrastructure translates domain concepts
class Order {
    private CustomerId $customerId;  // Domain concept
    private array $items;            // Domain OrderItem objects
    
    public function getCustomerId(): CustomerId {
        return $this->customerId;
    }
    
    public function getItems(): array {
        return $this->items;
    }
}

class DoctrineOrderRepository implements OrderRepository {
    public function save(Order $order): void {
        // Infrastructure handles translation
        $entity = new OrderEntity();
        $entity->customerId = $order->getCustomerId()->getValue();  // Translate to primitive
        $entity->items = array_map(
            fn(OrderItem $item) => $this->mapOrderItem($item),
            $order->getItems()
        );
        
        $this->entityManager->persist($entity);
    }
    
    private function mapOrderItem(OrderItem $domainItem): OrderItemEntity {
        // Translate domain objects to infrastructure entities
        $entity = new OrderItemEntity();
        $entity->productId = $domainItem->getProductId()->getValue();
        $entity->quantity = $domainItem->getQuantity()->getValue();
        return $entity;
    }
}
```

---
description: "Avoid domain logic in DTOs and application services"
alwaysApply: true
---

DTOs should only transfer data. Business logic belongs in domain objects, not in DTOs or application services.

**Example:**
```php
// Bad: Business logic in DTO
class CreateOrderDTO {
    public array $items;
    public string $customerId;
    
    public function getTotalPrice(): float {
        // Wrong: Business logic in DTO
        $total = 0;
        foreach ($this->items as $item) {
            $total += $item['price'] * $item['quantity'];
        }
        return $total;
    }
    
    public function isValidOrder(): bool {
        // Wrong: Business logic in DTO
        return count($this->items) > 0 && $this->getTotalPrice() > 0;
    }
}

// Good: DTO only transfers data
class CreateOrderDTO {
    public function __construct(
        public readonly string $customerId,
        public readonly array $items,
    ) {}
}

// Good: Business logic in domain
class Order {
    public function calculateTotal(): Money {
        return array_reduce(
            $this->items,
            fn(Money $total, OrderItem $item) => $total->add($item->getSubtotal()),
            Money::zero()
        );
    }
    
    public function isValid(): bool {
        return !empty($this->items) && $this->calculateTotal()->isGreaterThan(Money::zero());
    }
}

// Good: Application service coordinates
class OrderApplicationService {
    public function createOrder(CreateOrderDTO $dto): void {
        $customerId = new CustomerId($dto->customerId);
        $order = Order::createFor($customerId);
        
        foreach ($dto->items as $itemData) {
            $product = $this->productRepository->find(new ProductId($itemData['productId']));
            $order->addItem($product, new Quantity($itemData['quantity']));  // Domain logic
        }
        
        if (!$order->isValid()) {  // Domain logic
            throw new InvalidOrderException();
        }
        
        $this->orderRepository->save($order);
    }
}
```

---
description: "Don't create god aggregates - keep them focused and small"
alwaysApply: true
---

Aggregates should be small and focused on a single invariant. Large aggregates create performance and concurrency issues.

**Example:**
```php
// Bad: God aggregate with too many responsibilities
class Customer {
    private CustomerId $id;
    private string $name;
    private Email $email;
    
    // Wrong: Order management in Customer aggregate
    private array $orders = [];
    private array $payments = [];
    private array $shipments = [];
    private array $invoices = [];
    private array $returns = [];
    
    public function placeOrder(array $items): Order { /* Complex logic */ }
    public function makePayment(Money $amount): Payment { /* Complex logic */ }
    public function scheduleShipment(Order $order): Shipment { /* Complex logic */ }
    public function processReturn(Order $order): Return { /* Complex logic */ }
}

// Good: Separate focused aggregates
class Customer {  // Focused on customer identity and basic info
    private CustomerId $id;
    private string $name;
    private Email $email;
    private CustomerStatus $status;
    
    public function changeName(string $newName): void {
        $this->name = $newName;
    }
    
    public function changeEmail(Email $newEmail): void {
        $this->email = $newEmail;
    }
}

class Order {  // Focused on order lifecycle and invariants
    private OrderId $id;
    private CustomerId $customerId;  // Reference to customer, not embedded
    private array $items = [];
    private OrderStatus $status;
    
    public function addItem(Product $product, Quantity $quantity): void {
        // Order-specific business logic
    }
}

class Payment {  // Focused on payment processing
    private PaymentId $id;
    private OrderId $orderId;  // Reference to order
    private Money $amount;
    private PaymentStatus $status;
}
```

