# Circuit Breaker Pattern

## Обзор паттерна

**Circuit Breaker** - это паттерн проектирования для повышения отказоустойчивости и стабильности микросервисных систем путем предотвращения каскадных сбоев.

### Что это такое?

Circuit Breaker - это техника, которая предотвращает каскадные сбои в распределенных системах, временно блокируя вызовы к недоступным или медленным сервисам. Паттерн работает по аналогии с электрическим предохранителем: когда система обнаруживает слишком много ошибок, она "разрывает цепь" и перестает делать вызовы к проблемному сервису, позволяя ему восстановиться.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда один сервис становится недоступным или медленно отвечает, что приводит к каскадным сбоям во всей системе. Например:

1. **Сценарий**: Пользователь делает заказ, который требует проверки кредитного лимита через внешний сервис
2. **Требование**: Нужно быстро обработать заказ и показать результат пользователю
3. **Проблема**: Кредитный сервис медленно отвечает или недоступен, что блокирует все заказы
4. **Результат**: Система заказов становится недоступной, пользователи уходят, бизнес теряет деньги

## Детальный анализ проблемы

### Сценарий 1: Прямые вызовы без защиты

```java
@Service
public class OrderService {
    
    @Autowired
    private CreditService creditService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Проверяем кредитный лимит (ПРОБЛЕМА ЗДЕСЬ!)
        CreditLimitResponse creditLimit = creditService.checkLimit(request.getCustomerId());
        
        if (!creditLimit.isApproved()) {
            throw new InsufficientCreditException("Insufficient credit limit");
        }
        
        // 2. Создаем заказ
        Order order = orderRepository.save(new Order(request));
        
        return order;
    }
}
```

**Проблемы:**
- Если `CreditService` медленный, все заказы будут ждать
- Если `CreditService` недоступен, все заказы упадут с ошибкой
- Нет механизма fallback для обработки ошибок
- Каскадные сбои распространяются по всей системе

### Сценарий 2: Простая обработка исключений

```java
@Service
public class OrderService {
    
    @Autowired
    private CreditService creditService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        try {
            // 1. Проверяем кредитный лимит
            CreditLimitResponse creditLimit = creditService.checkLimit(request.getCustomerId());
            
            if (!creditLimit.isApproved()) {
                throw new InsufficientCreditException("Insufficient credit limit");
            }
            
            // 2. Создаем заказ
            Order order = orderRepository.save(new Order(request));
            return order;
            
        } catch (Exception e) {
            // 3. Fallback - создаем заказ без проверки (ПРОБЛЕМА ЗДЕСЬ!)
            log.warn("Credit service unavailable, creating order without check");
            return orderRepository.save(new Order(request));
        }
    }
}
```

**Проблемы:**
- Каждый запрос все равно пытается вызвать проблемный сервис
- Нет накопления информации о состоянии сервиса
- Система не учится на ошибках
- Потенциальные проблемы с безопасностью (пропуск проверок)

### Сценарий 3: Таймауты без Circuit Breaker

```java
@Service
public class OrderService {
    
    @Autowired
    private CreditService creditService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        try {
            // 1. Устанавливаем таймаут
            CompletableFuture<CreditLimitResponse> future = 
                CompletableFuture.supplyAsync(() -> 
                    creditService.checkLimit(request.getCustomerId())
                );
            
            // 2. Ждем с таймаутом (ПРОБЛЕМА ЗДЕСЬ!)
            CreditLimitResponse creditLimit = future.get(5, TimeUnit.SECONDS);
            
            if (!creditLimit.isApproved()) {
                throw new InsufficientCreditException("Insufficient credit limit");
            }
            
            return orderRepository.save(new Order(request));
            
        } catch (TimeoutException e) {
            // 3. Fallback при таймауте
            log.warn("Credit service timeout, creating order without check");
            return orderRepository.save(new Order(request));
        }
    }
}
```

**Проблемы:**
- Каждый запрос все равно ждет таймаут
- Нет памяти о предыдущих проблемах
- Ресурсы тратятся на ожидание
- Пользователи получают медленные ответы

### Корень проблемы

Основная проблема заключается в том, что **система не имеет механизма для предотвращения повторных вызовов к проблемным сервисам**. Это приводит к:

1. **Каскадным сбоям** - проблемы одного сервиса распространяются на всю систему
2. **Потере ресурсов** - время и память тратятся на ожидание недоступных сервисов
3. **Плохому UX** - пользователи получают медленные ответы или ошибки
4. **Отсутствию самовосстановления** - система не дает проблемным сервисам время на восстановление

### Как работает паттерн

1. **Закрытое состояние**: Circuit Breaker пропускает все вызовы и отслеживает ошибки
2. **Открытое состояние**: При превышении порога ошибок Circuit Breaker блокирует вызовы
3. **Полуоткрытое состояние**: После таймаута Circuit Breaker пропускает один тестовый вызов
4. **Автоматическое восстановление**: При успешном тестовом вызове Circuit Breaker возвращается в закрытое состояние

## Детальный принцип работы

### Шаг 1: Реализация Circuit Breaker

```java
@Component
public class CircuitBreaker {
    
    private CircuitState state = CircuitState.CLOSED;
    private int failureCount = 0;
    private int successCount = 0;
    private LocalDateTime lastFailureTime;
    
    @Value("${circuit-breaker.failure-threshold:5}")
    private int failureThreshold;
    
    @Value("${circuit-breaker.timeout-seconds:60}")
    private int timeoutSeconds;
    
    @Value("${circuit-breaker.success-threshold:2}")
    private int successThreshold;
    
    public <T> T execute(Supplier<T> supplier) {
        switch (state) {
            case CLOSED:
                return executeInClosedState(supplier);
            case OPEN:
                return executeInOpenState(supplier);
            case HALF_OPEN:
                return executeInHalfOpenState(supplier);
            default:
                throw new IllegalStateException("Unknown circuit state");
        }
    }
    
    private <T> T executeInClosedState(Supplier<T> supplier) {
        try {
            T result = supplier.get();
            resetFailureCount();
            return result;
        } catch (Exception e) {
            handleFailure();
            throw e;
        }
    }
    
    private <T> T executeInOpenState(Supplier<T> supplier) {
        if (shouldAttemptReset()) {
            state = CircuitState.HALF_OPEN;
            return executeInHalfOpenState(supplier);
        }
        
        throw new CircuitBreakerOpenException("Circuit breaker is OPEN");
    }
    
    private <T> T executeInHalfOpenState(Supplier<T> supplier) {
        try {
            T result = supplier.get();
            handleSuccess();
            return result;
        } catch (Exception e) {
            handleFailure();
            throw e;
        }
    }
    
    private void handleFailure() {
        failureCount++;
        lastFailureTime = LocalDateTime.now();
        
        if (failureCount >= failureThreshold) {
            state = CircuitState.OPEN;
            log.warn("Circuit breaker opened after {} failures", failureCount);
        }
    }
    
    private void handleSuccess() {
        successCount++;
        
        if (successCount >= successThreshold) {
            state = CircuitState.CLOSED;
            resetCounters();
            log.info("Circuit breaker closed after {} successes", successCount);
        }
    }
    
    private boolean shouldAttemptReset() {
        return lastFailureTime != null && 
               LocalDateTime.now().isAfter(lastFailureTime.plusSeconds(timeoutSeconds));
    }
    
    private void resetFailureCount() {
        failureCount = 0;
        successCount = 0;
    }
    
    private void resetCounters() {
        failureCount = 0;
        successCount = 0;
        lastFailureTime = null;
    }
}
```

**Ключевые моменты:**
- Три состояния: CLOSED, OPEN, HALF_OPEN
- Автоматический переход между состояниями
- Настраиваемые пороги для ошибок и успехов
- Логирование изменений состояния

### Шаг 2: Интеграция с сервисом

```java
@Service
public class OrderService {
    
    @Autowired
    private CreditService creditService;
    
    @Autowired
    private CircuitBreaker circuitBreaker;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        try {
            // 1. Вызываем через Circuit Breaker
            CreditLimitResponse creditLimit = circuitBreaker.execute(() ->
                creditService.checkLimit(request.getCustomerId())
            );
            
            if (!creditLimit.isApproved()) {
                throw new InsufficientCreditException("Insufficient credit limit");
            }
            
            // 2. Создаем заказ
            return orderRepository.save(new Order(request));
            
        } catch (CircuitBreakerOpenException e) {
            // 3. Fallback при открытом Circuit Breaker
            log.warn("Circuit breaker is open, creating order without credit check");
            return createOrderWithoutCreditCheck(request);
        } catch (Exception e) {
            // 4. Обработка других ошибок
            log.error("Error creating order", e);
            throw new OrderCreationException("Failed to create order", e);
        }
    }
    
    private Order createOrderWithoutCreditCheck(OrderRequest request) {
        // Бизнес-логика для создания заказа без проверки кредита
        Order order = new Order(request);
        order.setCreditCheckSkipped(true);
        return orderRepository.save(order);
    }
}
```

### Шаг 3: Конфигурация и мониторинг

```java
@Component
public class CircuitBreakerMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Map<String, CircuitBreaker> circuitBreakers = new ConcurrentHashMap<>();
    
    public CircuitBreakerMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void registerCircuitBreaker(String name, CircuitBreaker circuitBreaker) {
        circuitBreakers.put(name, circuitBreaker);
        
        // Регистрируем метрики
        Gauge.builder("circuit_breaker.state", circuitBreaker, this::getStateValue)
            .tag("name", name)
            .register(meterRegistry);
        
        Counter.builder("circuit_breaker.failures")
            .tag("name", name)
            .register(meterRegistry);
    }
    
    private double getStateValue(CircuitBreaker circuitBreaker) {
        switch (circuitBreaker.getState()) {
            case CLOSED: return 0.0;
            case HALF_OPEN: return 0.5;
            case OPEN: return 1.0;
            default: return -1.0;
        }
    }
}
```

### Шаг 4: Расширенная конфигурация

```properties
# application.properties
circuit-breaker.failure-threshold=5
circuit-breaker.timeout-seconds=60
circuit-breaker.success-threshold=2
circuit-breaker.enable-monitoring=true
circuit-breaker.log-state-changes=true
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Order Service  │    │ Circuit Breaker │    │ Credit Service  │
│                 │    │                 │    │                 │
│ 1. Create Order │───▶│ 2. Check State  │───▶│ 3. Check Limit  │
│                 │    │                 │    │                 │
│ 4. Return Order │◀───│ 5. Handle Result│◀───│ 4. Return Result│
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   Fallback     │
                       │   Strategy      │
                       └─────────────────┘
```

### Ключевые преимущества

1. **Предотвращение каскадных сбоев**: Проблемы одного сервиса не распространяются на всю систему
2. **Быстрое восстановление**: Система автоматически возвращается к нормальной работе
3. **Улучшение UX**: Пользователи получают быстрые ответы даже при проблемах
4. **Экономия ресурсов**: Не тратятся ресурсы на ожидание недоступных сервисов
5. **Мониторинг и отладка**: Легко отслеживать состояние сервисов

### Конфигурация Circuit Breaker

Настройка Circuit Breaker зависит от характеристик системы:

- **failureThreshold** - количество ошибок для открытия Circuit Breaker
- **timeoutSeconds** - время ожидания перед попыткой восстановления
- **successThreshold** - количество успешных вызовов для закрытия Circuit Breaker

**Пример конфигурации для разных сервисов:**
```properties
# Критичный сервис - более консервативные настройки
credit-service.circuit-breaker.failure-threshold=3
credit-service.circuit-breaker.timeout-seconds=30

# Некритичный сервис - более агрессивные настройки
notification-service.circuit-breaker.failure-threshold=10
notification-service.circuit-breaker.timeout-seconds=120
```

**Альтернативные стратегии:**
- **Адаптивные пороги**: Изменение порогов на основе нагрузки
- **Разные таймауты**: Для разных типов ошибок
- **Graceful degradation**: Постепенное снижение функциональности
- **Health checks**: Активные проверки состояния сервисов

### Ключевые принципы

- **Fail Fast**: Быстрое обнаружение и реакция на проблемы
- **Graceful Degradation**: Постепенное снижение функциональности
- **Self Healing**: Автоматическое восстановление после проблем
- **Monitoring**: Постоянное отслеживание состояния системы

### Когда использовать

Circuit Breaker подходит для случаев, когда:
- Система взаимодействует с внешними сервисами
- Требуется высокая отказоустойчивость
- Важна скорость ответа пользователям
- Есть риск каскадных сбоев
- Нужен механизм автоматического восстановления

## Преимущества и недостатки

### Преимущества

#### 1. **Предотвращение каскадных сбоев**
- Проблемы одного сервиса не распространяются на всю систему
- Изоляция сбоев на уровне отдельных компонентов
- Сохранение работоспособности критически важных функций

#### 2. **Быстрое восстановление**
- Автоматическое возвращение к нормальной работе
- Минимальное время простоя системы
- Снижение времени восстановления после сбоев

#### 3. **Улучшение пользовательского опыта**
- Пользователи получают быстрые ответы даже при проблемах
- Graceful degradation вместо полных сбоев
- Предсказуемое поведение системы

#### 4. **Экономия ресурсов**
- Не тратятся ресурсы на ожидание недоступных сервисов
- Снижение нагрузки на проблемные сервисы
- Более эффективное использование ресурсов

#### 5. **Мониторинг и отладка**
- Легко отслеживать состояние сервисов
- Прозрачность работы системы
- Возможность проактивного реагирования на проблемы

#### 6. **Гибкость конфигурации**
- Настраиваемые пороги для разных сервисов
- Возможность адаптации под конкретные требования
- Различные стратегии fallback

### Недостатки

#### 1. **Дополнительная сложность**
- Требуется дополнительная логика в коде
- Увеличение количества компонентов системы
- Сложность отладки при проблемах с Circuit Breaker

#### 2. **Потенциальные ложные срабатывания**
- Circuit Breaker может открыться при временных проблемах
- Риск блокировки вызовов к восстановившемуся сервису
- Необходимость тщательной настройки порогов

#### 3. **Сложность настройки**
- Требуется опыт для правильной настройки параметров
- Разные настройки для разных сервисов
- Необходимость мониторинга и корректировки настроек

#### 4. **Потенциальные проблемы с состоянием**
- Состояние Circuit Breaker может рассинхронизироваться в кластере
- Проблемы при масштабировании с состоянием
- Необходимость централизованного управления состоянием

#### 5. **Ограничения fallback стратегий**
- Не все функции можно реализовать в fallback
- Потенциальные проблемы с консистентностью данных
- Риск потери важной функциональности

#### 6. **Сложность тестирования**
- Трудно тестировать все сценарии работы Circuit Breaker
- Необходимость симуляции различных состояний
- Сложность интеграционного тестирования

### Сравнение с альтернативами

| Аспект | Circuit Breaker | Retry Pattern | Timeout Pattern |
|--------|-----------------|---------------|-----------------|
| **Сложность** | Средняя | Низкая | Низкая |
| **Производительность** | Высокая | Низкая | Средняя |
| **Надежность** | Высокая | Средняя | Низкая |
| **Масштабируемость** | Высокая | Низкая | Средняя |
| **Простота отладки** | Средняя | Высокая | Высокая |

## Тестирование

Интеграционные тесты показывают полный цикл работы Circuit Breaker паттерна и помогают убедиться в правильности работы всех состояний.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest
@Transactional
class CircuitBreakerIntegrationTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private CircuitBreaker circuitBreaker;
    
    @Autowired
    private TestCreditService testCreditService;
    
    @Test
    void shouldProcessCompleteCircuitBreakerFlow() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1"));
        
        // When - нормальный вызов
        Order order = orderService.createOrder(request);
        
        // Then - заказ создан с проверкой кредита
        assertThat(order.isCreditCheckSkipped()).isFalse();
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitState.CLOSED);
        
        // When - симулируем сбои
        testCreditService.setShouldFail(true);
        
        for (int i = 0; i < 5; i++) {
            try {
                orderService.createOrder(request);
            } catch (Exception e) {
                // Ожидаем исключения
            }
        }
        
        // Then - Circuit Breaker открыт
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitState.OPEN);
        
        // When - пытаемся создать заказ при открытом Circuit Breaker
        Order fallbackOrder = orderService.createOrder(request);
        
        // Then - заказ создан без проверки кредита
        assertThat(fallbackOrder.isCreditCheckSkipped()).isTrue();
        
        // When - исправляем сервис и ждем восстановления
        testCreditService.setShouldFail(false);
        Thread.sleep(61000); // Ждем timeout
        
        // Then - Circuit Breaker в полуоткрытом состоянии
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitState.HALF_OPEN);
        
        // When - успешный вызов
        Order recoveredOrder = orderService.createOrder(request);
        
        // Then - Circuit Breaker закрыт
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitState.CLOSED);
        assertThat(recoveredOrder.isCreditCheckSkipped()).isFalse();
    }
    
    @Test
    void shouldHandleCircuitBreakerOpenException() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1"));
        testCreditService.setShouldFail(true);
        
        // Открываем Circuit Breaker
        for (int i = 0; i < 5; i++) {
            try {
                orderService.createOrder(request);
            } catch (Exception e) {
                // Ожидаем исключения
            }
        }
        
        // When - пытаемся создать заказ при открытом Circuit Breaker
        Order order = orderService.createOrder(request);
        
        // Then - заказ создан через fallback
        assertThat(order.isCreditCheckSkipped()).isTrue();
        assertThat(order.getStatus()).isEqualTo("CREATED");
    }
}
```

#### TestCreditService для тестирования

```java
@Component
@Primary
public class TestCreditService implements CreditService {
    
    private boolean shouldFail = false;
    private int callCount = 0;
    
    @Override
    public CreditLimitResponse checkLimit(String customerId) {
        callCount++;
        
        if (shouldFail) {
            throw new RuntimeException("Test failure");
        }
        
        return new CreditLimitResponse(true, 10000.0);
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
    
    public int getCallCount() {
        return callCount;
    }
    
    public void reset() {
        callCount = 0;
        shouldFail = false;
    }
}
``` 