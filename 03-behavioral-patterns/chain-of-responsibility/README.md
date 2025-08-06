# Chain of Responsibility Pattern

## Обзор паттерна

**Chain of Responsibility Pattern** - это поведенческий паттерн проектирования, который позволяет передавать запросы по цепочке обработчиков до тех пор, пока один из них не обработает запрос.

### Что это такое?

Chain of Responsibility Pattern создает цепочку объектов-обработчиков, где каждый обработчик решает, может ли он обработать запрос, и передает его следующему обработчику в цепочке. Паттерн позволяет избежать жесткой привязки отправителя запроса к получателю, давая возможность нескольким объектам обработать запрос.

В микросервисной архитектуре Chain of Responsibility особенно полезен для реализации middleware, валидации запросов, аутентификации, логирования, обработки ошибок и других операций, которые должны выполняться последовательно.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда запрос должен пройти через несколько этапов обработки, и каждый этап может либо обработать запрос, либо передать его дальше. Например:

1. **Сценарий**: HTTP запрос должен пройти через аутентификацию, авторизацию, валидацию и логирование
2. **Требование**: Нужно реализовать гибкую систему обработки запросов с возможностью добавления новых этапов
3. **Проблема**: Если использовать жестко связанные обработчики, код становится сложным и трудно расширяемым
4. **Результат**: Монолитный код с множеством зависимостей, который сложно поддерживать и тестировать

## Детальный анализ проблемы

### Сценарий 1: Жестко связанные обработчики

```java
@Service
public class RequestProcessor {
    
    @Autowired
    private AuthenticationService authService;
    
    @Autowired
    private AuthorizationService authzService;
    
    @Autowired
    private ValidationService validationService;
    
    @Autowired
    private LoggingService loggingService;
    
    public Response processRequest(Request request) {
        // 1. Аутентификация (ПРОБЛЕМА ЗДЕСЬ!)
        if (!authService.authenticate(request)) {
            return new Response("UNAUTHORIZED", 401);
        }
        
        // 2. Авторизация
        if (!authzService.authorize(request)) {
            return new Response("FORBIDDEN", 403);
        }
        
        // 3. Валидация
        if (!validationService.validate(request)) {
            return new Response("BAD_REQUEST", 400);
        }
        
        // 4. Логирование
        loggingService.log(request);
        
        // 5. Обработка
        return processBusinessLogic(request);
    }
    
    private Response processBusinessLogic(Request request) {
        // Бизнес-логика
        return new Response("SUCCESS", 200);
    }
}
```

**Проблемы:**
- Нарушение принципа единственной ответственности - класс знает о всех обработчиках
- Сложность тестирования - каждый обработчик нужно тестировать в контексте основного класса
- Высокая связанность - изменение одного обработчика может повлиять на другие
- Сложность расширения - добавление нового обработчика требует изменения основного класса

### Сценарий 2: Наследование обработчиков

```java
public abstract class RequestHandler {
    protected RequestHandler nextHandler;
    
    public void setNext(RequestHandler next) {
        this.nextHandler = next;
    }
    
    public abstract Response handle(Request request);
}

public class AuthenticationHandler extends RequestHandler {
    @Override
    public Response handle(Request request) {
        if (!authenticate(request)) {
            return new Response("UNAUTHORIZED", 401);
        }
        
        // ПРОБЛЕМА ЗДЕСЬ!
        if (nextHandler != null) {
            return nextHandler.handle(request);
        }
        
        return new Response("SUCCESS", 200);
    }
    
    private boolean authenticate(Request request) {
        // Логика аутентификации
        return true;
    }
}

@Service
public class RequestProcessorWithInheritance {
    
    public Response processRequest(Request request) {
        // Создание цепочки
        RequestHandler authHandler = new AuthenticationHandler();
        RequestHandler authzHandler = new AuthorizationHandler();
        RequestHandler validationHandler = new ValidationHandler();
        
        authHandler.setNext(authzHandler);
        authzHandler.setNext(validationHandler);
        
        return authHandler.handle(request);
    }
}
```

**Проблемы:**
- Сложность создания цепочки - нужно знать порядок обработчиков
- Нарушение принципа инверсии зависимостей - сервис зависит от конкретных классов
- Сложность конфигурации - цепочка создается в коде, а не через DI
- Проблемы с состоянием - каждый обработчик создается заново

### Сценарий 3: Enum с обработчиками

```java
public enum RequestStep {
    AUTHENTICATION {
        @Override
        public Response process(Request request) {
            if (!authenticate(request)) {
                return new Response("UNAUTHORIZED", 401);
            }
            return null; // Продолжить цепочку
        }
    },
    AUTHORIZATION {
        @Override
        public Response process(Request request) {
            if (!authorize(request)) {
                return new Response("FORBIDDEN", 403);
            }
            return null; // Продолжить цепочку
        }
    },
    VALIDATION {
        @Override
        public Response process(Request request) {
            if (!validate(request)) {
                return new Response("BAD_REQUEST", 400);
            }
            return new Response("SUCCESS", 200);
        }
    };
    
    public abstract Response process(Request request);
}

@Service
public class RequestProcessorWithEnum {
    
    public Response processRequest(Request request) {
        // ПРОБЛЕМА ЗДЕСЬ!
        for (RequestStep step : RequestStep.values()) {
            Response response = step.process(request);
            if (response != null) {
                return response;
            }
        }
        
        return new Response("SUCCESS", 200);
    }
}
```

**Проблемы:**
- Ограниченная функциональность - enum не может иметь сложное состояние
- Сложность тестирования - логика смешана с перечислением
- Нарушение принципа единственной ответственности - enum отвечает за логику
- Сложность расширения - добавление нового обработчика требует изменения enum

### Корень проблемы

Основная проблема заключается в том, что **обработчики жестко связаны между собой и с кодом, который их использует**. Это приводит к:

1. **Нарушению принципа единственной ответственности** - классы знают о других обработчиках
2. **Сложности тестирования** - каждый обработчик нужно тестировать в контексте цепочки
3. **Высокой связанности** - изменение одного обработчика влияет на другие
4. **Сложности конфигурации** - порядок обработчиков определяется в коде

### Как работает паттерн

1. **Определение интерфейса обработчика** - создается общий интерфейс для всех обработчиков
2. **Реализация конкретных обработчиков** - каждый обработчик инкапсулируется в отдельный класс
3. **Создание цепочки** - обработчики связываются в цепочку через ссылки на следующий
4. **Передача запроса** - запрос передается по цепочке до первого обработчика, который его обработает

## Детальный принцип работы

### Шаг 1: Определение интерфейса обработчика

```java
public interface RequestHandler {
    Response handle(Request request);
    void setNext(RequestHandler next);
    boolean canHandle(Request request);
}
```

**Ключевые моменты:**
- Интерфейс определяет общий контракт для всех обработчиков
- Метод `canHandle()` позволяет определить, может ли обработчик обработать запрос
- Метод `setNext()` устанавливает следующий обработчик в цепочке

### Шаг 2: Реализация конкретных обработчиков

```java
@Component
public class AuthenticationHandler implements RequestHandler {
    
    @Autowired
    private AuthenticationService authService;
    
    private RequestHandler next;
    
    @Override
    public Response handle(Request request) {
        // Проверяем, можем ли мы обработать запрос
        if (!canHandle(request)) {
            return next != null ? next.handle(request) : new Response("SUCCESS", 200);
        }
        
        // Выполняем аутентификацию
        if (!authService.authenticate(request.getToken())) {
            return new Response("UNAUTHORIZED", 401, "Invalid token");
        }
        
        // Передаем запрос следующему обработчику
        return next != null ? next.handle(request) : new Response("SUCCESS", 200);
    }
    
    @Override
    public void setNext(RequestHandler next) {
        this.next = next;
    }
    
    @Override
    public boolean canHandle(Request request) {
        return request.getToken() != null;
    }
}

@Component
public class AuthorizationHandler implements RequestHandler {
    
    @Autowired
    private AuthorizationService authzService;
    
    private RequestHandler next;
    
    @Override
    public Response handle(Request request) {
        if (!canHandle(request)) {
            return next != null ? next.handle(request) : new Response("SUCCESS", 200);
        }
        
        // Проверяем права доступа
        if (!authzService.hasPermission(request.getUser(), request.getResource())) {
            return new Response("FORBIDDEN", 403, "Insufficient permissions");
        }
        
        return next != null ? next.handle(request) : new Response("SUCCESS", 200);
    }
    
    @Override
    public void setNext(RequestHandler next) {
        this.next = next;
    }
    
    @Override
    public boolean canHandle(Request request) {
        return request.getResource() != null;
    }
}

@Component
public class ValidationHandler implements RequestHandler {
    
    @Autowired
    private ValidationService validationService;
    
    private RequestHandler next;
    
    @Override
    public Response handle(Request request) {
        if (!canHandle(request)) {
            return next != null ? next.handle(request) : new Response("SUCCESS", 200);
        }
        
        // Валидируем запрос
        ValidationResult result = validationService.validate(request);
        if (!result.isValid()) {
            return new Response("BAD_REQUEST", 400, result.getErrors());
        }
        
        return next != null ? next.handle(request) : new Response("SUCCESS", 200);
    }
    
    @Override
    public void setNext(RequestHandler next) {
        this.next = next;
    }
    
    @Override
    public boolean canHandle(Request request) {
        return request.getData() != null;
    }
}

@Component
public class LoggingHandler implements RequestHandler {
    
    @Autowired
    private LoggingService loggingService;
    
    private RequestHandler next;
    
    @Override
    public Response handle(Request request) {
        // Логируем запрос
        loggingService.logRequest(request);
        
        // Передаем запрос следующему обработчику
        Response response = next != null ? next.handle(request) : new Response("SUCCESS", 200);
        
        // Логируем ответ
        loggingService.logResponse(response);
        
        return response;
    }
    
    @Override
    public void setNext(RequestHandler next) {
        this.next = next;
    }
    
    @Override
    public boolean canHandle(Request request) {
        return true; // Логирование применяется ко всем запросам
    }
}
```

### Шаг 3: Фабрика цепочки обработчиков

```java
@Component
public class RequestHandlerChainFactory {
    
    @Autowired
    private List<RequestHandler> handlers;
    
    public RequestHandler createChain() {
        if (handlers.isEmpty()) {
            return null;
        }
        
        // Сортируем обработчики по приоритету (если есть)
        List<RequestHandler> sortedHandlers = handlers.stream()
            .sorted(this::compareByPriority)
            .collect(Collectors.toList());
        
        // Создаем цепочку
        for (int i = 0; i < sortedHandlers.size() - 1; i++) {
            sortedHandlers.get(i).setNext(sortedHandlers.get(i + 1));
        }
        
        return sortedHandlers.get(0);
    }
    
    private int compareByPriority(RequestHandler h1, RequestHandler h2) {
        // Логика сравнения по приоритету
        return Integer.compare(getPriority(h1), getPriority(h2));
    }
    
    private int getPriority(RequestHandler handler) {
        if (handler instanceof AuthenticationHandler) return 1;
        if (handler instanceof AuthorizationHandler) return 2;
        if (handler instanceof ValidationHandler) return 3;
        if (handler instanceof LoggingHandler) return 4;
        return 5;
    }
}
```

### Шаг 4: Сервис с использованием цепочки

```java
@Service
public class RequestProcessor {
    
    @Autowired
    private RequestHandlerChainFactory chainFactory;
    
    public Response processRequest(Request request) {
        // 1. Создаем цепочку обработчиков
        RequestHandler chain = chainFactory.createChain();
        
        // 2. Передаем запрос в цепочку
        if (chain != null) {
            return chain.handle(request);
        }
        
        // 3. Если цепочка пустая, возвращаем успешный ответ
        return new Response("SUCCESS", 200);
    }
}

// Альтернативный подход с конфигурацией через @PostConstruct
@Service
public class RequestProcessorWithConfig {
    
    @Autowired
    private AuthenticationHandler authHandler;
    
    @Autowired
    private AuthorizationHandler authzHandler;
    
    @Autowired
    private ValidationHandler validationHandler;
    
    @Autowired
    private LoggingHandler loggingHandler;
    
    @PostConstruct
    public void setupChain() {
        // Настраиваем цепочку при инициализации
        authHandler.setNext(authzHandler);
        authzHandler.setNext(validationHandler);
        validationHandler.setNext(loggingHandler);
    }
    
    public Response processRequest(Request request) {
        return authHandler.handle(request);
    }
}
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ RequestProcessor│    │ AuthHandler     │    │ AuthzHandler    │
│                 │    │                 │    │                 │
│ 1. Create Chain │───▶│ 2. Authenticate │───▶│ 3. Authorize    │
│                 │    │                 │    │                 │
│ 6. Return Result│    │ 5. Pass to Next │    │ 4. Pass to Next │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                         │
                                                         ▼
                                               ┌─────────────────┐
                                               │ValidationHandler│
                                               │                 │
                                               │ 4. Validate     │
                                               │ 5. Pass to Next │
                                               └─────────────────┘
                                                         │
                                                         ▼
                                               ┌─────────────────┐
                                               │ LoggingHandler  │
                                               │                 │
                                               │ 5. Log Request  │
                                               │ 6. Log Response │
                                               └─────────────────┘
```

### Ключевые преимущества

1. **Гибкость**: Легко добавлять новые обработчики без изменения существующего кода
2. **Тестируемость**: Каждый обработчик можно тестировать независимо
3. **Расширяемость**: Новые обработчики не влияют на существующие
4. **Инкапсуляция**: Логика обработки скрыта от клиентского кода
5. **Конфигурируемость**: Порядок обработчиков можно настраивать

### Ключевые принципы

- **Единственная ответственность**: Каждый обработчик отвечает только за свою логику
- **Открытость/закрытость**: Система открыта для расширения, но закрыта для модификации
- **Слабая связанность**: Обработчики не знают друг о друге
- **Полиморфизм**: Клиент работает с любыми реализациями интерфейса

### Когда использовать

Chain of Responsibility Pattern подходит для случаев, когда:
- Есть несколько объектов, которые могут обработать запрос
- Нужно избежать жесткой привязки отправителя к получателю
- Требуется возможность динамического изменения цепочки обработчиков
- Важна возможность добавления новых обработчиков без изменения существующего кода

## Преимущества и недостатки

### Преимущества

#### 1. **Гибкость и расширяемость**
- Легко добавлять новые обработчики без изменения существующего кода
- Каждый обработчик инкапсулирован в отдельном классе
- Система открыта для расширения, но закрыта для модификации

#### 2. **Упрощение тестирования**
- Каждый обработчик можно тестировать независимо
- Моки и стабы легко создавать для конкретных обработчиков
- Изолированное тестирование логики обработки

#### 3. **Устранение жесткой связанности**
- Отправитель не знает о конкретных получателях
- Обработчики не знают друг о друге
- Слабая связанность между компонентами

#### 4. **Динамическая конфигурация**
- Цепочку можно изменять во время выполнения
- Порядок обработчиков можно настраивать
- Возможность условного добавления обработчиков

#### 5. **Единообразная обработка**
- Все обработчики следуют единому интерфейсу
- Стандартизированный подход к обработке запросов
- Легко понять и поддерживать

#### 6. **Повторное использование**
- Обработчики можно использовать в разных цепочках
- Общие обработчики выносятся в отдельные классы
- Уменьшение дублирования кода

### Недостатки

#### 1. **Сложность отладки**
- Логика распределена по нескольким классам
- Сложнее отследить выполнение запроса
- Может потребоваться дополнительное логирование

#### 2. **Потенциальные проблемы производительности**
- Запрос может пройти через всю цепочку
- Дополнительные вызовы методов
- Накладные расходы на передачу запроса

#### 3. **Сложность конфигурации**
- Нужно правильно настроить порядок обработчиков
- Может потребоваться фабрика для создания цепочки
- Сложность управления зависимостями

#### 4. **Риск бесконечной цепочки**
- Если обработчик не передает запрос дальше, цепочка прерывается
- Возможность потери запроса в цепочке
- Сложность обработки ошибок

#### 5. **Зависимость от интерфейса**
- Все обработчики должны соответствовать общему интерфейсу
- Изменение интерфейса влияет на все обработчики
- Сложность эволюции API

#### 6. **Потенциальная избыточность**
- Для простых случаев паттерн может быть избыточным
- Дополнительная сложность для простой обработки
- Риск over-engineering

### Сравнение с альтернативами

| Аспект | Chain of Responsibility | Decorator Pattern | Pipeline Pattern |
|--------|------------------------|-------------------|------------------|
| **Сложность** | Средняя | Низкая | Высокая |
| **Гибкость** | Высокая | Средняя | Высокая |
| **Тестируемость** | Высокая | Средняя | Высокая |
| **Производительность** | Средняя | Высокая | Низкая |
| **Отладка** | Сложная | Средняя | Сложная |

## Тестирование

Интеграционные тесты показывают полный цикл работы Chain of Responsibility Pattern и помогают понять, как обработчики работают в цепочке.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest
@Transactional
class RequestHandlerChainIntegrationTest {
    
    @Autowired
    private RequestProcessor requestProcessor;
    
    @MockBean
    private AuthenticationService authService;
    
    @MockBean
    private AuthorizationService authzService;
    
    @MockBean
    private ValidationService validationService;
    
    @MockBean
    private LoggingService loggingService;
    
    @Test
    void shouldProcessRequestThroughFullChain() {
        // Given
        Request request = new Request("valid-token", "user123", "resource1", "data");
        when(authService.authenticate("valid-token")).thenReturn(true);
        when(authzService.hasPermission("user123", "resource1")).thenReturn(true);
        when(validationService.validate(request)).thenReturn(new ValidationResult(true, null));
        
        // When
        Response response = requestProcessor.processRequest(request);
        
        // Then - проверяем, что запрос прошел через всю цепочку
        assertThat(response.getStatus()).isEqualTo(200);
        assertThat(response.getMessage()).isEqualTo("SUCCESS");
        
        // And - проверяем, что все сервисы были вызваны
        verify(authService).authenticate("valid-token");
        verify(authzService).hasPermission("user123", "resource1");
        verify(validationService).validate(request);
        verify(loggingService).logRequest(request);
        verify(loggingService).logResponse(response);
    }
    
    @Test
    void shouldStopChainOnAuthenticationFailure() {
        // Given
        Request request = new Request("invalid-token", "user123", "resource1", "data");
        when(authService.authenticate("invalid-token")).thenReturn(false);
        
        // When
        Response response = requestProcessor.processRequest(request);
        
        // Then - проверяем, что цепочка остановилась на аутентификации
        assertThat(response.getStatus()).isEqualTo(401);
        assertThat(response.getMessage()).isEqualTo("UNAUTHORIZED");
        
        // And - проверяем, что следующие обработчики не были вызваны
        verify(authzService, never()).hasPermission(any(), any());
        verify(validationService, never()).validate(any());
    }
    
    @Test
    void shouldStopChainOnAuthorizationFailure() {
        // Given
        Request request = new Request("valid-token", "user123", "resource1", "data");
        when(authService.authenticate("valid-token")).thenReturn(true);
        when(authzService.hasPermission("user123", "resource1")).thenReturn(false);
        
        // When
        Response response = requestProcessor.processRequest(request);
        
        // Then - проверяем, что цепочка остановилась на авторизации
        assertThat(response.getStatus()).isEqualTo(403);
        assertThat(response.getMessage()).isEqualTo("FORBIDDEN");
        
        // And - проверяем, что следующие обработчики не были вызваны
        verify(validationService, never()).validate(any());
    }
    
    @Test
    void shouldHandleEmptyChain() {
        // Given
        Request request = new Request("valid-token", "user123", "resource1", "data");
        
        // When - создаем процессор с пустой цепочкой
        RequestProcessor emptyProcessor = new RequestProcessor();
        Response response = emptyProcessor.processRequest(request);
        
        // Then - проверяем, что возвращается успешный ответ
        assertThat(response.getStatus()).isEqualTo(200);
        assertThat(response.getMessage()).isEqualTo("SUCCESS");
    }
}
```

#### TestRequestHandler для тестирования

```java
@Component
@Primary
public class TestRequestHandler implements RequestHandler {
    
    private final List<Request> processedRequests = new ArrayList<>();
    private boolean shouldFail = false;
    private String supportedType = "TEST";
    private RequestHandler next;
    
    @Override
    public Response handle(Request request) {
        if (shouldFail) {
            return new Response("TEST_ERROR", 500, "Test failure");
        }
        
        processedRequests.add(request);
        
        // Проверяем, можем ли мы обработать запрос
        if (canHandle(request)) {
            return new Response("TEST_PROCESSED", 200, "Test handler processed request");
        }
        
        // Передаем запрос следующему обработчику
        return next != null ? next.handle(request) : new Response("SUCCESS", 200);
    }
    
    @Override
    public void setNext(RequestHandler next) {
        this.next = next;
    }
    
    @Override
    public boolean canHandle(Request request) {
        return supportedType.equals(request.getType());
    }
    
    public List<Request> getProcessedRequests() {
        return new ArrayList<>(processedRequests);
    }
    
    public void clear() {
        processedRequests.clear();
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
    
    public void setSupportedType(String supportedType) {
        this.supportedType = supportedType;
    }
}
``` 