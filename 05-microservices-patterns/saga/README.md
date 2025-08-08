# Saga Pattern

## Обзор паттерна

**Saga Pattern** - это паттерн проектирования для управления распределенными транзакциями в микросервисной архитектуре, обеспечивающий консистентность данных через последовательность локальных транзакций и компенсирующих действий.

### Что это такое?

Saga Pattern - это техника координации длительных бизнес-процессов, состоящих из множества локальных транзакций в разных микросервисах. Вместо использования глобальных транзакций, Saga разбивает сложную операцию на последовательность локальных транзакций, где каждая транзакция может быть отменена с помощью компенсирующего действия.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда бизнес-операция требует взаимодействия нескольких сервисов. Например:

1. **Сценарий**: Пользователь создает заказ, который требует проверки товара, резервирования, списания средств и отправки уведомления
2. **Требование**: Все операции должны быть выполнены атомарно или отменены полностью
3. **Проблема**: Традиционные ACID транзакции не работают в распределенной среде
4. **Результат**: Система должна обеспечить консистентность данных через компенсирующие действия

## Детальный анализ проблемы

### Сценарий 1: Попытка использовать глобальные транзакции

```java
@Service
public class OrderService {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Создаем заказ
        Order order = orderRepository.save(new Order(request));
        
        // 2. Проверяем и резервируем товар (ПРОБЛЕМА ЗДЕСЬ!)
        inventoryService.reserveItems(order.getItems());
        
        // 3. Списываем средства (ПРОБЛЕМА ЗДЕСЬ!)
        paymentService.chargeCustomer(order.getCustomerId(), order.getTotal());
        
        // 4. Отправляем уведомление (ПРОБЛЕМА ЗДЕСЬ!)
        notificationService.sendOrderConfirmation(order);
        
        return order;
    }
}
```

**Проблемы:**
- Если один из сервисов недоступен, вся транзакция откатится
- Нет возможности частичного отката
- Блокировка ресурсов на длительное время
- Не масштабируется в распределенной среде

### Сценарий 2: Асинхронная обработка без координации

```java
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Создаем заказ
        Order order = orderRepository.save(new Order(request));
        
        // 2. Отправляем события в очередь
        eventPublisher.publish("order.created", order);
        
        return order;
    }
}

@Component
public class OrderEventHandler {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Каждый сервис обрабатывает независимо
        inventoryService.reserveItems(event.getItems());
        paymentService.chargeCustomer(event.getCustomerId(), event.getTotal());
        notificationService.sendOrderConfirmation(event.getOrder());
    }
}
```

**Проблемы:**
- Нет гарантии, что все сервисы обработают событие
- Сложность отката при ошибках
- Нет видимости состояния всего процесса
- Трудно отследить, где именно произошла ошибка

### Сценарий 3: Синхронные вызовы без отката

```java
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        try {
            // 1. Создаем заказ
            Order order = orderRepository.save(new Order(request));
            
            // 2. Последовательно вызываем сервисы
            inventoryService.reserveItems(order.getItems());
            paymentService.chargeCustomer(order.getCustomerId(), order.getTotal());
            notificationService.sendOrderConfirmation(order);
            
            return order;
            
        } catch (Exception e) {
            // ПРОБЛЕМА ЗДЕСЬ! Нет отката уже выполненных операций
            log.error("Error creating order", e);
            throw e;
        }
    }
}
```

**Проблемы:**
- Если ошибка произойдет на третьем шаге, первые два уже выполнены
- Нет механизма отката выполненных операций
- Данные остаются в несогласованном состоянии
- Сложность восстановления после сбоев

### Корень проблемы

Основная проблема заключается в том, что **распределенные транзакции не могут обеспечить ACID свойства в микросервисной архитектуре**. Это приводит к:

1. **Несогласованности данных** - частично выполненные операции оставляют систему в неопределенном состоянии
2. **Сложности отката** - нет простого способа отменить уже выполненные операции
3. **Блокировкам** - длительные транзакции блокируют ресурсы
4. **Отсутствию видимости** - трудно понять, на каком этапе произошла ошибка

### Как работает паттерн

1. **Разбиение на локальные транзакции**: Сложная операция разбивается на последовательность локальных транзакций
2. **Компенсирующие действия**: Для каждой транзакции определяется действие отката
3. **Координация процесса**: Saga Orchestrator или Choreography управляет последовательностью
4. **Обработка ошибок**: При ошибке выполняются компенсирующие действия в обратном порядке

## Детальный принцип работы

### Шаг 1: Определение Saga и локальных транзакций

```java
@Service
public class OrderSagaService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private SagaOrchestrator sagaOrchestrator;
    
    public Order createOrder(OrderRequest request) {
        // 1. Создаем Saga
        Saga saga = new Saga("create-order-saga");
        
        // 2. Определяем шаги Saga
        saga.addStep(new CreateOrderStep(orderRepository, request));
        saga.addStep(new ReserveInventoryStep(inventoryService, request.getItems()));
        saga.addStep(new ChargePaymentStep(paymentService, request.getCustomerId(), request.getTotal()));
        saga.addStep(new SendNotificationStep(notificationService, request));
        
        // 3. Запускаем Saga
        return sagaOrchestrator.execute(saga);
    }
}
```

**Ключевые моменты:**
- Каждый шаг представляет локальную транзакцию
- Для каждого шага определено компенсирующее действие
- Saga Orchestrator управляет выполнением

### Шаг 2: Реализация шагов Saga

```java
@Component
public class CreateOrderStep implements SagaStep<Order> {
    
    private final OrderRepository orderRepository;
    private final OrderRequest request;
    
    public CreateOrderStep(OrderRepository orderRepository, OrderRequest request) {
        this.orderRepository = orderRepository;
        this.request = request;
    }
    
    @Override
    @Transactional
    public Order execute(SagaContext context) {
        Order order = orderRepository.save(new Order(request));
        context.setData("orderId", order.getId());
        return order;
    }
    
    @Override
    @Transactional
    public void compensate(SagaContext context) {
        Long orderId = context.getData("orderId");
        orderRepository.deleteById(orderId);
    }
}

@Component
public class ReserveInventoryStep implements SagaStep<Void> {
    
    private final InventoryService inventoryService;
    private final List<String> items;
    
    public ReserveInventoryStep(InventoryService inventoryService, List<String> items) {
        this.inventoryService = inventoryService;
        this.items = items;
    }
    
    @Override
    @Transactional
    public Void execute(SagaContext context) {
        inventoryService.reserveItems(items);
        context.setData("reservedItems", items);
        return null;
    }
    
    @Override
    @Transactional
    public void compensate(SagaContext context) {
        List<String> reservedItems = context.getData("reservedItems");
        inventoryService.releaseItems(reservedItems);
    }
}
```

### Шаг 3: Saga Orchestrator

```java
@Component
public class SagaOrchestrator {
    
    @Autowired
    private SagaRepository sagaRepository;
    
    public <T> T execute(Saga saga) {
        SagaExecution execution = new SagaExecution(saga);
        sagaRepository.save(execution);
        
        try {
            // Выполняем шаги последовательно
            for (SagaStep<?> step : saga.getSteps()) {
                try {
                    step.execute(execution.getContext());
                    execution.markStepCompleted(step);
                    
                } catch (Exception e) {
                    // При ошибке выполняем компенсирующие действия
                    execution.markStepFailed(step, e);
                    compensate(execution, step);
                    throw new SagaExecutionException("Saga failed", e);
                }
            }
            
            execution.markCompleted();
            sagaRepository.save(execution);
            return (T) execution.getContext().getResult();
            
        } catch (Exception e) {
            execution.markFailed(e);
            sagaRepository.save(execution);
            throw e;
        }
    }
    
    private void compensate(SagaExecution execution, SagaStep<?> failedStep) {
        List<SagaStep<?>> completedSteps = execution.getCompletedSteps();
        
        // Выполняем компенсирующие действия в обратном порядке
        for (int i = completedSteps.size() - 1; i >= 0; i--) {
            SagaStep<?> step = completedSteps.get(i);
            try {
                step.compensate(execution.getContext());
            } catch (Exception e) {
                log.error("Compensation failed for step: {}", step.getClass().getSimpleName(), e);
            }
        }
    }
}
```

### Шаг 4: Структура базы данных для Saga

```sql
CREATE TABLE saga_executions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    saga_id VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL,
    current_step INT DEFAULT 0,
    context_data JSON,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    error_message TEXT,
    INDEX idx_saga_status (saga_id, status)
);

CREATE TABLE saga_steps (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    execution_id BIGINT NOT NULL,
    step_name VARCHAR(255) NOT NULL,
    step_order INT NOT NULL,
    status VARCHAR(50) NOT NULL,
    executed_at TIMESTAMP NULL,
    compensated_at TIMESTAMP NULL,
    error_message TEXT,
    FOREIGN KEY (execution_id) REFERENCES saga_executions(id),
    INDEX idx_execution_order (execution_id, step_order)
);
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Order Service  │    │ Saga Orchestrator│    │  Step 1: Create │
│                 │    │                 │    │      Order      │
│ 1. Create Saga  │───▶│ 2. Execute Step │───▶│ 3. Success      │
│                 │    │                 │    │                 │
│ 4. Return Order │    │ 5. Next Step    │    │ 4. Compensate   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │  Step 2: Reserve│    │  Step 3: Charge │
                       │   Inventory     │    │    Payment      │
                       │                 │    │                 │
                       │ 3. Success      │    │ 3. Success      │
                       │                 │    │                 │
                       │ 4. Compensate   │    │ 4. Compensate   │
                       └─────────────────┘    └─────────────────┘
```

### Ключевые преимущества

1. **Консистентность данных**: Обеспечивает согласованность через компенсирующие действия
2. **Масштабируемость**: Каждый сервис может масштабироваться независимо
3. **Отказоустойчивость**: Система может восстановиться после сбоев
4. **Видимость процесса**: Легко отслеживать состояние выполнения
5. **Гибкость**: Можно изменять порядок и логику шагов

### Конфигурация Saga

Настройка Saga зависит от бизнес-требований и характеристик системы:

**Пример конфигурации:**
```properties
# application.properties
saga.timeout=30000
saga.max-retries=3
saga.retry-delay=1000
saga.compensation-timeout=10000
```

**Альтернативные стратегии координации:**
- **Choreography**: Сервисы обмениваются событиями без центрального координатора
- **Orchestration**: Центральный координатор управляет процессом
- **Hybrid**: Комбинация обоих подходов
- **Event-driven**: Асинхронная обработка через события

### Choreography-based Saga (Сага на основе хореографии)

В Choreography-based Saga каждый сервис знает только о своих собственных действиях и компенсирующих операциях. Сервисы обмениваются событиями через Message Broker, и каждый сервис решает, что делать дальше, основываясь на полученных событиях.

#### Пример реализации Choreography-based Saga

```java
// Order Service - инициирует процесс
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Создаем заказ
        Order order = orderRepository.save(new Order(request));
        
        // 2. Публикуем событие - Order Service НЕ ЗНАЕТ что будет дальше!
        eventPublisher.publish("order.created", new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotal()
        ));
        
        return order;
    }
    
    // Обработчик события отката
    @EventListener
    public void handleOrderCancelled(OrderCancelledEvent event) {
        // Компенсирующее действие - удаляем заказ
        orderRepository.deleteById(event.getOrderId());
    }
}

// Inventory Service - реагирует на события
@Service
public class InventoryService {
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    // Обработчик события создания заказа
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Резервируем товары
            List<String> reservedItems = inventoryRepository.reserveItems(event.getItems());
            
            // Публикуем событие успешного резервирования
            eventPublisher.publish("inventory.reserved", new InventoryReservedEvent(
                event.getOrderId(),
                reservedItems
            ));
            
        } catch (Exception e) {
            // При ошибке публикуем событие отмены заказа
            eventPublisher.publish("order.cancelled", new OrderCancelledEvent(
                event.getOrderId(),
                "Inventory reservation failed"
            ));
        }
    }
    
    // Обработчик события отмены заказа
    @EventListener
    public void handleOrderCancelled(OrderCancelledEvent event) {
        // Компенсирующее действие - освобождаем товары
        inventoryRepository.releaseItems(event.getOrderId());
    }
}

// Payment Service - реагирует на события
@Service
public class PaymentService {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    // Обработчик события резервирования товаров
    @EventListener
    public void handleInventoryReserved(InventoryReservedEvent event) {
        try {
            // Списываем средства
            Payment payment = paymentRepository.chargeCustomer(
                event.getOrderId(),
                event.getAmount()
            );
            
            // Публикуем событие успешного списания
            eventPublisher.publish("payment.completed", new PaymentCompletedEvent(
                event.getOrderId(),
                payment.getId()
            ));
            
        } catch (Exception e) {
            // При ошибке публикуем событие отмены заказа
            eventPublisher.publish("order.cancelled", new OrderCancelledEvent(
                event.getOrderId(),
                "Payment failed"
            ));
        }
    }
    
    // Обработчик события отмены заказа
    @EventListener
    public void handleOrderCancelled(OrderCancelledEvent event) {
        // Компенсирующее действие - возвращаем средства
        paymentRepository.refundCustomer(event.getOrderId());
    }
}

// Notification Service - финальный шаг
@Service
public class NotificationService {
    
    @Autowired
    private NotificationRepository notificationRepository;
    
    // Обработчик события завершения платежа
    @EventListener
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        try {
            // Отправляем уведомление
            notificationRepository.sendOrderConfirmation(event.getOrderId());
            
            // Публикуем событие завершения процесса
            eventPublisher.publish("order.confirmed", new OrderConfirmedEvent(
                event.getOrderId()
            ));
            
        } catch (Exception e) {
            // При ошибке публикуем событие отмены заказа
            eventPublisher.publish("order.cancelled", new OrderCancelledEvent(
                event.getOrderId(),
                "Notification failed"
            ));
        }
    }
}
```

#### Архитектурная схема Choreography-based Saga

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Order Service  │    │  Message Broker │    │ Inventory Service│
│                 │    │                 │    │                 │
│ 1. Create Order │───▶│ 2. order.created│───▶│ 3. Reserve Items│
│                 │    │                 │    │                 │
│ 8. Handle Cancel│◀───│ 7. order.cancelled│◀───│ 4. Publish Event│
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │ Payment Service │    │Notification Svc │
                       │                 │    │                 │
                       │ 5. Charge Money │    │ 6. Send Notif   │
                       │                 │    │                 │
                       │ 4. Publish Event│    │ 4. Publish Event│
                       └─────────────────┘    └─────────────────┘
```

#### Ключевые особенности Choreography-based Saga:

1. **Отсутствие центрального координатора** - каждый сервис сам решает, что делать
2. **Слабая связанность** - сервисы не знают друг о друге напрямую
3. **Событийная архитектура** - все взаимодействие через события
4. **Автоматическая компенсация** - при ошибке каждый сервис выполняет свое компенсирующее действие
5. **Масштабируемость** - легко добавлять новые сервисы без изменения существующих

#### Преимущества Choreography-based Saga:

- **Меньшая связанность** между сервисами
- **Простота добавления** новых шагов
- **Отсутствие единой точки отказа** (центрального координатора)
- **Естественная асинхронность** через события

#### Недостатки Choreography-based Saga:

- **Сложность отладки** - трудно отследить весь flow
- **Сложность тестирования** - нужно мокировать множество событий
- **Потенциальные race conditions** - события могут приходить в неправильном порядке
- **Сложность мониторинга** - нет центрального места для отслеживания состояния

### Ключевые принципы

- **Локальные транзакции**: Каждый шаг выполняется в рамках локальной транзакции
- **Компенсирующие действия**: Для каждого шага определено действие отката
- **Последовательность**: Шаги выполняются в определенном порядке
- **Идемпотентность**: Компенсирующие действия должны быть идемпотентными

### Когда использовать

Saga Pattern подходит для случаев, когда:
- Бизнес-процесс состоит из множества шагов в разных сервисах
- Требуется обеспечить консистентность данных в распределенной среде
- Нужна возможность отката частично выполненных операций
- Система должна быть устойчивой к сбоям отдельных сервисов

## Преимущества и недостатки

### Преимущества

#### 1. **Консистентность данных в распределенной среде**
- Обеспечивает согласованность данных через компенсирующие действия
- Позволяет откатывать частично выполненные операции
- Гарантирует, что система остается в согласованном состоянии

#### 2. **Масштабируемость и производительность**
- Каждый сервис может масштабироваться независимо
- Нет длительных блокировок ресурсов
- Позволяет параллельное выполнение независимых шагов

#### 3. **Отказоустойчивость**
- Система может восстановиться после сбоев отдельных сервисов
- Автоматические повторные попытки при временных ошибках
- Возможность ручного вмешательства при критических сбоях

#### 4. **Видимость и мониторинг**
- Легко отслеживать состояние выполнения Saga
- Возможность анализа неудачных выполнений
- Простота отладки и диагностики проблем

#### 5. **Гибкость и адаптивность**
- Можно изменять порядок и логику шагов
- Поддержка различных стратегий координации
- Возможность добавления новых шагов без изменения существующих

#### 6. **Изоляция сервисов**
- Каждый сервис остается независимым
- Минимальная связанность между сервисами
- Простота разработки и тестирования отдельных сервисов

### Недостатки

#### 1. **Сложность реализации**
- Требуется реализация компенсирующих действий для каждого шага
- Сложность координации между сервисами
- Необходимость обработки различных сценариев ошибок

#### 2. **Сложность отладки**
- Трудно отследить причину сбоя в длинной цепочке операций
- Сложность воспроизведения проблем в тестовой среде
- Необходимость анализа логов множества сервисов

#### 3. **Потенциальные проблемы производительности**
- Последовательное выполнение может замедлить процесс
- Дополнительные накладные расходы на координацию
- Возможные блокировки при высокой нагрузке

#### 4. **Сложность тестирования**
- Трудно протестировать все возможные сценарии
- Сложность создания интеграционных тестов
- Необходимость мокирования множества сервисов

#### 5. **Риски компенсирующих действий**
- Компенсирующие действия могут сами потерпеть неудачу
- Сложность обеспечения идемпотентности
- Возможность каскадных сбоев при откатах

#### 6. **Сложность мониторинга**
- Требуется специальная инфраструктура для отслеживания
- Сложность алертинга при проблемах
- Необходимость централизованного логирования

### Сравнение с альтернативами

| Аспект | Saga Pattern | 2PC (Two-Phase Commit) | Event Sourcing |
|--------|--------------|------------------------|----------------|
| **Сложность** | Средняя | Высокая | Высокая |
| **Производительность** | Высокая | Низкая | Средняя |
| **Надежность** | Высокая | Высокая | Высокая |
| **Масштабируемость** | Высокая | Низкая | Средняя |
| **Простота отладки** | Средняя | Низкая | Низкая |

### Сравнение Orchestration vs Choreography

| Аспект | Orchestration-based Saga | Choreography-based Saga |
|--------|-------------------------|-------------------------|
| **Сложность реализации** | Средняя | Низкая |
| **Централизация** | Высокая (есть координатор) | Низкая (нет координатора) |
| **Видимость процесса** | Высокая (центральный мониторинг) | Низкая (распределенный мониторинг) |
| **Отладка** | Простая (один координатор) | Сложная (множество сервисов) |
| **Тестирование** | Простое (мокирование координатора) | Сложное (мокирование событий) |
| **Масштабируемость** | Средняя (координатор может стать узким местом) | Высокая (нет единой точки отказа) |
| **Добавление новых шагов** | Сложное (изменение координатора) | Простое (добавление обработчика событий) |
| **Связанность сервисов** | Высокая (сервисы знают о координаторе) | Низкая (сервисы знают только о событиях) |

## Тестирование

Интеграционные тесты показывают полный цикл работы Saga Pattern и помогают убедиться в корректности выполнения компенсирующих действий.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest
@Transactional
class SagaIntegrationTest {
    
    @Autowired
    private OrderSagaService orderSagaService;
    
    @Autowired
    private SagaRepository sagaRepository;
    
    @Autowired
    private TestInventoryService testInventoryService;
    
    @Autowired
    private TestPaymentService testPaymentService;
    
    @Test
    void shouldProcessCompleteSagaFlow() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1", "item2"));
        
        // When
        Order order = orderSagaService.createOrder(request);
        
        // Then - проверяем, что заказ создан
        assertThat(order).isNotNull();
        assertThat(order.getStatus()).isEqualTo("CONFIRMED");
        
        // And - проверяем, что товары зарезервированы
        assertThat(testInventoryService.getReservedItems())
            .containsAll(request.getItems());
        
        // And - проверяем, что платеж списан
        assertThat(testPaymentService.getChargedAmount())
            .isEqualTo(order.getTotal());
        
        // And - проверяем, что Saga завершена успешно
        List<SagaExecution> executions = sagaRepository.findBySagaId("create-order-saga");
        assertThat(executions).hasSize(1);
        assertThat(executions.get(0).getStatus()).isEqualTo("COMPLETED");
    }
    
    @Test
    void shouldCompensateOnInventoryFailure() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1"));
        testInventoryService.setShouldFail(true);
        
        // When & Then - ожидаем исключение
        assertThatThrownBy(() -> orderSagaService.createOrder(request))
            .isInstanceOf(SagaExecutionException.class);
        
        // Then - проверяем, что заказ удален (компенсация)
        List<Order> orders = orderRepository.findAll();
        assertThat(orders).isEmpty();
        
        // And - проверяем, что товары не зарезервированы
        assertThat(testInventoryService.getReservedItems()).isEmpty();
        
        // And - проверяем, что платеж не списан
        assertThat(testPaymentService.getChargedAmount()).isZero();
        
        // And - проверяем статус Saga
        List<SagaExecution> executions = sagaRepository.findBySagaId("create-order-saga");
        assertThat(executions).hasSize(1);
        assertThat(executions.get(0).getStatus()).isEqualTo("FAILED");
    }
    
    @Test
    void shouldCompensateOnPaymentFailure() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1"));
        testPaymentService.setShouldFail(true);
        
        // When & Then - ожидаем исключение
        assertThatThrownBy(() -> orderSagaService.createOrder(request))
            .isInstanceOf(SagaExecutionException.class);
        
        // Then - проверяем, что заказ удален
        List<Order> orders = orderRepository.findAll();
        assertThat(orders).isEmpty();
        
        // And - проверяем, что товары освобождены
        assertThat(testInventoryService.getReservedItems()).isEmpty();
        
        // And - проверяем, что платеж не списан
        assertThat(testPaymentService.getChargedAmount()).isZero();
    }
}
```

#### Test сервисы для тестирования

```java
@Component
@Primary
public class TestInventoryService implements InventoryService {
    
    private final List<String> reservedItems = new ArrayList<>();
    private boolean shouldFail = false;
    
    @Override
    public void reserveItems(List<String> items) {
        if (shouldFail) {
            throw new RuntimeException("Inventory service failure");
        }
        reservedItems.addAll(items);
    }
    
    @Override
    public void releaseItems(List<String> items) {
        reservedItems.removeAll(items);
    }
    
    public List<String> getReservedItems() {
        return new ArrayList<>(reservedItems);
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
    
    public void clear() {
        reservedItems.clear();
    }
}

@Component
@Primary
public class TestPaymentService implements PaymentService {
    
    private BigDecimal chargedAmount = BigDecimal.ZERO;
    private boolean shouldFail = false;
    
    @Override
    public void chargeCustomer(String customerId, BigDecimal amount) {
        if (shouldFail) {
            throw new RuntimeException("Payment service failure");
        }
        chargedAmount = chargedAmount.add(amount);
    }
    
    @Override
    public void refundCustomer(String customerId, BigDecimal amount) {
        chargedAmount = chargedAmount.subtract(amount);
    }
    
    public BigDecimal getChargedAmount() {
        return chargedAmount;
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
    
    public void clear() {
        chargedAmount = BigDecimal.ZERO;
    }
}
```

#### Тест для Choreography-based Saga

```java
@SpringBootTest
@Transactional
class ChoreographySagaIntegrationTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private TestEventPublisher testEventPublisher;
    
    @Autowired
    private TestInventoryService testInventoryService;
    
    @Autowired
    private TestPaymentService testPaymentService;
    
    @Test
    void shouldProcessCompleteChoreographyFlow() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1", "item2"));
        
        // When
        Order order = orderService.createOrder(request);
        
        // Then - проверяем, что заказ создан
        assertThat(order).isNotNull();
        
        // And - проверяем, что событие order.created опубликовано
        assertThat(testEventPublisher.getPublishedEvents())
            .anyMatch(event -> event.getType().equals("order.created"));
        
        // When - симулируем обработку событий
        testEventPublisher.processEvents();
        
        // Then - проверяем, что товары зарезервированы
        assertThat(testInventoryService.getReservedItems())
            .containsAll(request.getItems());
        
        // And - проверяем, что платеж списан
        assertThat(testPaymentService.getChargedAmount())
            .isEqualTo(order.getTotal());
        
        // And - проверяем, что событие order.confirmed опубликовано
        assertThat(testEventPublisher.getPublishedEvents())
            .anyMatch(event -> event.getType().equals("order.confirmed"));
    }
    
    @Test
    void shouldCompensateOnInventoryFailure() {
        // Given
        OrderRequest request = new OrderRequest("customer123", List.of("item1"));
        testInventoryService.setShouldFail(true);
        
        // When
        Order order = orderService.createOrder(request);
        testEventPublisher.processEvents();
        
        // Then - проверяем, что заказ удален (компенсация)
        List<Order> orders = orderRepository.findAll();
        assertThat(orders).isEmpty();
        
        // And - проверяем, что товары не зарезервированы
        assertThat(testInventoryService.getReservedItems()).isEmpty();
        
        // And - проверяем, что платеж не списан
        assertThat(testPaymentService.getChargedAmount()).isZero();
        
        // And - проверяем, что событие order.cancelled опубликовано
        assertThat(testEventPublisher.getPublishedEvents())
            .anyMatch(event -> event.getType().equals("order.cancelled"));
    }
}

@Component
@Primary
public class TestEventPublisher implements EventPublisher {
    
    private final List<Event> publishedEvents = new ArrayList<>();
    private final List<EventListener> eventListeners = new ArrayList<>();
    
    @Override
    public void publish(String eventType, Object payload) {
        Event event = new Event(eventType, payload);
        publishedEvents.add(event);
    }
    
    @Override
    public void subscribe(EventListener listener) {
        eventListeners.add(listener);
    }
    
    public void processEvents() {
        for (Event event : publishedEvents) {
            for (EventListener listener : eventListeners) {
                listener.onEvent(event);
            }
        }
    }
    
    public List<Event> getPublishedEvents() {
        return new ArrayList<>(publishedEvents);
    }
    
    public void clear() {
        publishedEvents.clear();
    }
}
``` 