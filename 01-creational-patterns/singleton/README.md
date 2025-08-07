# Singleton (Одиночка)

## Обзор паттерна

**Краткое определение** - паттерн проектирования, который гарантирует, что у класса есть только один экземпляр, и предоставляет глобальную точку доступа к нему.

### Что это такое?

Singleton - это креационный паттерн, который решает проблему создания единственного экземпляра класса в приложении. В микросервисной архитектуре этот паттерн часто используется для управления глобальными ресурсами, такими как конфигурация приложения, подключения к базе данных, кэши или логгеры.

Паттерн обеспечивает, что класс имеет только один экземпляр во всем приложении, и предоставляет глобальную точку доступа к этому экземпляру. Это особенно важно в многопоточной среде, где необходимо гарантировать единственность экземпляра.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда необходимо иметь единственный экземпляр определенного класса во всем приложении. Например:

1. **Сценарий**: Приложение использует несколько сервисов, которые должны работать с одной и той же конфигурацией, подключением к базе данных или кэшем
2. **Требование**: Обеспечить, чтобы все компоненты использовали один и тот же экземпляр ресурса, а не создавали свои собственные копии
3. **Проблема**: Без Singleton каждый сервис может создать свой экземпляр, что приведет к избыточному потреблению ресурсов, несогласованности данных и сложностям в управлении
4. **Результат**: Singleton обеспечивает единственность экземпляра, экономит ресурсы и гарантирует согласованность данных

## Детальный анализ проблемы

### Сценарий 1: Создание множественных экземпляров конфигурации

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Transactional
    public User createUser(CreateUserRequest request) {
        // 1. Сохраняем пользователя
        User user = userRepository.save(new User(request));
        
        // 2. Создаем новый экземпляр конфигурации (ПРОБЛЕМА ЗДЕСЬ!)
        AppConfig config = new AppConfig();
        config.loadFromFile("application.properties");
        
        // 3. Используем конфигурацию
        if (config.isFeatureEnabled("user-validation")) {
            validateUser(user);
        }
        
        return user;
    }
}

@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // 1. Сохраняем заказ
        Order order = orderRepository.save(new Order(request));
        
        // 2. Создаем ЕЩЕ ОДИН экземпляр конфигурации (ПРОБЛЕМА ЗДЕСЬ!)
        AppConfig config = new AppConfig();
        config.loadFromFile("application.properties");
        
        // 3. Используем конфигурацию
        if (config.isFeatureEnabled("order-validation")) {
            validateOrder(order);
        }
        
        return order;
    }
}
```

**Проблемы:**
- Каждый сервис создает свой экземпляр конфигурации, что приводит к избыточному потреблению памяти
- Файл конфигурации читается несколько раз, что замедляет работу приложения
- Изменения в конфигурации не синхронизируются между сервисами
- Нет централизованного управления конфигурацией

### Сценарий 2: Множественные подключения к базе данных

```java
@Component
public class DatabaseConnection {
    
    private Connection connection;
    private final String url;
    private final String username;
    private final String password;
    
    public DatabaseConnection() {
        // ПРОБЛЕМА ЗДЕСЬ! - каждый экземпляр создает новое подключение
        this.url = "jdbc:mysql://localhost:3306/mydb";
        this.username = "user";
        this.password = "password";
        this.connection = createConnection();
    }
    
    private Connection createConnection() {
        try {
            return DriverManager.getConnection(url, username, password);
        } catch (SQLException e) {
            throw new RuntimeException("Failed to create database connection", e);
        }
    }
    
    public Connection getConnection() {
        return connection;
    }
}

@Service
public class UserService {
    
    private DatabaseConnection dbConnection = new DatabaseConnection(); // ПРОБЛЕМА ЗДЕСЬ!
    
    public List<User> getAllUsers() {
        // Используем подключение
        return dbConnection.getConnection().createStatement()
            .executeQuery("SELECT * FROM users");
    }
}

@Service
public class OrderService {
    
    private DatabaseConnection dbConnection = new DatabaseConnection(); // ПРОБЛЕМА ЗДЕСЬ!
    
    public List<Order> getAllOrders() {
        // Используем другое подключение
        return dbConnection.getConnection().createStatement()
            .executeQuery("SELECT * FROM orders");
    }
}
```

**Проблемы:**
- Создается множество подключений к базе данных, что превышает лимиты соединений
- Высокое потребление ресурсов сервера базы данных
- Отсутствие пула соединений и их переиспользования
- Сложность в управлении жизненным циклом соединений

### Сценарий 3: Глобальный кэш без синхронизации

```java
@Component
public class CacheManager {
    
    private Map<String, Object> cache = new HashMap<>();
    
    public void put(String key, Object value) {
        cache.put(key, value);
    }
    
    public Object get(String key) {
        return cache.get(key);
    }
    
    public void clear() {
        cache.clear();
    }
}

@Service
public class ProductService {
    
    private CacheManager cache = new CacheManager(); // ПРОБЛЕМА ЗДЕСЬ!
    
    public Product getProduct(Long id) {
        String key = "product:" + id;
        
        // Проверяем кэш
        Object cached = cache.get(key);
        if (cached != null) {
            return (Product) cached;
        }
        
        // Загружаем из БД
        Product product = productRepository.findById(id).orElse(null);
        if (product != null) {
            cache.put(key, product);
        }
        
        return product;
    }
}

@Service
public class CategoryService {
    
    private CacheManager cache = new CacheManager(); // ПРОБЛЕМА ЗДЕСЬ!
    
    public Category getCategory(Long id) {
        String key = "category:" + id;
        
        // Проверяем кэш
        Object cached = cache.get(key);
        if (cached != null) {
            return (Category) cached;
        }
        
        // Загружаем из БД
        Category category = categoryRepository.findById(id).orElse(null);
        if (category != null) {
            cache.put(key, category);
        }
        
        return category;
    }
}
```

**Проблемы:**
- Каждый сервис имеет свой собственный кэш, что приводит к дублированию данных в памяти
- Отсутствие переиспользования кэшированных данных между сервисами
- Неэффективное использование памяти
- Отсутствие централизованного управления кэшем

### Корень проблемы

Основная проблема заключается в том, что каждый компонент приложения создает свой собственный экземпляр глобального ресурса, не осознавая, что этот ресурс должен быть единственным во всем приложении. Это приводит к:

1. **Избыточное потребление ресурсов** - создание множественных экземпляров приводит к неэффективному использованию памяти и других ресурсов
2. **Несогласованность данных** - разные экземпляры могут содержать разные данные, что приводит к ошибкам в логике приложения
3. **Сложность управления** - отсутствие централизованного управления жизненным циклом ресурса
4. **Проблемы с производительностью** - повторная инициализация ресурсов замедляет работу приложения

## Принцип работы

### Как работает паттерн

1. **Шаг 1**: Создание приватного конструктора, который предотвращает создание экземпляров извне
2. **Шаг 2**: Создание приватного статического поля для хранения единственного экземпляра
3. **Шаг 3**: Создание публичного статического метода для получения экземпляра (getInstance)
4. **Шаг 4**: Реализация ленивой инициализации и потокобезопасности

## Детальный принцип работы

### Шаг 1: Создание приватного конструктора

```java
public class AppConfig {
    
    // Приватный конструктор предотвращает создание экземпляров извне
    private AppConfig() {
        // Инициализация конфигурации
        loadConfiguration();
    }
    
    private void loadConfiguration() {
        try {
            Properties props = new Properties();
            props.load(getClass().getClassLoader().getResourceAsStream("application.properties"));
            
            // Загружаем настройки
            this.databaseUrl = props.getProperty("database.url");
            this.databaseUsername = props.getProperty("database.username");
            this.databasePassword = props.getProperty("database.password");
            this.featureFlags = parseFeatureFlags(props);
            
        } catch (IOException e) {
            throw new RuntimeException("Failed to load configuration", e);
        }
    }
    
    private Map<String, Boolean> parseFeatureFlags(Properties props) {
        Map<String, Boolean> flags = new HashMap<>();
        for (String key : props.stringPropertyNames()) {
            if (key.startsWith("feature.")) {
                flags.put(key.substring(8), Boolean.parseBoolean(props.getProperty(key)));
            }
        }
        return flags;
    }
}
```

**Ключевые моменты:**
- Конструктор объявлен как `private`, что предотвращает создание экземпляров извне
- Инициализация происходит в конструкторе
- Обработка ошибок при загрузке конфигурации

### Шаг 2: Создание приватного статического поля

```java
public class AppConfig {
    
    // Приватное статическое поле для хранения единственного экземпляра
    private static volatile AppConfig instance;
    
    // Приватные поля для хранения конфигурации
    private String databaseUrl;
    private String databaseUsername;
    private String databasePassword;
    private Map<String, Boolean> featureFlags;
    
    // Приватный конструктор
    private AppConfig() {
        loadConfiguration();
    }
    
    // Методы для доступа к конфигурации
    public String getDatabaseUrl() {
        return databaseUrl;
    }
    
    public String getDatabaseUsername() {
        return databaseUsername;
    }
    
    public String getDatabasePassword() {
        return databasePassword;
    }
    
    public boolean isFeatureEnabled(String featureName) {
        return featureFlags.getOrDefault(featureName, false);
    }
}
```

### Шаг 3: Создание публичного метода getInstance

```java
public class AppConfig {
    
    private static volatile AppConfig instance;
    
    private AppConfig() {
        loadConfiguration();
    }
    
    // Публичный статический метод для получения экземпляра
    public static AppConfig getInstance() {
        // Первая проверка (без блокировки)
        if (instance == null) {
            // Синхронизация для потокобезопасности
            synchronized (AppConfig.class) {
                // Вторая проверка (double-checked locking)
                if (instance == null) {
                    instance = new AppConfig();
                }
            }
        }
        return instance;
    }
    
    // Альтернативная реализация с ленивой инициализацией
    public static AppConfig getInstanceLazy() {
        return AppConfigHolder.INSTANCE;
    }
    
    // Приватный статический класс для ленивой инициализации
    private static class AppConfigHolder {
        private static final AppConfig INSTANCE = new AppConfig();
    }
}
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   UserService   │    │  OrderService   │    │ PaymentService  │
│                 │    │                 │    │                 │
│ 1. getInstance()│    │ 1. getInstance()│    │ 1. getInstance()│
│                 │    │                 │    │                 │
│ 2. useConfig()  │    │ 2. useConfig()  │    │ 2. useConfig()  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                                 ▼
                       ┌─────────────────┐
                       │   AppConfig     │
                       │   Singleton     │
                       │                 │
                       │ • instance      │
                       │ • getInstance() │
                       │ • config data   │
                       └─────────────────┘
```

### Ключевые преимущества

1. **Единственность экземпляра**: Гарантирует, что в приложении существует только один экземпляр класса
2. **Глобальная точка доступа**: Предоставляет удобный способ доступа к экземпляру из любой части приложения
3. **Ленивая инициализация**: Экземпляр создается только при первом обращении
4. **Потокобезопасность**: Правильная реализация обеспечивает безопасность в многопоточной среде
5. **Экономия ресурсов**: Предотвращает создание множественных экземпляров

## Конфигурация Singleton

### Конфигурация приложения

Описание конфигурируемых параметров для Singleton паттерна

**Пример конфигурации:**
```properties
# application.properties
# Настройки для Singleton конфигурации
app.config.reload.enabled=false
app.config.cache.ttl=3600
app.config.validation.enabled=true
app.config.encryption.enabled=false

# Настройки подключения к БД
database.url=jdbc:mysql://localhost:3306/mydb
database.username=user
database.password=password
database.pool.size=10

# Feature flags
feature.user-validation=true
feature.order-validation=true
feature.payment-processing=false
feature.notifications=true
```

**Альтернативные стратегии:**
- **Enum Singleton**: Использование enum для создания потокобезопасного Singleton
- **Initialization-on-demand holder**: Ленивая инициализация через приватный статический класс
- **Thread-safe Singleton**: Использование volatile и synchronized для потокобезопасности
- **Serializable Singleton**: Реализация с поддержкой сериализации

## Ключевые принципы

- **Принцип единственности**: Класс должен иметь только один экземпляр во всем приложении
- **Принцип глобального доступа**: Должна быть глобальная точка доступа к экземпляру
- **Принцип ленивой инициализации**: Экземпляр создается только при первом обращении
- **Принцип потокобезопасности**: Реализация должна быть безопасной в многопоточной среде

## Когда использовать

Singleton подходит для случаев, когда:
- Необходим единственный экземпляр класса во всем приложении
- Требуется глобальная точка доступа к экземпляру
- Нужно контролировать доступ к общему ресурсу (конфигурация, подключение к БД, кэш)
- Требуется ленивая инициализация дорогостоящего ресурса
- Необходимо обеспечить потокобезопасность доступа к общему ресурсу

## Преимущества и недостатки

### Преимущества

#### 1. **Гарантированная единственность экземпляра**
- Обеспечивает, что в приложении существует только один экземпляр класса
- Предотвращает создание множественных экземпляров, что экономит ресурсы
- Упрощает управление глобальным состоянием

#### 2. **Глобальная точка доступа**
- Предоставляет удобный способ доступа к экземпляру из любой части приложения
- Упрощает архитектуру приложения, избавляя от необходимости передавать экземпляр между компонентами
- Делает код более читаемым и понятным

#### 3. **Ленивая инициализация**
- Экземпляр создается только при первом обращении, что экономит ресурсы
- Ускоряет запуск приложения, так как не все Singleton создаются сразу
- Позволяет инициализировать ресурсы только при необходимости

#### 4. **Потокобезопасность**
- Правильная реализация обеспечивает безопасность в многопоточной среде
- Предотвращает создание множественных экземпляров в конкурентной среде
- Использует double-checked locking или другие механизмы синхронизации

#### 5. **Экономия ресурсов**
- Предотвращает создание множественных экземпляров дорогостоящих ресурсов
- Уменьшает потребление памяти и других системных ресурсов
- Особенно важно для ресурсов, которые требуют значительных затрат на инициализацию

#### 6. **Централизованное управление**
- Позволяет централизованно управлять жизненным циклом ресурса
- Упрощает мониторинг и отладку
- Обеспечивает единообразное поведение во всем приложении

### Недостатки

#### 1. **Глобальное состояние**
- Создает глобальное состояние, что может усложнить тестирование
- Затрудняет изоляцию компонентов в unit-тестах
- Может привести к скрытым зависимостям между компонентами

#### 2. **Нарушение принципа единственной ответственности**
- Singleton часто берет на себя слишком много ответственности
- Может стать "божественным объектом", который знает слишком много
- Усложняет понимание архитектуры приложения

#### 3. **Сложность тестирования**
- Трудно мокать Singleton в тестах
- Состояние Singleton может влиять на другие тесты
- Требует специальных подходов для изоляции тестов

#### 4. **Проблемы с многопоточностью**
- Неправильная реализация может привести к созданию множественных экземпляров
- Сложность в обеспечении потокобезопасности
- Возможные deadlock'и при неправильном использовании synchronized

#### 5. **Нарушение принципа инверсии зависимостей**
- Компоненты напрямую зависят от конкретной реализации Singleton
- Затрудняет замену реализации на альтернативную
- Усложняет внедрение зависимостей

#### 6. **Проблемы с сериализацией**
- При сериализации/десериализации может создаться новый экземпляр
- Требует специальной обработки для сохранения единственности
- Может привести к неожиданному поведению в распределенных системах

### Сравнение с альтернативами

| Аспект | Singleton | Dependency Injection | Service Locator |
|--------|-----------|---------------------|-----------------|
| **Сложность** | Низкая | Средняя | Средняя |
| **Производительность** | Высокая | Высокая | Средняя |
| **Надежность** | Средняя | Высокая | Средняя |
| **Масштабируемость** | Низкая | Высокая | Средняя |
| **Простота отладки** | Низкая | Высокая | Средняя |

## Тестирование

Тестирование Singleton паттерна требует особого подхода из-за глобального состояния и сложности мокирования. Важно тестировать как создание экземпляра, так и его использование в различных сценариях.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest
@Transactional
class SingletonIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private AppConfig appConfig;
    
    @Test
    void shouldUseSameConfigInstanceAcrossServices() {
        // Given
        CreateUserRequest userRequest = new CreateUserRequest("John", "john@example.com");
        CreateOrderRequest orderRequest = new CreateOrderRequest(1L, 100.0);
        
        // When
        User user = userService.createUser(userRequest);
        Order order = orderService.createOrder(orderRequest);
        
        // Then - проверяем, что используется один экземпляр конфигурации
        AppConfig config1 = AppConfig.getInstance();
        AppConfig config2 = AppConfig.getInstance();
        
        assertThat(config1).isSameAs(config2);
        assertThat(config1.getDatabaseUrl()).isEqualTo("jdbc:mysql://localhost:3306/mydb");
        assertThat(config1.isFeatureEnabled("user-validation")).isTrue();
        assertThat(config1.isFeatureEnabled("order-validation")).isTrue();
        
        // And - проверяем, что сервисы работают корректно
        assertThat(user).isNotNull();
        assertThat(user.getName()).isEqualTo("John");
        assertThat(order).isNotNull();
        assertThat(order.getAmount()).isEqualTo(100.0);
    }
    
    @Test
    void shouldHandleConcurrentAccessToSingleton() throws InterruptedException {
        // Given
        int threadCount = 10;
        CountDownLatch latch = new CountDownLatch(threadCount);
        List<AppConfig> instances = Collections.synchronizedList(new ArrayList<>());
        
        // When - создаем несколько потоков, которые одновременно получают экземпляр
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                AppConfig instance = AppConfig.getInstance();
                instances.add(instance);
                latch.countDown();
            }).start();
        }
        
        // Ждем завершения всех потоков
        latch.await(5, TimeUnit.SECONDS);
        
        // Then - проверяем, что все потоки получили один и тот же экземпляр
        assertThat(instances).hasSize(threadCount);
        AppConfig firstInstance = instances.get(0);
        
        for (AppConfig instance : instances) {
            assertThat(instance).isSameAs(firstInstance);
        }
    }
    
    @Test
    void shouldHandleConfigurationReload() {
        // Given
        AppConfig config = AppConfig.getInstance();
        String originalUrl = config.getDatabaseUrl();
        
        // When - изменяем конфигурацию
        config.reloadConfiguration();
        
        // Then - проверяем, что изменения применились
        assertThat(config.getDatabaseUrl()).isEqualTo(originalUrl); // или новое значение, если файл изменился
    }
}
```

#### TestAppConfig для тестирования

```java
@Component
@Primary
public class TestAppConfig extends AppConfig {
    
    private Map<String, Object> testConfig = new HashMap<>();
    private boolean shouldFail = false;
    
    public TestAppConfig() {
        // Инициализируем тестовую конфигурацию
        testConfig.put("database.url", "jdbc:h2:mem:testdb");
        testConfig.put("database.username", "test");
        testConfig.put("database.password", "test");
        testConfig.put("feature.user-validation", true);
        testConfig.put("feature.order-validation", true);
        testConfig.put("feature.payment-processing", false);
    }
    
    @Override
    public String getDatabaseUrl() {
        if (shouldFail) {
            throw new RuntimeException("Test failure");
        }
        return (String) testConfig.get("database.url");
    }
    
    @Override
    public String getDatabaseUsername() {
        return (String) testConfig.get("database.username");
    }
    
    @Override
    public String getDatabasePassword() {
        return (String) testConfig.get("database.password");
    }
    
    @Override
    public boolean isFeatureEnabled(String featureName) {
        return (Boolean) testConfig.getOrDefault("feature." + featureName, false);
    }
    
    public void setConfigValue(String key, Object value) {
        testConfig.put(key, value);
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
    
    public void reset() {
        testConfig.clear();
        shouldFail = false;
        // Переинициализируем тестовую конфигурацию
        testConfig.put("database.url", "jdbc:h2:mem:testdb");
        testConfig.put("database.username", "test");
        testConfig.put("database.password", "test");
        testConfig.put("feature.user-validation", true);
        testConfig.put("feature.order-validation", true);
        testConfig.put("feature.payment-processing", false);
    }
}
``` 