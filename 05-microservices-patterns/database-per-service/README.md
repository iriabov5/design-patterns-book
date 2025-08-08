# Database per Service Pattern

## Обзор паттерна

**Database per Service** - это паттерн проектирования в микросервисной архитектуре, где каждый сервис имеет собственную базу данных, изолированную от других сервисов.

### Что это такое?

Database per Service - это принцип, согласно которому каждый микросервис владеет и управляет своей собственной базой данных. Это означает, что данные сервиса полностью изолированы, и другие сервисы не могут напрямую обращаться к его базе данных. Взаимодействие между сервисами происходит только через API, а не через общую базу данных.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда несколько сервисов используют одну общую базу данных. Например:

1. **Сценарий**: Сервис заказов и сервис пользователей используют одну базу данных MySQL
2. **Требование**: Нужно изменить схему данных для одного сервиса без влияния на другой
3. **Проблема**: Изменения в схеме могут сломать другой сервис, а общая база становится узким местом
4. **Результат**: Система становится хрупкой, сложной в поддержке и масштабировании

## Детальный анализ проблемы

### Сценарий 1: Общая база данных

```java
@Service
public class OrderService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Проверяем пользователя в общей БД
        String sql = "SELECT * FROM users WHERE id = ?";
        User user = jdbcTemplate.queryForObject(sql, new Object[]{request.getUserId()}, User.class);
        
        // 2. Создаем заказ в той же БД
        String insertSql = "INSERT INTO orders (user_id, total_amount, status) VALUES (?, ?, ?)";
        jdbcTemplate.update(insertSql, user.getId(), request.getTotalAmount(), "PENDING");
        
        return order;
    }
}
```

**Проблемы:**
- Если изменить схему таблицы `users`, может сломаться `OrderService`
- Общая база данных становится узким местом производительности
- Сложно масштабировать отдельные сервисы независимо
- Нарушение принципа слабой связанности

### Сценарий 2: Прямые SQL запросы между сервисами

```java
@Service
public class NotificationService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void sendOrderNotification(Long orderId) {
        // ПРОБЛЕМА ЗДЕСЬ! - прямой доступ к данным другого сервиса
        String sql = "SELECT o.*, u.email FROM orders o " +
                    "JOIN users u ON o.user_id = u.id " +
                    "WHERE o.id = ?";
        
        OrderWithUser order = jdbcTemplate.queryForObject(sql, new Object[]{orderId}, OrderWithUser.class);
        
        // Отправляем уведомление
        emailService.sendEmail(order.getEmail(), "Order created", "Your order has been created");
    }
}
```

**Проблемы:**
- Нарушение границ сервисов
- Создание скрытых зависимостей
- Сложность рефакторинга и изменений
- Нарушение принципа единственной ответственности

### Сценарий 3: Общие транзакции

```java
@Service
public class OrderService {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Обновляем данные пользователя
        userService.updateUserBalance(request.getUserId(), request.getTotalAmount());
        
        // 2. Создаем заказ
        Order order = orderRepository.save(new Order(request));
        
        // 3. Обрабатываем платеж
        paymentService.processPayment(request.getPaymentDetails());
        
        return order;
    }
}
```

**Проблемы:**
- Распределенные транзакции сложны в реализации
- Высокий риск частичных откатов
- Сложность отладки при ошибках
- Нарушение принципа автономности сервисов

### Корень проблемы

Основная проблема заключается в том, что **сервисы не являются автономными и имеют сильную связанность через общую базу данных**. Это приводит к:

1. **Нарушению границ сервисов** - сервисы знают о внутренней структуре данных друг друга
2. **Сложности масштабирования** - нельзя масштабировать сервисы независимо
3. **Проблемам с производительностью** - общая база становится узким местом
4. **Сложности развертывания** - изменения в схеме влияют на все сервисы

### Как работает паттерн

1. **Изоляция данных**: Каждый сервис имеет собственную базу данных
2. **API-связи**: Сервисы взаимодействуют только через API
3. **Автономность**: Каждый сервис полностью контролирует свои данные
4. **Независимое масштабирование**: Сервисы можно масштабировать отдельно

## Детальный принцип работы

### Шаг 1: Изоляция баз данных

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private UserServiceClient userServiceClient; // HTTP клиент
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Получаем данные пользователя через API
        User user = userServiceClient.getUser(request.getUserId());
        
        // 2. Создаем заказ в своей БД
        Order order = new Order();
        order.setUserId(user.getId());
        order.setUserEmail(user.getEmail()); // Дублируем необходимые данные
        order.setTotalAmount(request.getTotalAmount());
        order.setStatus("PENDING");
        
        return orderRepository.save(order);
    }
}
```

**Ключевые моменты:**
- Сервис не обращается напрямую к БД другого сервиса
- Необходимые данные дублируются локально
- Каждый сервис полностью контролирует свою схему данных

### Шаг 2: Структура баз данных

```sql
-- База данных сервиса заказов (orders_db)
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(255) NOT NULL,
    user_email VARCHAR(255) NOT NULL, -- Дублированные данные
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
);

-- База данных сервиса пользователей (users_db)
CREATE TABLE users (
    id VARCHAR(255) PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    balance DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_email (email)
);
```

### Шаг 3: API-клиенты для межсервисного взаимодействия

```java
@Component
public class UserServiceClient {
    
    @Value("${user.service.url}")
    private String userServiceUrl;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public User getUser(String userId) {
        try {
            ResponseEntity<User> response = restTemplate.getForEntity(
                userServiceUrl + "/users/" + userId, 
                User.class
            );
            return response.getBody();
        } catch (Exception e) {
            throw new ServiceUnavailableException("User service is unavailable", e);
        }
    }
    
    public void updateUserBalance(String userId, BigDecimal amount) {
        try {
            restTemplate.put(
                userServiceUrl + "/users/" + userId + "/balance",
                new BalanceUpdateRequest(amount)
            );
        } catch (Exception e) {
            throw new ServiceUnavailableException("User service is unavailable", e);
        }
    }
}
```

### Шаг 4: Обработка ошибок и отказоустойчивость

```java
@Service
public class OrderService {
    
    @Autowired
    private UserServiceClient userServiceClient;
    
    @Autowired
    private CircuitBreaker circuitBreaker;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Получаем данные пользователя с защитой от сбоев
        User user = circuitBreaker.execute(() -> 
            userServiceClient.getUser(request.getUserId())
        );
        
        // 2. Если пользователь недоступен, используем кэшированные данные
        if (user == null) {
            user = getUserFromCache(request.getUserId());
        }
        
        // 3. Создаем заказ
        Order order = orderRepository.save(new Order(request, user));
        
        // 4. Асинхронно синхронизируем данные пользователя
        asyncUpdateUserData(user);
        
        return order;
    }
    
    @Async
    public void asyncUpdateUserData(User user) {
        try {
            userServiceClient.updateUserData(user);
        } catch (Exception e) {
            // Логируем ошибку, но не прерываем основной поток
            log.error("Failed to update user data", e);
        }
    }
}
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Order Service  │    │  User Service   │    │Payment Service  │
│                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │  Orders DB  │ │    │ │  Users DB   │ │    │ │ Payments DB │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
│                 │    │                 │    │                 │
│ 1. Create Order │    │ 2. Get User     │    │ 3. Process     │
│                 │    │                 │    │ Payment         │
│ 4. Return Order │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 ▼
                       ┌─────────────────┐
                       │   API Gateway   │
                       │                 │
                       │ Load Balancer   │
                       └─────────────────┘
```

### Ключевые преимущества

1. **Автономность сервисов**: Каждый сервис полностью контролирует свои данные
2. **Независимое масштабирование**: Сервисы можно масштабировать отдельно
3. **Изоляция отказов**: Проблемы в одной БД не влияют на другие сервисы
4. **Гибкость технологий**: Каждый сервис может использовать подходящую БД
5. **Простота развертывания**: Изменения в схеме не влияют на другие сервисы

### Конфигурация подключений

Каждый сервис настраивает подключение к своей базе данных независимо:

**Пример конфигурации Order Service:**
```properties
# application.properties
spring.datasource.url=jdbc:mysql://orders-db:3306/orders_db
spring.datasource.username=orders_user
spring.datasource.password=orders_pass
spring.jpa.hibernate.ddl-auto=validate
```

**Пример конфигурации User Service:**
```properties
# application.properties
spring.datasource.url=jdbc:postgresql://users-db:5432/users_db
spring.datasource.username=users_user
spring.datasource.password=users_pass
spring.jpa.hibernate.ddl-auto=validate
```

**Альтернативные стратегии:**
- **Разные типы БД**: MySQL для заказов, PostgreSQL для пользователей, MongoDB для логов
- **Разные схемы**: Один экземпляр БД, но разные схемы
- **Разные инстансы**: Полностью изолированные экземпляры БД
- **Контейнеризация**: Каждая БД в отдельном контейнере

### Ключевые принципы

- **Автономность**: Каждый сервис полностью контролирует свои данные
- **Изоляция**: Сервисы не имеют прямого доступа к БД друг друга
- **API-связи**: Взаимодействие происходит только через API
- **Дублирование данных**: Необходимые данные могут дублироваться для автономности

### Когда использовать

Database per Service подходит для случаев, когда:
- Требуется высокая автономность сервисов
- Нужно независимое масштабирование компонентов
- Важна изоляция отказов
- Разные сервисы имеют разные требования к данным

## Преимущества и недостатки

### Преимущества

#### 1. **Полная автономность сервисов**
- Каждый сервис полностью контролирует свои данные
- Изменения в схеме одного сервиса не влияют на другие
- Возможность выбора оптимальной БД для каждого сервиса
- Независимое развитие и развертывание

#### 2. **Независимое масштабирование**
- Можно масштабировать сервисы независимо друг от друга
- Оптимизация производительности для каждого сервиса
- Возможность горизонтального масштабирования отдельных компонентов
- Гибкость в распределении ресурсов

#### 3. **Изоляция отказов**
- Проблемы в одной БД не влияют на другие сервисы
- Повышение общей надежности системы
- Возможность локального восстановления после сбоев
- Снижение риска каскадных отказов

#### 4. **Гибкость технологий**
- Каждый сервис может использовать подходящую БД
- Возможность выбора SQL/NoSQL решений
- Оптимизация под конкретные требования
- Использование специализированных БД (графовые, документные)

#### 5. **Простота развертывания**
- Изменения в схеме не требуют координации между командами
- Возможность независимого CI/CD для каждого сервиса
- Снижение сложности миграций
- Быстрое внедрение новых функций

#### 6. **Улучшенная безопасность**
- Изоляция данных между сервисами
- Возможность настройки разных уровней доступа
- Снижение риска несанкционированного доступа
- Лучший контроль над данными

### Недостатки

#### 1. **Сложность управления данными**
- Необходимость синхронизации данных между сервисами
- Сложность обеспечения консистентности
- Проблемы с дублированием данных
- Сложность реализации распределенных транзакций

#### 2. **Дублирование данных**
- Необходимость хранения одних и тех же данных в разных БД
- Увеличение объема хранимых данных
- Сложность синхронизации при изменениях
- Риск рассинхронизации данных

#### 3. **Сложность запросов**
- Невозможность выполнения JOIN между данными разных сервисов
- Необходимость агрегации данных на уровне приложения
- Сложность реализации сложных отчетов
- Потребность в дополнительных сервисах для агрегации

#### 4. **Проблемы с производительностью**
- Дополнительные сетевые вызовы между сервисами
- Задержки при получении данных из других сервисов
- Сложность оптимизации запросов
- Потенциальные узкие места в API

#### 5. **Сложность операций**
- Увеличение количества компонентов для мониторинга
- Сложность резервного копирования и восстановления
- Необходимость управления множественными БД
- Повышенные требования к инфраструктуре

#### 6. **Проблемы с транзакциями**
- Сложность реализации распределенных транзакций
- Риск частичных откатов при сбоях
- Необходимость использования Saga паттернов
- Сложность обеспечения ACID свойств

### Сравнение с альтернативами

| Аспект | Database per Service | Shared Database | Event Sourcing |
|--------|---------------------|-----------------|----------------|
| **Сложность** | Средняя | Низкая | Высокая |
| **Производительность** | Средняя | Высокая | Низкая |
| **Надежность** | Высокая | Низкая | Высокая |
| **Масштабируемость** | Высокая | Низкая | Высокая |
| **Простота отладки** | Средняя | Высокая | Низкая |

## Тестирование

Интеграционные тесты показывают, как сервисы взаимодействуют через API и как обеспечивается изоляция данных.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest
@Transactional
class DatabasePerServiceIntegrationTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private TestUserServiceClient testUserServiceClient;
    
    @Test
    void shouldCreateOrderWithUserData() {
        // Given
        OrderRequest request = new OrderRequest("user123", BigDecimal.valueOf(100.0));
        User user = new User("user123", "user@example.com", "John Doe");
        testUserServiceClient.setUser(user);
        
        // When
        Order order = orderService.createOrder(request);
        
        // Then - проверяем, что заказ создан с данными пользователя
        assertThat(order.getUserId()).isEqualTo("user123");
        assertThat(order.getUserEmail()).isEqualTo("user@example.com");
        assertThat(order.getTotalAmount()).isEqualTo(BigDecimal.valueOf(100.0));
        
        // And - проверяем, что данные сохранены в локальной БД
        Order savedOrder = orderRepository.findById(order.getId()).orElseThrow();
        assertThat(savedOrder.getUserEmail()).isEqualTo("user@example.com");
    }
    
    @Test
    void shouldHandleUserServiceFailure() {
        // Given
        OrderRequest request = new OrderRequest("user123", BigDecimal.valueOf(100.0));
        testUserServiceClient.setShouldFail(true);
        
        // When
        Order order = orderService.createOrder(request);
        
        // Then - заказ создан с кэшированными данными пользователя
        assertThat(order.getUserId()).isEqualTo("user123");
        assertThat(order.getUserEmail()).isNotNull(); // Из кэша
        
        // And - проверяем, что попытка синхронизации была выполнена
        assertThat(testUserServiceClient.getCallCount()).isGreaterThan(0);
    }
}
```

#### TestUserServiceClient для тестирования

```java
@Component
@Primary
public class TestUserServiceClient implements UserServiceClient {
    
    private User user = new User("test123", "test@example.com", "Test User");
    private boolean shouldFail = false;
    private int callCount = 0;
    
    @Override
    public User getUser(String userId) {
        callCount++;
        if (shouldFail) {
            throw new ServiceUnavailableException("User service unavailable");
        }
        return user;
    }
    
    @Override
    public void updateUserBalance(String userId, BigDecimal amount) {
        callCount++;
        if (shouldFail) {
            throw new ServiceUnavailableException("User service unavailable");
        }
        // Test implementation
    }
    
    public void setUser(User user) {
        this.user = user;
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