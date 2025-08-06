# Strategy Pattern

## Обзор паттерна

**Strategy Pattern** - это поведенческий паттерн проектирования, который позволяет определять семейство алгоритмов, инкапсулировать каждый из них и делать их взаимозаменяемыми.

### Что это такое?

Strategy Pattern позволяет выбирать алгоритм во время выполнения программы, не изменяя код, который использует этот алгоритм. Паттерн определяет семейство схожих алгоритмов и помещает каждый из них в отдельный класс, после чего алгоритмы можно взаимозаменять прямо во время выполнения программы.

В микросервисной архитектуре Strategy Pattern особенно полезен для реализации различных стратегий обработки данных, валидации, аутентификации, кэширования и других операций, которые могут меняться в зависимости от контекста или требований.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда один и тот же сервис должен обрабатывать данные по-разному в зависимости от различных условий. Например:

1. **Сценарий**: Сервис платежей должен обрабатывать платежи через разные платежные системы (Visa, MasterCard, PayPal)
2. **Требование**: Нужно реализовать единый интерфейс для обработки платежей с возможностью выбора стратегии
3. **Проблема**: Если использовать условные операторы (if-else), код становится сложным и трудно расширяемым
4. **Результат**: Монолитный код с множеством условий, который сложно поддерживать и тестировать

## Детальный анализ проблемы

### Сценарий 1: Условные операторы

```java
@Service
public class PaymentService {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    public PaymentResult processPayment(PaymentRequest request) {
        // 1. Сохраняем платеж
        Payment payment = paymentRepository.save(new Payment(request));
        
        // 2. Обрабатываем платеж (ПРОБЛЕМА ЗДЕСЬ!)
        if ("VISA".equals(request.getPaymentType())) {
            return processVisaPayment(payment);
        } else if ("MASTERCARD".equals(request.getPaymentType())) {
            return processMasterCardPayment(payment);
        } else if ("PAYPAL".equals(request.getPaymentType())) {
            return processPayPalPayment(payment);
        } else {
            throw new UnsupportedPaymentTypeException(request.getPaymentType());
        }
    }
    
    private PaymentResult processVisaPayment(Payment payment) {
        // Специфичная логика для Visa
        return new PaymentResult("VISA_PROCESSED", payment.getId());
    }
    
    private PaymentResult processMasterCardPayment(Payment payment) {
        // Специфичная логика для MasterCard
        return new PaymentResult("MASTERCARD_PROCESSED", payment.getId());
    }
    
    private PaymentResult processPayPalPayment(Payment payment) {
        // Специфичная логика для PayPal
        return new PaymentResult("PAYPAL_PROCESSED", payment.getId());
    }
}
```

**Проблемы:**
- Нарушение принципа Open/Closed - для добавления новой платежной системы нужно изменять существующий код
- Сложность тестирования - каждый метод нужно тестировать отдельно
- Высокая связанность - класс знает о всех возможных типах платежей
- Сложность расширения - добавление новой стратегии требует изменения основного класса

### Сценарий 2: Наследование

```java
public abstract class PaymentProcessor {
    public abstract PaymentResult process(Payment payment);
}

public class VisaPaymentProcessor extends PaymentProcessor {
    @Override
    public PaymentResult process(Payment payment) {
        // Логика для Visa
        return new PaymentResult("VISA_PROCESSED", payment.getId());
    }
}

public class MasterCardPaymentProcessor extends PaymentProcessor {
    @Override
    public PaymentResult process(Payment payment) {
        // Логика для MasterCard
        return new PaymentResult("MASTERCARD_PROCESSED", payment.getId());
    }
}

@Service
public class PaymentService {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    public PaymentResult processPayment(PaymentRequest request) {
        Payment payment = paymentRepository.save(new Payment(request));
        
        // ПРОБЛЕМА ЗДЕСЬ!
        PaymentProcessor processor = createProcessor(request.getPaymentType());
        return processor.process(payment);
    }
    
    private PaymentProcessor createProcessor(String paymentType) {
        switch (paymentType) {
            case "VISA": return new VisaPaymentProcessor();
            case "MASTERCARD": return new MasterCardPaymentProcessor();
            default: throw new UnsupportedPaymentTypeException(paymentType);
        }
    }
}
```

**Проблемы:**
- Сложность создания объектов - нужно знать все возможные типы
- Нарушение принципа инверсии зависимостей - сервис зависит от конкретных классов
- Сложность конфигурации - стратегии создаются в коде, а не через DI
- Проблемы с состоянием - каждый процессор создается заново

### Сценарий 3: Enum с методами

```java
public enum PaymentType {
    VISA {
        @Override
        public PaymentResult process(Payment payment) {
            // Логика для Visa
            return new PaymentResult("VISA_PROCESSED", payment.getId());
        }
    },
    MASTERCARD {
        @Override
        public PaymentResult process(Payment payment) {
            // Логика для MasterCard
            return new PaymentResult("MASTERCARD_PROCESSED", payment.getId());
        }
    };
    
    public abstract PaymentResult process(Payment payment);
}

@Service
public class PaymentService {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    public PaymentResult processPayment(PaymentRequest request) {
        Payment payment = paymentRepository.save(new Payment(request));
        
        // ПРОБЛЕМА ЗДЕСЬ!
        return PaymentType.valueOf(request.getPaymentType()).process(payment);
    }
}
```

**Проблемы:**
- Ограниченная функциональность - enum не может иметь сложное состояние
- Сложность тестирования - логика смешана с перечислением
- Нарушение принципа единственной ответственности - enum отвечает за логику
- Сложность расширения - добавление новой стратегии требует изменения enum

### Корень проблемы

Основная проблема заключается в том, что **алгоритмы жестко связаны с кодом, который их использует**. Это приводит к:

1. **Нарушению принципа Open/Closed** - добавление новых алгоритмов требует изменения существующего кода
2. **Сложности тестирования** - каждый алгоритм нужно тестировать в контексте основного класса
3. **Высокой связанности** - классы знают о конкретных реализациях алгоритмов
4. **Сложности конфигурации** - выбор алгоритма происходит в коде, а не через DI

### Как работает паттерн

1. **Определение интерфейса стратегии** - создается общий интерфейс для всех алгоритмов
2. **Реализация конкретных стратегий** - каждый алгоритм инкапсулируется в отдельный класс
3. **Инъекция стратегии** - стратегия передается в контекст через конструктор или метод
4. **Выполнение алгоритма** - контекст использует стратегию, не зная о её конкретной реализации

## Детальный принцип работы

### Шаг 1: Определение интерфейса стратегии

```java
public interface PaymentStrategy {
    PaymentResult process(Payment payment);
    boolean supports(String paymentType);
}
```

**Ключевые моменты:**
- Интерфейс определяет общий контракт для всех стратегий
- Метод `supports()` позволяет определить, подходит ли стратегия для конкретного типа
- Стратегия получает все необходимые данные через параметры

### Шаг 2: Реализация конкретных стратегий

```java
@Component
public class VisaPaymentStrategy implements PaymentStrategy {
    
    @Autowired
    private VisaPaymentGateway visaGateway;
    
    @Override
    public PaymentResult process(Payment payment) {
        // Специфичная логика для Visa
        VisaPaymentRequest request = new VisaPaymentRequest(
            payment.getAmount(),
            payment.getCardNumber(),
            payment.getExpiryDate()
        );
        
        VisaPaymentResponse response = visaGateway.process(request);
        
        return new PaymentResult(
            response.getTransactionId(),
            payment.getId(),
            response.getStatus()
        );
    }
    
    @Override
    public boolean supports(String paymentType) {
        return "VISA".equals(paymentType);
    }
}

@Component
public class MasterCardPaymentStrategy implements PaymentStrategy {
    
    @Autowired
    private MasterCardPaymentGateway masterCardGateway;
    
    @Override
    public PaymentResult process(Payment payment) {
        // Специфичная логика для MasterCard
        MasterCardPaymentRequest request = new MasterCardPaymentRequest(
            payment.getAmount(),
            payment.getCardNumber(),
            payment.getCvv()
        );
        
        MasterCardPaymentResponse response = masterCardGateway.process(request);
        
        return new PaymentResult(
            response.getTransactionId(),
            payment.getId(),
            response.getStatus()
        );
    }
    
    @Override
    public boolean supports(String paymentType) {
        return "MASTERCARD".equals(paymentType);
    }
}

@Component
public class PayPalPaymentStrategy implements PaymentStrategy {
    
    @Autowired
    private PayPalPaymentGateway payPalGateway;
    
    @Override
    public PaymentResult process(Payment payment) {
        // Специфичная логика для PayPal
        PayPalPaymentRequest request = new PayPalPaymentRequest(
            payment.getAmount(),
            payment.getEmail(),
            payment.getToken()
        );
        
        PayPalPaymentResponse response = payPalGateway.process(request);
        
        return new PaymentResult(
            response.getTransactionId(),
            payment.getId(),
            response.getStatus()
        );
    }
    
    @Override
    public boolean supports(String paymentType) {
        return "PAYPAL".equals(paymentType);
    }
}
```

### Шаг 3: Контекст с инъекцией стратегий

```java
@Service
public class PaymentService {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private List<PaymentStrategy> paymentStrategies;
    
    public PaymentResult processPayment(PaymentRequest request) {
        // 1. Сохраняем платеж
        Payment payment = paymentRepository.save(new Payment(request));
        
        // 2. Находим подходящую стратегию
        PaymentStrategy strategy = findStrategy(request.getPaymentType());
        
        // 3. Обрабатываем платеж через стратегию
        return strategy.process(payment);
    }
    
    private PaymentStrategy findStrategy(String paymentType) {
        return paymentStrategies.stream()
            .filter(strategy -> strategy.supports(paymentType))
            .findFirst()
            .orElseThrow(() -> new UnsupportedPaymentTypeException(paymentType));
    }
}
```

### Шаг 4: Фабрика стратегий (альтернативный подход)

```java
@Component
public class PaymentStrategyFactory {
    
    private final Map<String, PaymentStrategy> strategies;
    
    public PaymentStrategyFactory(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(
                strategy -> strategy.getClass().getSimpleName(),
                strategy -> strategy
            ));
    }
    
    public PaymentStrategy getStrategy(String paymentType) {
        return strategies.values().stream()
            .filter(strategy -> strategy.supports(paymentType))
            .findFirst()
            .orElseThrow(() -> new UnsupportedPaymentTypeException(paymentType));
    }
}

@Service
public class PaymentServiceWithFactory {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private PaymentStrategyFactory strategyFactory;
    
    public PaymentResult processPayment(PaymentRequest request) {
        Payment payment = paymentRepository.save(new Payment(request));
        
        PaymentStrategy strategy = strategyFactory.getStrategy(request.getPaymentType());
        return strategy.process(payment);
    }
}
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ PaymentService  │    │ PaymentStrategy │    │ VisaStrategy    │
│                 │    │   (Interface)   │    │                 │
│ 1. Save Payment │───▶│ 2. process()    │───▶│ 3. Visa Logic   │
│                 │    │                 │    │                 │
│ 4. Get Result   │    │                 │    │ 4. Return       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │MasterCardStrategy│
                       │                 │
                       │ MasterCard Logic│
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │ PayPalStrategy  │
                       │                 │
                       │  PayPal Logic   │
                       └─────────────────┘
```

### Ключевые преимущества

1. **Гибкость**: Легко добавлять новые стратегии без изменения существующего кода
2. **Тестируемость**: Каждую стратегию можно тестировать независимо
3. **Расширяемость**: Новые алгоритмы не влияют на существующие
4. **Инкапсуляция**: Логика алгоритмов скрыта от клиентского кода
5. **Конфигурируемость**: Стратегии можно настраивать через DI контейнер

### Ключевые принципы

- **Инверсия зависимостей**: Контекст зависит от абстракции, а не от конкретных реализаций
- **Открытость/закрытость**: Система открыта для расширения, но закрыта для модификации
- **Единственная ответственность**: Каждая стратегия отвечает только за свой алгоритм
- **Полиморфизм**: Контекст работает с любыми реализациями интерфейса

### Когда использовать

Strategy Pattern подходит для случаев, когда:
- Есть семейство схожих алгоритмов, которые отличаются только деталями реализации
- Нужно скрыть сложную логику выбора алгоритма от клиентского кода
- Требуется возможность выбора алгоритма во время выполнения
- Важна возможность легкого добавления новых алгоритмов

## Преимущества и недостатки

### Преимущества

#### 1. **Гибкость и расширяемость**
- Легко добавлять новые стратегии без изменения существующего кода
- Каждая стратегия инкапсулирована в отдельном классе
- Система открыта для расширения, но закрыта для модификации

#### 2. **Упрощение тестирования**
- Каждую стратегию можно тестировать независимо
- Моки и стабы легко создавать для конкретных стратегий
- Изолированное тестирование алгоритмов

#### 3. **Устранение условных операторов**
- Заменяет сложные if-else конструкции
- Упрощает читаемость кода
- Уменьшает цикломатическую сложность

#### 4. **Инкапсуляция алгоритмов**
- Логика алгоритмов скрыта от клиентского кода
- Легко изменять реализацию без влияния на другие части
- Четкое разделение ответственности

#### 5. **Конфигурируемость**
- Стратегии можно настраивать через DI контейнер
- Возможность динамического выбора стратегии
- Легкая замена стратегий в runtime

#### 6. **Повторное использование**
- Стратегии можно использовать в разных контекстах
- Общие алгоритмы выносятся в отдельные классы
- Уменьшение дублирования кода

### Недостатки

#### 1. **Увеличение количества классов**
- Каждая стратегия требует отдельного класса
- Может привести к "взрыву" количества классов
- Сложность навигации по коду

#### 2. **Сложность выбора стратегии**
- Клиент должен знать о доступных стратегиях
- Может потребоваться фабрика для создания стратегий
- Сложность конфигурации в простых случаях

#### 3. **Потенциальная избыточность**
- Для простых случаев паттерн может быть избыточным
- Дополнительная сложность для простых алгоритмов
- Риск over-engineering

#### 4. **Сложность отладки**
- Логика распределена по нескольким классам
- Сложнее отследить выполнение алгоритма
- Может потребоваться дополнительное логирование

#### 5. **Зависимость от интерфейса**
- Все стратегии должны соответствовать общему интерфейсу
- Изменение интерфейса влияет на все стратегии
- Сложность эволюции API

#### 6. **Потенциальные проблемы производительности**
- Дополнительные вызовы методов
- Создание объектов стратегий
- Накладные расходы на полиморфизм

### Сравнение с альтернативами

| Аспект | Strategy Pattern | Template Method | Command Pattern |
|--------|------------------|-----------------|-----------------|
| **Сложность** | Средняя | Низкая | Высокая |
| **Гибкость** | Высокая | Средняя | Высокая |
| **Тестируемость** | Высокая | Средняя | Высокая |
| **Расширяемость** | Высокая | Средняя | Высокая |
| **Производительность** | Средняя | Высокая | Низкая |

## Тестирование

Интеграционные тесты показывают полный цикл работы Strategy Pattern и помогают понять, как стратегии работают в реальном контексте.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest
@Transactional
class PaymentStrategyIntegrationTest {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @MockBean
    private VisaPaymentGateway visaGateway;
    
    @MockBean
    private MasterCardPaymentGateway masterCardGateway;
    
    @Test
    void shouldProcessVisaPaymentWithStrategy() {
        // Given
        PaymentRequest request = new PaymentRequest("VISA", 100.0, "4111111111111111");
        VisaPaymentResponse mockResponse = new VisaPaymentResponse("TXN_123", "SUCCESS");
        when(visaGateway.process(any(VisaPaymentRequest.class)))
            .thenReturn(mockResponse);
        
        // When
        PaymentResult result = paymentService.processPayment(request);
        
        // Then - проверяем, что платеж сохранен
        Payment savedPayment = paymentRepository.findByTransactionId(result.getTransactionId());
        assertThat(savedPayment).isNotNull();
        assertThat(savedPayment.getPaymentType()).isEqualTo("VISA");
        
        // And - проверяем, что использована правильная стратегия
        assertThat(result.getTransactionId()).isEqualTo("TXN_123");
        assertThat(result.getStatus()).isEqualTo("SUCCESS");
        
        // And - проверяем, что gateway был вызван
        verify(visaGateway).process(any(VisaPaymentRequest.class));
    }
    
    @Test
    void shouldHandleUnsupportedPaymentType() {
        // Given
        PaymentRequest request = new PaymentRequest("UNSUPPORTED", 100.0, "4111111111111111");
        
        // When & Then
        assertThatThrownBy(() -> paymentService.processPayment(request))
            .isInstanceOf(UnsupportedPaymentTypeException.class)
            .hasMessageContaining("UNSUPPORTED");
    }
    
    @Test
    void shouldProcessMultiplePaymentTypes() {
        // Given
        PaymentRequest visaRequest = new PaymentRequest("VISA", 100.0, "4111111111111111");
        PaymentRequest masterCardRequest = new PaymentRequest("MASTERCARD", 200.0, "5555555555554444");
        
        when(visaGateway.process(any())).thenReturn(new VisaPaymentResponse("TXN_VISA", "SUCCESS"));
        when(masterCardGateway.process(any())).thenReturn(new MasterCardPaymentResponse("TXN_MC", "SUCCESS"));
        
        // When
        PaymentResult visaResult = paymentService.processPayment(visaRequest);
        PaymentResult masterCardResult = paymentService.processPayment(masterCardRequest);
        
        // Then
        assertThat(visaResult.getTransactionId()).isEqualTo("TXN_VISA");
        assertThat(masterCardResult.getTransactionId()).isEqualTo("TXN_MC");
        
        verify(visaGateway).process(any(VisaPaymentRequest.class));
        verify(masterCardGateway).process(any(MasterCardPaymentRequest.class));
    }
}
```

#### TestPaymentStrategy для тестирования

```java
@Component
@Primary
public class TestPaymentStrategy implements PaymentStrategy {
    
    private final List<Payment> processedPayments = new ArrayList<>();
    private boolean shouldFail = false;
    private String supportedType = "TEST";
    
    @Override
    public PaymentResult process(Payment payment) {
        if (shouldFail) {
            throw new RuntimeException("Test payment failure");
        }
        
        processedPayments.add(payment);
        return new PaymentResult("TEST_TXN_" + payment.getId(), payment.getId(), "SUCCESS");
    }
    
    @Override
    public boolean supports(String paymentType) {
        return supportedType.equals(paymentType);
    }
    
    public List<Payment> getProcessedPayments() {
        return new ArrayList<>(processedPayments);
    }
    
    public void clear() {
        processedPayments.clear();
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
    
    public void setSupportedType(String supportedType) {
        this.supportedType = supportedType;
    }
}
``` 