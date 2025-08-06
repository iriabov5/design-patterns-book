# Facade Pattern

## Обзор паттерна

**Facade Pattern** - это структурный паттерн проектирования, который предоставляет упрощенный интерфейс к сложной подсистеме, скрывая её внутреннюю сложность.

### Что это такое?

Facade Pattern создает единую точку входа для взаимодействия с подсистемой, инкапсулируя сложность и предоставляя простой интерфейс. Паттерн не изменяет функциональность подсистемы, а только упрощает её использование, скрывая детали реализации и координируя работу различных компонентов.

В микросервисной архитектуре Facade Pattern особенно полезен для создания API Gateway, упрощения взаимодействия с внешними сервисами, объединения сложных операций и предоставления единого интерфейса для клиентов.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда клиент должен взаимодействовать с множеством сложных подсистем, каждая из которых имеет свой интерфейс и логику. Например:

1. **Сценарий**: Клиент хочет создать заказ, который требует взаимодействия с сервисами товаров, пользователей, платежей и уведомлений
2. **Требование**: Нужно предоставить простой интерфейс для создания заказа, скрывая сложность взаимодействия с множеством сервисов
3. **Проблема**: Если клиент напрямую взаимодействует с каждым сервисом, код становится сложным и трудно поддерживаемым
4. **Результат**: Сложный код с множеством зависимостей, который сложно тестировать и отлаживать

## Детальный анализ проблемы

### Сценарий 1: Прямое взаимодействие с подсистемами

```java
@Service
public class OrderService {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private InventoryService inventoryService;
    
    public OrderResult createOrder(OrderRequest request) {
        // 1. Проверяем товар (ПРОБЛЕМА ЗДЕСЬ!)
        Product product = productService.getProduct(request.getProductId());
        if (product == null) {
            throw new ProductNotFoundException(request.getProductId());
        }
        
        // 2. Проверяем пользователя
        User user = userService.getUser(request.getUserId());
        if (user == null) {
            throw new UserNotFoundException(request.getUserId());
        }
        
        // 3. Проверяем наличие товара
        boolean available = inventoryService.checkAvailability(request.getProductId(), request.getQuantity());
        if (!available) {
            throw new ProductNotAvailableException(request.getProductId());
        }
        
        // 4. Обрабатываем платеж
        PaymentResult payment = paymentService.processPayment(
            user.getPaymentMethod(),
            product.getPrice() * request.getQuantity()
        );
        
        if (!payment.isSuccess()) {
            throw new PaymentFailedException(payment.getError());
        }
        
        // 5. Создаем заказ
        Order order = new Order(user.getId(), product.getId(), request.getQuantity(), payment.getTransactionId());
        order = orderRepository.save(order);
        
        // 6. Обновляем инвентарь
        inventoryService.updateStock(request.getProductId(), request.getQuantity());
        
        // 7. Отправляем уведомление
        notificationService.sendOrderConfirmation(user.getEmail(), order);
        
        return new OrderResult(order.getId(), "SUCCESS");
    }
}
```

**Проблемы:**
- Нарушение принципа единственной ответственности - класс знает о всех подсистемах
- Сложность тестирования - каждый сервис нужно тестировать в контексте основного класса
- Высокая связанность - изменение одного сервиса может повлиять на другие
- Сложность отладки - трудно понять, какой сервис вызвал ошибку

### Сценарий 2: Наследование от базового сервиса

```java
public abstract class BaseOrderService {
    protected abstract ProductService getProductService();
    protected abstract UserService getUserService();
    protected abstract PaymentService getPaymentService();
    
    public OrderResult createOrder(OrderRequest request) {
        // Общая логика создания заказа
        Product product = getProductService().getProduct(request.getProductId());
        User user = getUserService().getUser(request.getUserId());
        
        // ПРОБЛЕМА ЗДЕСЬ!
        PaymentResult payment = getPaymentService().processPayment(
            user.getPaymentMethod(),
            product.getPrice() * request.getQuantity()
        );
        
        return new OrderResult(1L, "SUCCESS");
    }
}

@Service
public class ConcreteOrderService extends BaseOrderService {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Override
    protected ProductService getProductService() {
        return productService;
    }
    
    @Override
    protected UserService getUserService() {
        return userService;
    }
    
    @Override
    protected PaymentService getPaymentService() {
        return paymentService;
    }
}
```

**Проблемы:**
- Сложность наследования - каждый подкласс должен реализовать все методы
- Нарушение принципа композиции - предпочтение наследования над композицией
- Сложность расширения - добавление нового сервиса требует изменения базового класса
- Проблемы с тестированием - сложно мокать абстрактные методы

### Сценарий 3: Утилитарный класс

```java
public class OrderUtils {
    
    public static OrderResult createOrder(OrderRequest request) {
        // ПРОБЛЕМА ЗДЕСЬ!
        Product product = ProductService.getInstance().getProduct(request.getProductId());
        User user = UserService.getInstance().getUser(request.getUserId());
        PaymentResult payment = PaymentService.getInstance().processPayment(
            user.getPaymentMethod(),
            product.getPrice() * request.getQuantity()
        );
        
        return new OrderResult(1L, "SUCCESS");
    }
    
    public static void cancelOrder(Long orderId) {
        // Логика отмены заказа
    }
    
    public static OrderStatus getOrderStatus(Long orderId) {
        // Логика получения статуса заказа
    }
}

@Service
public class OrderServiceWithUtils {
    
    public OrderResult createOrder(OrderRequest request) {
        return OrderUtils.createOrder(request);
    }
}
```

**Проблемы:**
- Статические методы - сложно тестировать и мокать
- Нарушение принципов ООП - нет инкапсуляции состояния
- Сложность конфигурации - утилитарные классы не поддерживают DI
- Проблемы с состоянием - статические методы не могут иметь состояние

### Корень проблемы

Основная проблема заключается в том, что **клиент должен знать о всех подсистемах и их интерфейсах**. Это приводит к:

1. **Нарушению принципа единственной ответственности** - классы знают о множестве подсистем
2. **Сложности тестирования** - каждый сервис нужно тестировать в контексте основного класса
3. **Высокой связанности** - изменение одной подсистемы влияет на другие
4. **Сложности отладки** - трудно понять, какая подсистема вызвала ошибку

### Как работает паттерн

1. **Создание фасада** - создается класс, который предоставляет упрощенный интерфейс
2. **Инкапсуляция подсистем** - фасад скрывает сложность взаимодействия с подсистемами
3. **Координация операций** - фасад координирует работу различных компонентов
4. **Упрощение интерфейса** - клиент взаимодействует только с фасадом

## Детальный принцип работы

### Шаг 1: Определение интерфейса фасада

```java
public interface OrderFacade {
    OrderResult createOrder(OrderRequest request);
    OrderResult cancelOrder(Long orderId);
    OrderStatus getOrderStatus(Long orderId);
    List<Order> getUserOrders(Long userId);
}
```

**Ключевые моменты:**
- Интерфейс определяет простой контракт для клиентов
- Методы скрывают сложность взаимодействия с подсистемами
- Фасад предоставляет только необходимые операции

### Шаг 2: Реализация фасада

```java
@Service
public class OrderFacadeImpl implements OrderFacade {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Override
    @Transactional
    public OrderResult createOrder(OrderRequest request) {
        try {
            // 1. Валидируем запрос
            validateOrderRequest(request);
            
            // 2. Получаем данные о товаре и пользователе
            Product product = getProduct(request.getProductId());
            User user = getUser(request.getUserId());
            
            // 3. Проверяем наличие товара
            checkProductAvailability(request.getProductId(), request.getQuantity());
            
            // 4. Обрабатываем платеж
            PaymentResult payment = processPayment(user, product, request.getQuantity());
            
            // 5. Создаем заказ
            Order order = createOrderEntity(user, product, request, payment);
            
            // 6. Обновляем инвентарь
            updateInventory(request.getProductId(), request.getQuantity());
            
            // 7. Отправляем уведомление
            sendOrderConfirmation(user, order);
            
            return new OrderResult(order.getId(), "SUCCESS");
            
        } catch (Exception e) {
            log.error("Error creating order: {}", e.getMessage(), e);
            return new OrderResult(null, "FAILED", e.getMessage());
        }
    }
    
    @Override
    @Transactional
    public OrderResult cancelOrder(Long orderId) {
        try {
            Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));
            
            // Проверяем возможность отмены
            if (!canCancelOrder(order)) {
                return new OrderResult(orderId, "CANCEL_FAILED", "Order cannot be cancelled");
            }
            
            // Отменяем платеж
            paymentService.refundPayment(order.getPaymentTransactionId());
            
            // Возвращаем товар в инвентарь
            inventoryService.returnToStock(order.getProductId(), order.getQuantity());
            
            // Обновляем статус заказа
            order.setStatus(OrderStatus.CANCELLED);
            orderRepository.save(order);
            
            // Отправляем уведомление об отмене
            User user = userService.getUser(order.getUserId());
            notificationService.sendOrderCancellation(user.getEmail(), order);
            
            return new OrderResult(orderId, "CANCELLED");
            
        } catch (Exception e) {
            log.error("Error cancelling order: {}", e.getMessage(), e);
            return new OrderResult(orderId, "CANCEL_FAILED", e.getMessage());
        }
    }
    
    @Override
    public OrderStatus getOrderStatus(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        return order.getStatus();
    }
    
    @Override
    public List<Order> getUserOrders(Long userId) {
        return orderRepository.findByUserId(userId);
    }
    
    // Приватные методы для инкапсуляции логики
    private void validateOrderRequest(OrderRequest request) {
        if (request.getProductId() == null) {
            throw new IllegalArgumentException("Product ID is required");
        }
        if (request.getUserId() == null) {
            throw new IllegalArgumentException("User ID is required");
        }
        if (request.getQuantity() <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
    }
    
    private Product getProduct(Long productId) {
        Product product = productService.getProduct(productId);
        if (product == null) {
            throw new ProductNotFoundException(productId);
        }
        return product;
    }
    
    private User getUser(Long userId) {
        User user = userService.getUser(userId);
        if (user == null) {
            throw new UserNotFoundException(userId);
        }
        return user;
    }
    
    private void checkProductAvailability(Long productId, int quantity) {
        boolean available = inventoryService.checkAvailability(productId, quantity);
        if (!available) {
            throw new ProductNotAvailableException(productId);
        }
    }
    
    private PaymentResult processPayment(User user, Product product, int quantity) {
        PaymentResult payment = paymentService.processPayment(
            user.getPaymentMethod(),
            product.getPrice() * quantity
        );
        
        if (!payment.isSuccess()) {
            throw new PaymentFailedException(payment.getError());
        }
        
        return payment;
    }
    
    private Order createOrderEntity(User user, Product product, OrderRequest request, PaymentResult payment) {
        Order order = new Order(
            user.getId(),
            product.getId(),
            request.getQuantity(),
            payment.getTransactionId(),
            product.getPrice() * request.getQuantity()
        );
        return orderRepository.save(order);
    }
    
    private void updateInventory(Long productId, int quantity) {
        inventoryService.updateStock(productId, quantity);
    }
    
    private void sendOrderConfirmation(User user, Order order) {
        notificationService.sendOrderConfirmation(user.getEmail(), order);
    }
    
    private boolean canCancelOrder(Order order) {
        return order.getStatus() == OrderStatus.CREATED || 
               order.getStatus() == OrderStatus.PROCESSING;
    }
}
```

### Шаг 3: Конфигурация фасада

```java
@Configuration
public class OrderFacadeConfig {
    
    @Bean
    public OrderFacade orderFacade(
            ProductService productService,
            UserService userService,
            PaymentService paymentService,
            NotificationService notificationService,
            InventoryService inventoryService,
            OrderRepository orderRepository) {
        
        return new OrderFacadeImpl(
            productService,
            userService,
            paymentService,
            notificationService,
            inventoryService,
            orderRepository
        );
    }
}
```

### Шаг 4: Контроллер с использованием фасада

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderFacade orderFacade;
    
    @PostMapping
    public ResponseEntity<OrderResult> createOrder(@RequestBody OrderRequest request) {
        OrderResult result = orderFacade.createOrder(request);
        
        if ("SUCCESS".equals(result.getStatus())) {
            return ResponseEntity.ok(result);
        } else {
            return ResponseEntity.badRequest().body(result);
        }
    }
    
    @DeleteMapping("/{orderId}")
    public ResponseEntity<OrderResult> cancelOrder(@PathVariable Long orderId) {
        OrderResult result = orderFacade.cancelOrder(orderId);
        
        if ("CANCELLED".equals(result.getStatus())) {
            return ResponseEntity.ok(result);
        } else {
            return ResponseEntity.badRequest().body(result);
        }
    }
    
    @GetMapping("/{orderId}/status")
    public ResponseEntity<OrderStatus> getOrderStatus(@PathVariable Long orderId) {
        OrderStatus status = orderFacade.getOrderStatus(orderId);
        return ResponseEntity.ok(status);
    }
    
    @GetMapping("/user/{userId}")
    public ResponseEntity<List<Order>> getUserOrders(@PathVariable Long userId) {
        List<Order> orders = orderFacade.getUserOrders(userId);
        return ResponseEntity.ok(orders);
    }
}
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Client        │    │   OrderFacade   │    │ ProductService  │
│                 │    │                 │    │                 │
│ 1. Create Order │───▶│ 2. Coordinate   │───▶│ 3. Get Product  │
│                 │    │    Operations    │    │                 │
│ 6. Get Result   │◀───│ 5. Return       │◀───│ 4. Return       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │ UserService     │
                       │                 │
                       │ Get User        │
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │PaymentService   │
                       │                 │
                       │Process Payment  │
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │InventoryService │
                       │                 │
                       │Update Stock     │
                       └─────────────────┘
```

### Ключевые преимущества

1. **Упрощение интерфейса**: Клиент взаимодействует только с фасадом
2. **Инкапсуляция сложности**: Детали подсистем скрыты от клиента
3. **Координация операций**: Фасад координирует работу подсистем
4. **Слабая связанность**: Клиент не зависит от конкретных подсистем
5. **Упрощение тестирования**: Фасад можно тестировать независимо

### Конфигурация фасада

Описание конфигурируемых параметров для настройки фасада

**Пример конфигурации:**
```properties
# application.properties
facade.order.timeout=30000
facade.order.retry.attempts=3
facade.order.retry.delay=1000
facade.notification.enabled=true
facade.payment.timeout=15000
```

**Альтернативные стратегии:**
- **Стратегия синхронного вызова**: Все операции выполняются последовательно
- **Стратегия асинхронного вызова**: Операции выполняются параллельно с помощью CompletableFuture
- **Стратегия кэширования**: Результаты подсистем кэшируются для улучшения производительности
- **Стратегия отказоустойчивости**: Реализация Circuit Breaker для защиты от сбоев подсистем

### Ключевые принципы

- **Единственная ответственность**: Фасад отвечает только за координацию подсистем
- **Инкапсуляция**: Сложность подсистем скрыта от клиента
- **Слабая связанность**: Клиент не знает о внутренней структуре
- **Простота использования**: Клиент получает простой интерфейс

### Когда использовать

Facade Pattern подходит для случаев, когда:
- Есть сложная подсистема с множеством компонентов
- Нужно упростить взаимодействие с подсистемой
- Требуется единая точка входа для операций
- Важна инкапсуляция сложности от клиента

## Преимущества и недостатки

### Преимущества

#### 1. **Упрощение интерфейса**
- Клиент получает простой и понятный интерфейс
- Скрывается сложность взаимодействия с подсистемами
- Уменьшается количество зависимостей для клиента

#### 2. **Инкапсуляция сложности**
- Детали реализации подсистем скрыты от клиента
- Изменения в подсистемах не влияют на клиента
- Единая точка для координации операций

#### 3. **Упрощение тестирования**
- Фасад можно тестировать независимо от подсистем
- Моки и стабы легко создавать для подсистем
- Изолированное тестирование логики координации

#### 4. **Слабая связанность**
- Клиент не зависит от конкретных подсистем
- Подсистемы можно изменять независимо
- Легко добавлять новые подсистемы

#### 5. **Координация операций**
- Фасад координирует работу различных компонентов
- Обеспечивается правильный порядок операций
- Централизованная обработка ошибок

#### 6. **Повторное использование**
- Фасад можно использовать в разных контекстах
- Общая логика координации выносится в отдельный класс
- Уменьшение дублирования кода

### Недостатки

#### 1. **Потенциальная избыточность**
- Для простых случаев фасад может быть избыточным
- Дополнительный слой абстракции
- Риск over-engineering

#### 2. **Сложность отладки**
- Логика распределена между фасадом и подсистемами
- Сложнее отследить выполнение операций
- Может потребоваться дополнительное логирование

#### 3. **Зависимость от фасада**
- Клиент зависит от интерфейса фасада
- Изменение фасада влияет на всех клиентов
- Сложность эволюции API

#### 4. **Потенциальные проблемы производительности**
- Дополнительный слой вызовов методов
- Возможные накладные расходы на координацию
- Риск создания узкого места

#### 5. **Сложность конфигурации**
- Нужно правильно настроить зависимости фасада
- Может потребоваться сложная конфигурация DI
- Сложность управления жизненным циклом

#### 6. **Риск создания "божественного объекта"**
- Фасад может стать слишком большим и сложным
- Нарушение принципа единственной ответственности
- Сложность поддержки и расширения

### Сравнение с альтернативами

| Аспект | Facade Pattern | Adapter Pattern | Mediator Pattern |
|--------|----------------|-----------------|------------------|
| **Сложность** | Низкая | Средняя | Высокая |
| **Гибкость** | Средняя | Высокая | Высокая |
| **Тестируемость** | Высокая | Средняя | Высокая |
| **Производительность** | Высокая | Средняя | Низкая |
| **Отладка** | Средняя | Сложная | Сложная |

## Тестирование

Интеграционные тесты показывают полный цикл работы Facade Pattern и помогают понять, как фасад координирует работу подсистем.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest
@Transactional
class OrderFacadeIntegrationTest {
    
    @Autowired
    private OrderFacade orderFacade;
    
    @MockBean
    private ProductService productService;
    
    @MockBean
    private UserService userService;
    
    @MockBean
    private PaymentService paymentService;
    
    @MockBean
    private NotificationService notificationService;
    
    @MockBean
    private InventoryService inventoryService;
    
    @Test
    void shouldCreateOrderSuccessfully() {
        // Given
        OrderRequest request = new OrderRequest(1L, 1L, 2);
        Product product = new Product(1L, "Test Product", 100.0);
        User user = new User(1L, "user@test.com", "VISA");
        PaymentResult payment = new PaymentResult("TXN_123", true, null);
        
        when(productService.getProduct(1L)).thenReturn(product);
        when(userService.getUser(1L)).thenReturn(user);
        when(inventoryService.checkAvailability(1L, 2)).thenReturn(true);
        when(paymentService.processPayment("VISA", 200.0)).thenReturn(payment);
        
        // When
        OrderResult result = orderFacade.createOrder(request);
        
        // Then - проверяем, что заказ создан успешно
        assertThat(result.getStatus()).isEqualTo("SUCCESS");
        assertThat(result.getOrderId()).isNotNull();
        
        // And - проверяем, что все сервисы были вызваны
        verify(productService).getProduct(1L);
        verify(userService).getUser(1L);
        verify(inventoryService).checkAvailability(1L, 2);
        verify(paymentService).processPayment("VISA", 200.0);
        verify(notificationService).sendOrderConfirmation("user@test.com", any(Order.class));
    }
    
    @Test
    void shouldHandleProductNotFound() {
        // Given
        OrderRequest request = new OrderRequest(999L, 1L, 2);
        when(productService.getProduct(999L)).thenReturn(null);
        
        // When
        OrderResult result = orderFacade.createOrder(request);
        
        // Then - проверяем, что заказ не создан
        assertThat(result.getStatus()).isEqualTo("FAILED");
        assertThat(result.getErrorMessage()).contains("Product not found");
        
        // And - проверяем, что следующие сервисы не были вызваны
        verify(userService, never()).getUser(any());
        verify(paymentService, never()).processPayment(any(), any());
    }
    
    @Test
    void shouldHandlePaymentFailure() {
        // Given
        OrderRequest request = new OrderRequest(1L, 1L, 2);
        Product product = new Product(1L, "Test Product", 100.0);
        User user = new User(1L, "user@test.com", "VISA");
        PaymentResult payment = new PaymentResult(null, false, "Insufficient funds");
        
        when(productService.getProduct(1L)).thenReturn(product);
        when(userService.getUser(1L)).thenReturn(user);
        when(inventoryService.checkAvailability(1L, 2)).thenReturn(true);
        when(paymentService.processPayment("VISA", 200.0)).thenReturn(payment);
        
        // When
        OrderResult result = orderFacade.createOrder(request);
        
        // Then - проверяем, что заказ не создан из-за ошибки платежа
        assertThat(result.getStatus()).isEqualTo("FAILED");
        assertThat(result.getErrorMessage()).contains("Payment failed");
        
        // And - проверяем, что уведомление не отправлялось
        verify(notificationService, never()).sendOrderConfirmation(any(), any());
    }
    
    @Test
    void shouldCancelOrderSuccessfully() {
        // Given
        Long orderId = 1L;
        Order order = new Order(1L, 1L, 2, "TXN_123", 200.0);
        order.setStatus(OrderStatus.CREATED);
        
        when(orderRepository.findById(orderId)).thenReturn(Optional.of(order));
        when(userService.getUser(1L)).thenReturn(new User(1L, "user@test.com", "VISA"));
        
        // When
        OrderResult result = orderFacade.cancelOrder(orderId);
        
        // Then - проверяем, что заказ отменен успешно
        assertThat(result.getStatus()).isEqualTo("CANCELLED");
        assertThat(result.getOrderId()).isEqualTo(orderId);
        
        // And - проверяем, что все операции отмены выполнены
        verify(paymentService).refundPayment("TXN_123");
        verify(inventoryService).returnToStock(1L, 2);
        verify(notificationService).sendOrderCancellation("user@test.com", order);
    }
}
```

#### TestOrderFacade для тестирования

```java
@Component
@Primary
public class TestOrderFacade implements OrderFacade {
    
    private final List<OrderRequest> createdOrders = new ArrayList<>();
    private final List<Long> cancelledOrders = new ArrayList<>();
    private boolean shouldFail = false;
    private String failureReason = "Test failure";
    
    @Override
    public OrderResult createOrder(OrderRequest request) {
        if (shouldFail) {
            return new OrderResult(null, "FAILED", failureReason);
        }
        
        createdOrders.add(request);
        return new OrderResult(1L, "SUCCESS");
    }
    
    @Override
    public OrderResult cancelOrder(Long orderId) {
        if (shouldFail) {
            return new OrderResult(orderId, "CANCEL_FAILED", failureReason);
        }
        
        cancelledOrders.add(orderId);
        return new OrderResult(orderId, "CANCELLED");
    }
    
    @Override
    public OrderStatus getOrderStatus(Long orderId) {
        return OrderStatus.CREATED;
    }
    
    @Override
    public List<Order> getUserOrders(Long userId) {
        return List.of(new Order(userId, 1L, 1, "TXN_TEST", 100.0));
    }
    
    public List<OrderRequest> getCreatedOrders() {
        return new ArrayList<>(createdOrders);
    }
    
    public List<Long> getCancelledOrders() {
        return new ArrayList<>(cancelledOrders);
    }
    
    public void clear() {
        createdOrders.clear();
        cancelledOrders.clear();
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
    
    public void setFailureReason(String failureReason) {
        this.failureReason = failureReason;
    }
}
``` 