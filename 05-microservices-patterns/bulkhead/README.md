# Bulkhead Pattern

## Обзор паттерна

**Bulkhead** - это паттерн проектирования для изоляции ресурсов и предотвращения каскадных сбоев в микросервисной архитектуре.

### Что это такое?

Bulkhead Pattern основан на принципе кораблестроения, где корабль разделен на водонепроницаемые отсеки (bulkheads), чтобы предотвратить затопление всего судна при повреждении одного отсека. В программном обеспечении этот паттерн изолирует ресурсы (потоки, соединения, память) для предотвращения распространения сбоев между различными частями системы.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда один сервис становится медленным или недоступным, что приводит к каскадным сбоям во всей системе. Например:

1. **Сценарий**: Сервис платежей становится медленным из-за высокой нагрузки
2. **Требование**: Нужно изолировать этот сервис, чтобы он не влиял на другие части системы
3. **Проблема**: Медленный сервис блокирует все потоки в пуле, вызывая сбои в других сервисах
4. **Результат**: Вся система становится недоступной из-за одного проблемного компонента

## Детальный анализ проблемы

### Сценарий 1: Общий пул потоков

```java
@Service
public class OrderService {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Async("commonThreadPool") // ПРОБЛЕМА ЗДЕСЬ!
    public CompletableFuture<Order> createOrder(OrderRequest request) {
        // 1. Проверяем наличие товара
        inventoryService.checkAvailability(request.getItems());
        
        // 2. Обрабатываем платеж (может быть медленным)
        PaymentResult payment = paymentService.processPayment(request.getPayment());
        
        // 3. Отправляем уведомление
        notificationService.sendNotification(request.getCustomerId());
        
        return CompletableFuture.completedFuture(new Order(request));
    }
}
```

**Проблемы:**
- Если `PaymentService` медленный, он блокирует все потоки в пуле
- Другие операции (инвентарь, уведомления) не могут выполняться
- Каскадный сбой распространяется на всю систему

### Сценарий 2: Общие соединения с базой данных

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Transactional
    public UserProfile getUserProfile(Long userId) {
        // 1. Получаем данные пользователя
        User user = userRepository.findById(userId).orElseThrow();
        
        // 2. Получаем платежную информацию (может быть медленной)
        List<Payment> payments = paymentRepository.findByUserId(userId); // ПРОБЛЕМА ЗДЕСЬ!
        
        // 3. Получаем заказы
        List<Order> orders = orderRepository.findByUserId(userId);
        
        return new UserProfile(user, payments, orders);
    }
}
```

**Проблемы:**
- Медленный запрос к платежам блокирует соединения с БД
- Другие операции не могут получить соединение
- Вся система становится медленной

### Сценарий 3: Отсутствие изоляции ресурсов

```java
@RestController
public class ApiController {
    
    @Autowired
    private ExternalApiClient externalApiClient;
    
    @GetMapping("/api/data")
    public ResponseEntity<Data> getData() {
        // ПРОБЛЕМА ЗДЕСЬ! - нет изоляции ресурсов
        return externalApiClient.fetchData()
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @GetMapping("/api/health")
    public ResponseEntity<String> health() {
        // Этот endpoint тоже может быть заблокирован
        return ResponseEntity.ok("OK");
    }
}
```

**Проблемы:**
- Медленный внешний API блокирует все запросы
- Health check не может выполниться
- Система выглядит недоступной

### Корень проблемы

Основная проблема заключается в том, что **ресурсы не изолированы между различными компонентами системы**. Это приводит к:

1. **Каскадным сбоям** - проблема в одном компоненте влияет на всю систему
2. **Блокировке ресурсов** - медленные операции занимают все доступные ресурсы
3. **Отсутствию отказоустойчивости** - система не может работать частично
4. **Сложности масштабирования** - невозможно масштабировать отдельные компоненты

### Как работает паттерн

1. **Разделение ресурсов**: Создание отдельных пулов потоков для разных типов операций
2. **Изоляция соединений**: Разделение соединений с внешними сервисами и базами данных
3. **Ограничение ресурсов**: Установка лимитов на использование ресурсов
4. **Graceful degradation**: Продолжение работы системы при сбоях отдельных компонентов

## Детальный принцип работы

### Шаг 1: Создание изолированных пулов потоков

```java
@Configuration
public class BulkheadConfig {
    
    @Bean("paymentThreadPool")
    public Executor paymentThreadPool() {
        return new ThreadPoolTaskExecutor() {{
            setCorePoolSize(5);
            setMaxPoolSize(10);
            setQueueCapacity(25);
            setThreadNamePrefix("payment-");
            setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        }};
    }
    
    @Bean("inventoryThreadPool")
    public Executor inventoryThreadPool() {
        return new ThreadPoolTaskExecutor() {{
            setCorePoolSize(3);
            setMaxPoolSize(5);
            setQueueCapacity(10);
            setThreadNamePrefix("inventory-");
            setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        }};
    }
    
    @Bean("notificationThreadPool")
    public Executor notificationThreadPool() {
        return new ThreadPoolTaskExecutor() {{
            setCorePoolSize(2);
            setMaxPoolSize(4);
            setQueueCapacity(5);
            setThreadNamePrefix("notification-");
            setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        }};
    }
}
```

**Ключевые моменты:**
- Каждый тип операции имеет свой пул потоков
- Разные лимиты для разных типов операций
- CallerRunsPolicy предотвращает отбрасывание задач

### Шаг 2: Изолированные сервисы

```java
@Service
public class BulkheadOrderService {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Async("paymentThreadPool")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                return paymentService.processPayment(request);
            } catch (Exception e) {
                log.error("Payment processing failed", e);
                throw new PaymentProcessingException("Payment failed", e);
            }
        }, paymentThreadPool);
    }
    
    @Async("inventoryThreadPool")
    public CompletableFuture<InventoryResult> checkInventory(List<String> items) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                return inventoryService.checkAvailability(items);
            } catch (Exception e) {
                log.error("Inventory check failed", e);
                throw new InventoryException("Inventory check failed", e);
            }
        }, inventoryThreadPool);
    }
    
    @Async("notificationThreadPool")
    public CompletableFuture<Void> sendNotification(String customerId) {
        return CompletableFuture.runAsync(() -> {
            try {
                notificationService.sendNotification(customerId);
            } catch (Exception e) {
                log.error("Notification sending failed", e);
                // Не прерываем основной поток из-за ошибки уведомления
            }
        }, notificationThreadPool);
    }
}
```

### Шаг 3: Изоляция соединений с базой данных

```java
@Configuration
public class DatabaseBulkheadConfig {
    
    @Bean("userDataSource")
    @ConfigurationProperties("spring.datasource.user")
    public DataSource userDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://localhost:3306/user_db")
            .username("user")
            .password("password")
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .build();
    }
    
    @Bean("paymentDataSource")
    @ConfigurationProperties("spring.datasource.payment")
    public DataSource paymentDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://localhost:3306/payment_db")
            .username("payment")
            .password("password")
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .build();
    }
    
    @Bean("orderDataSource")
    @ConfigurationProperties("spring.datasource.order")
    public DataSource orderDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://localhost:3306/order_db")
            .username("order")
            .password("password")
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .build();
    }
}
```

### Шаг 4: Circuit Breaker для внешних API

```java
@Service
public class BulkheadExternalApiClient {
    
    private final CircuitBreaker paymentCircuitBreaker;
    private final CircuitBreaker inventoryCircuitBreaker;
    
    public BulkheadExternalApiClient() {
        CircuitBreakerConfig paymentConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .ringBufferSizeInHalfOpenState(2)
            .ringBufferSizeInClosedState(100)
            .build();
        
        CircuitBreakerConfig inventoryConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(30)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .ringBufferSizeInHalfOpenState(1)
            .ringBufferSizeInClosedState(50)
            .build();
        
        this.paymentCircuitBreaker = CircuitBreaker.of("payment", paymentConfig);
        this.inventoryCircuitBreaker = CircuitBreaker.of("inventory", inventoryConfig);
    }
    
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            return paymentCircuitBreaker.executeSupplier(() -> {
                // Вызов внешнего API платежей
                return externalPaymentApi.process(request);
            });
        });
    }
    
    public CompletableFuture<InventoryResult> checkInventory(List<String> items) {
        return CompletableFuture.supplyAsync(() -> {
            return inventoryCircuitBreaker.executeSupplier(() -> {
                // Вызов внешнего API инвентаря
                return externalInventoryApi.check(items);
            });
        });
    }
}
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Order Service │    │ Payment Thread  │    │ Payment Service │
│                 │    │     Pool        │    │                 │
│ 1. Process      │───▶│ 2. Isolated     │───▶│ 3. External API │
│    Payment      │    │    Execution    │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Inventory Thread│    │ Notification    │    │ Database        │
│     Pool        │    │ Thread Pool     │    │ Connections     │
│                 │    │                 │    │                 │
│ Isolated        │    │ Isolated        │    │ Isolated        │
│ Execution       │    │ Execution       │    │ Resources       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Ключевые преимущества

1. **Изоляция сбоев**: Проблемы в одном компоненте не влияют на другие
2. **Улучшенная производительность**: Каждый компонент может работать независимо
3. **Graceful degradation**: Система продолжает работать при сбоях отдельных частей
4. **Масштабируемость**: Возможность масштабировать отдельные компоненты
5. **Управляемость ресурсов**: Контроль над использованием ресурсов

### Конфигурация пулов потоков

Размеры пулов потоков зависят от:

- **Критичности операции** - платежи требуют больше ресурсов
- **Времени выполнения** - медленные операции нуждаются в большем пуле
- **Частоты вызовов** - часто вызываемые операции требуют больше потоков
- **Доступных ресурсов** - ограничения сервера

**Пример конфигурации:**
```properties
# application.properties
bulkhead.payment.core-pool-size=5
bulkhead.payment.max-pool-size=10
bulkhead.payment.queue-capacity=25

bulkhead.inventory.core-pool-size=3
bulkhead.inventory.max-pool-size=5
bulkhead.inventory.queue-capacity=10

bulkhead.notification.core-pool-size=2
bulkhead.notification.max-pool-size=4
bulkhead.notification.queue-capacity=5
```

**Альтернативные стратегии изоляции:**
- **Процессная изоляция** - разные процессы для разных компонентов
- **Контейнерная изоляция** - Docker контейнеры для каждого компонента
- **Сетевая изоляция** - отдельные сети для разных сервисов
- **Ресурсная изоляция** - ограничения CPU и памяти

### Ключевые принципы

- **Изоляция ресурсов**: Каждый компонент имеет свои ресурсы
- **Ограничение влияния**: Сбои не распространяются между компонентами
- **Graceful degradation**: Система продолжает работать при частичных сбоях
- **Масштабируемость**: Возможность независимого масштабирования компонентов

### Когда использовать

Bulkhead Pattern подходит для случаев, когда:
- Система состоит из множества компонентов с разными требованиями к ресурсам
- Требуется изоляция сбоев между компонентами
- Нужна возможность graceful degradation
- Важна независимость компонентов друг от друга

## Преимущества и недостатки

### Преимущества

#### 1. **Изоляция сбоев**
- Проблемы в одном компоненте не влияют на другие части системы
- Каскадные сбои предотвращаются на уровне архитектуры
- Система остается стабильной даже при сбоях отдельных компонентов

#### 2. **Улучшенная производительность**
- Каждый компонент может работать с оптимальными для него ресурсами
- Нет конкуренции за ресурсы между разными типами операций
- Возможность настройки производительности для каждого компонента

#### 3. **Graceful degradation**
- Система продолжает работать при сбоях отдельных компонентов
- Критичные функции остаются доступными
- Пользователи получают частичную функциональность вместо полного отказа

#### 4. **Масштабируемость**
- Возможность независимого масштабирования компонентов
- Горизонтальное масштабирование отдельных частей системы
- Оптимизация ресурсов под конкретные потребности

#### 5. **Управляемость ресурсов**
- Точный контроль над использованием ресурсов
- Предотвращение исчерпания ресурсов одним компонентом
- Возможность мониторинга и оптимизации

#### 6. **Простота отладки**
- Легче локализовать проблемы в конкретном компоненте
- Изолированные логи и метрики
- Возможность независимого тестирования компонентов

### Недостатки

#### 1. **Увеличение сложности**
- Требуется дополнительная конфигурация пулов потоков
- Сложность управления множественными ресурсами
- Необходимость мониторинга всех изолированных компонентов

#### 2. **Неэффективное использование ресурсов**
- Возможное недоиспользование ресурсов в некоторых пулах
- Сложность балансировки нагрузки между пулами
- Риск создания слишком много пулов

#### 3. **Сложность конфигурации**
- Требуется тщательная настройка размеров пулов
- Необходимость мониторинга производительности каждого пула
- Сложность определения оптимальных параметров

#### 4. **Потенциальные deadlock'и**
- Возможность взаимных блокировок между пулами
- Сложность отладки проблем с синхронизацией
- Риск создания циклических зависимостей

#### 5. **Увеличение потребления памяти**
- Каждый пул потоков потребляет дополнительную память
- Накладные расходы на управление пулами
- Возможное дублирование ресурсов

#### 6. **Сложность мониторинга**
- Необходимость мониторинга множества компонентов
- Сложность анализа производительности всей системы
- Риск пропуска проблем в отдельных пулах

### Сравнение с альтернативами

| Аспект | Bulkhead Pattern | Circuit Breaker | Retry Pattern |
|--------|------------------|-----------------|---------------|
| **Сложность** | Средняя | Низкая | Низкая |
| **Производительность** | Высокая | Средняя | Низкая |
| **Надежность** | Высокая | Высокая | Средняя |
| **Масштабируемость** | Высокая | Средняя | Низкая |
| **Простота отладки** | Средняя | Высокая | Высокая |

## Тестирование

Интеграционные тесты показывают, как Bulkhead Pattern изолирует ресурсы и предотвращает каскадные сбои в системе.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest
@Transactional
class BulkheadIntegrationTest {
    
    @Autowired
    private BulkheadOrderService orderService;
    
    @Autowired
    private TestPaymentService testPaymentService;
    
    @Autowired
    private TestInventoryService testInventoryService;
    
    @Test
    void shouldProcessOrderWithIsolatedResources() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1"));
        
        // When
        CompletableFuture<Order> orderFuture = orderService.createOrder(request);
        
        // Then - проверяем, что заказ создан
        Order order = orderFuture.get(5, TimeUnit.SECONDS);
        assertThat(order).isNotNull();
        assertThat(order.getStatus()).isEqualTo("CREATED");
        
        // And - проверяем, что все операции выполнены
        assertThat(testPaymentService.getProcessedPayments()).hasSize(1);
        assertThat(testInventoryService.getCheckedItems()).hasSize(1);
    }
    
    @Test
    void shouldHandleSlowPaymentService() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1"));
        testPaymentService.setDelay(5000); // 5 секунд задержки
        
        // When
        CompletableFuture<Order> orderFuture = orderService.createOrder(request);
        
        // Then - заказ должен быть создан, несмотря на медленный платеж
        Order order = orderFuture.get(10, TimeUnit.SECONDS);
        assertThat(order).isNotNull();
        
        // And - другие операции не должны быть заблокированы
        assertThat(testInventoryService.getCheckedItems()).hasSize(1);
    }
    
    @Test
    void shouldHandlePaymentServiceFailure() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1"));
        testPaymentService.setShouldFail(true);
        
        // When & Then
        assertThatThrownBy(() -> {
            orderService.createOrder(request).get(5, TimeUnit.SECONDS);
        }).isInstanceOf(ExecutionException.class)
          .hasCauseInstanceOf(PaymentProcessingException.class);
        
        // But - другие операции должны быть выполнены
        assertThat(testInventoryService.getCheckedItems()).hasSize(1);
    }
}
```

#### TestPaymentService для тестирования

```java
@Component
@Primary
public class TestPaymentService implements PaymentService {
    
    private final List<PaymentRequest> processedPayments = new ArrayList<>();
    private boolean shouldFail = false;
    private long delay = 0;
    
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        if (delay > 0) {
            try {
                Thread.sleep(delay);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        if (shouldFail) {
            throw new PaymentProcessingException("Test failure");
        }
        
        processedPayments.add(request);
        return new PaymentResult("SUCCESS", "transaction123");
    }
    
    public List<PaymentRequest> getProcessedPayments() {
        return new ArrayList<>(processedPayments);
    }
    
    public void clear() {
        processedPayments.clear();
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
    
    public void setDelay(long delay) {
        this.delay = delay;
    }
}
``` 