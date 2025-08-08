# API Gateway Pattern

## Обзор паттерна

**API Gateway** - это паттерн проектирования, который предоставляет единую точку входа для всех клиентских запросов к микросервисной системе.

### Что это такое?

API Gateway выступает в роли "фасада" для всей микросервисной архитектуры, скрывая от клиентов сложность внутренней структуры системы. Он обрабатывает все входящие запросы, маршрутизирует их к соответствующим микросервисам, агрегирует ответы и предоставляет единый API для клиентов.

### Проблема, которую решает паттерн

В микросервисной архитектуре часто возникает ситуация, когда клиентам нужно взаимодействовать с множеством различных сервисов. Например:

1. **Сценарий**: Мобильное приложение должно получить данные пользователя, его заказы и рекомендации
2. **Требование**: Нужно сделать один запрос и получить все необходимые данные
3. **Проблема**: Клиент должен знать адреса всех сервисов и делать множественные запросы
4. **Результат**: Сложность клиентского кода, проблемы с производительностью и безопасностью

## Детальный анализ проблемы

### Сценарий 1: Прямое обращение к микросервисам

```java
@Service
public class MobileAppService {
    
    @Autowired
    private UserServiceClient userServiceClient;
    
    @Autowired
    private OrderServiceClient orderServiceClient;
    
    @Autowired
    private RecommendationServiceClient recommendationServiceClient;
    
    public UserDashboardResponse getUserDashboard(String userId) {
        // 1. Получаем данные пользователя
        User user = userServiceClient.getUser(userId);
        
        // 2. Получаем заказы пользователя (ПРОБЛЕМА ЗДЕСЬ!)
        List<Order> orders = orderServiceClient.getUserOrders(userId);
        
        // 3. Получаем рекомендации
        List<Recommendation> recommendations = recommendationServiceClient.getRecommendations(userId);
        
        return new UserDashboardResponse(user, orders, recommendations);
    }
}
```

**Проблемы:**
- Клиент должен знать адреса всех микросервисов
- Множественные сетевые запросы снижают производительность
- Нет централизованной аутентификации и авторизации
- Сложность обработки ошибок и retry логики

### Сценарий 2: Использование Load Balancer

```java
@RestController
public class DirectServiceController {
    
    @Value("${user.service.url}")
    private String userServiceUrl;
    
    @Value("${order.service.url}")
    private String orderServiceUrl;
    
    @GetMapping("/api/user/{userId}")
    public ResponseEntity<User> getUser(@PathVariable String userId) {
        // Прямой запрос к сервису через Load Balancer
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.getForEntity(userServiceUrl + "/users/" + userId, User.class);
    }
}
```

**Проблемы:**
- Нет агрегации данных из разных сервисов
- Отсутствует централизованная обработка ошибок
- Нет возможности кэширования и rate limiting
- Сложность мониторинга и логирования

### Сценарий 3: Использование Service Mesh

```java
@Service
public class ServiceMeshClient {
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    public UserDashboardResponse getUserDashboard(String userId) {
        // Получаем адреса сервисов через Service Discovery
        ServiceInstance userService = discoveryClient.getInstances("user-service").get(0);
        ServiceInstance orderService = discoveryClient.getInstances("order-service").get(0);
        
        // Прямые запросы к сервисам (ПРОБЛЕМА ЗДЕСЬ!)
        User user = callService(userService.getUri() + "/users/" + userId, User.class);
        List<Order> orders = callService(orderService.getUri() + "/users/" + userId + "/orders", List.class);
        
        return new UserDashboardResponse(user, orders);
    }
}
```

**Проблемы:**
- Сложность конфигурации Service Mesh
- Отсутствие единого API для клиентов
- Нет возможности трансформации данных
- Сложность реализации бизнес-логики на уровне API

### Корень проблемы

Основная проблема заключается в том, что **клиенты вынуждены взаимодействовать с множеством микросервисов напрямую**. Это приводит к:

1. **Сложности клиентского кода** - клиенты должны знать архитектуру системы
2. **Проблемам безопасности** - нет централизованной аутентификации и авторизации
3. **Низкой производительности** - множественные сетевые запросы
4. **Сложности мониторинга** - нет единой точки для сбора метрик

### Как работает паттерн

1. **Единая точка входа**: Все запросы проходят через API Gateway
2. **Маршрутизация**: Gateway направляет запросы к соответствующим микросервисам
3. **Агрегация**: Объединяет ответы от нескольких сервисов в один
4. **Трансформация**: Преобразует данные в формат, удобный для клиентов

## Детальный принцип работы

### Шаг 1: Конфигурация маршрутов

```java
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // Маршрут для пользователей
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .rewritePath("/api/users/(?<segment>.*)", "/users/${segment}")
                    .addRequestHeader("X-Response-Time", System.currentTimeMillis() + "")
                )
                .uri("lb://user-service"))
            
            // Маршрут для заказов
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .rewritePath("/api/orders/(?<segment>.*)", "/orders/${segment}")
                    .addRequestHeader("X-Response-Time", System.currentTimeMillis() + "")
                )
                .uri("lb://order-service"))
            
            // Маршрут для агрегированных данных
            .route("dashboard-service", r -> r
                .path("/api/dashboard/**")
                .filters(f -> f
                    .rewritePath("/api/dashboard/(?<segment>.*)", "/dashboard/${segment}")
                )
                .uri("lb://dashboard-service"))
            .build();
    }
}
```

**Ключевые моменты:**
- Каждый маршрут имеет уникальное имя
- Используется паттерн перезаписи путей
- Добавляются кастомные заголовки для мониторинга
- Используется Load Balancer для распределения нагрузки

### Шаг 2: Агрегация данных

```java
@Service
public class DashboardAggregationService {
    
    @Autowired
    private WebClient.Builder webClientBuilder;
    
    public UserDashboardResponse getUserDashboard(String userId) {
        // 1. Параллельные запросы к микросервисам
        Mono<User> userMono = webClientBuilder.build()
            .get()
            .uri("http://user-service/users/{userId}", userId)
            .retrieve()
            .bodyToMono(User.class);
        
        Mono<List<Order>> ordersMono = webClientBuilder.build()
            .get()
            .uri("http://order-service/users/{userId}/orders", userId)
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<List<Order>>() {});
        
        Mono<List<Recommendation>> recommendationsMono = webClientBuilder.build()
            .get()
            .uri("http://recommendation-service/users/{userId}/recommendations", userId)
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<List<Recommendation>>() {});
        
        // 2. Агрегация результатов
        return Mono.zip(userMono, ordersMono, recommendationsMono)
            .map(tuple -> new UserDashboardResponse(tuple.getT1(), tuple.getT2(), tuple.getT3()))
            .block();
    }
}
```

### Шаг 3: Аутентификация и авторизация

```java
@Component
public class AuthenticationFilter implements GlobalFilter {
    
    @Autowired
    private JwtTokenProvider jwtTokenProvider;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // 1. Извлекаем токен из заголовка
        String token = getTokenFromRequest(request);
        
        if (token != null && jwtTokenProvider.validateToken(token)) {
            // 2. Извлекаем информацию о пользователе
            String userId = jwtTokenProvider.getUserIdFromToken(token);
            String userRole = jwtTokenProvider.getUserRoleFromToken(token);
            
            // 3. Добавляем информацию в заголовки
            ServerHttpRequest modifiedRequest = request.mutate()
                .header("X-User-ID", userId)
                .header("X-User-Role", userRole)
                .build();
            
            return chain.filter(exchange.mutate().request(modifiedRequest).build());
        }
        
        // 4. Если токен недействителен, возвращаем ошибку
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }
    
    private String getTokenFromRequest(ServerHttpRequest request) {
        String bearerToken = request.getHeaders().getFirst("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### Шаг 4: Rate Limiting и кэширование

```java
@Component
public class RateLimitingFilter implements GlobalFilter {
    
    private final RateLimiter rateLimiter = RateLimiter.create(100.0); // 100 запросов в секунду
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1. Проверяем лимит запросов
        if (!rateLimiter.tryAcquire()) {
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        }
        
        // 2. Добавляем заголовки для мониторинга
        ServerHttpRequest modifiedRequest = exchange.getRequest().mutate()
            .header("X-Rate-Limit-Remaining", String.valueOf(rateLimiter.getRate()))
            .build();
        
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
}
```

### Архитектурная схема

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Client App    │    │   API Gateway   │    │  Microservices  │
│                 │    │                 │    │                 │
│ 1. HTTP Request │───▶│ 2. Auth/Route   │───▶│ 3. User Service │
│                 │    │                 │    │                 │
│ 4. Response     │◀───│ 5. Aggregate    │◀───│ 4. Order Service│
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │  Rate Limiting  │
                       │   Caching       │
                       │   Monitoring    │
                       └─────────────────┘
```

### Ключевые преимущества

1. **Единая точка входа**: Все запросы проходят через один компонент
2. **Централизованная безопасность**: Аутентификация и авторизация в одном месте
3. **Агрегация данных**: Объединение ответов от нескольких сервисов
4. **Кэширование**: Улучшение производительности за счет кэширования
5. **Мониторинг**: Единая точка для сбора метрик и логов

### Конфигурация маршрутизации

API Gateway поддерживает различные стратегии маршрутизации:

**Пример конфигурации:**
```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - RewritePath=/api/users/(?<segment>.*), /users/${segment}
            - AddRequestHeader=X-Response-Time, ${timestamp}
        
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - RewritePath=/api/orders/(?<segment>.*), /orders/${segment}
```

**Альтернативные стратегии маршрутизации:**
- **Path-based routing**: Маршрутизация по пути запроса
- **Header-based routing**: Маршрутизация по заголовкам
- **Weight-based routing**: Распределение нагрузки по весам
- **Circuit breaker routing**: Маршрутизация с защитой от сбоев

### Ключевые принципы

- **Единая точка входа**: Все клиентские запросы проходят через Gateway
- **Прозрачность**: Клиенты не знают о внутренней архитектуре
- **Безопасность**: Централизованная аутентификация и авторизация
- **Производительность**: Кэширование и оптимизация запросов
- **Мониторинг**: Единая точка для сбора метрик

### Когда использовать

API Gateway подходит для случаев, когда:
- У вас есть микросервисная архитектура
- Клиентам нужен единый API для доступа к сервисам
- Требуется централизованная безопасность
- Нужна агрегация данных из нескольких сервисов
- Важны производительность и кэширование

## Преимущества и недостатки

### Преимущества

#### 1. **Единая точка входа**
- Клиенты взаимодействуют только с одним компонентом
- Скрывает сложность микросервисной архитектуры
- Упрощает клиентский код

#### 2. **Централизованная безопасность**
- Аутентификация и авторизация в одном месте
- Единая политика безопасности для всех сервисов
- Упрощение управления токенами и сессиями

#### 3. **Агрегация данных**
- Объединение ответов от нескольких микросервисов
- Уменьшение количества сетевых запросов
- Оптимизация производительности клиентов

#### 4. **Кэширование и оптимизация**
- Кэширование часто запрашиваемых данных
- Сжатие ответов для экономии трафика
- Оптимизация размера передаваемых данных

#### 5. **Мониторинг и аналитика**
- Единая точка для сбора метрик
- Централизованное логирование
- Анализ производительности и использования

#### 6. **Гибкость и масштабируемость**
- Легкое добавление новых сервисов
- Возможность изменения внутренней архитектуры
- Горизонтальное масштабирование Gateway

### Недостатки

#### 1. **Единая точка отказа**
- Если Gateway недоступен, вся система не работает
- Потенциальные узкие места производительности
- Сложность обеспечения высокой доступности

#### 2. **Дополнительная сложность**
- Требуется дополнительный компонент для поддержки
- Сложность конфигурации и настройки
- Необходимость мониторинга и обслуживания

#### 3. **Потенциальные проблемы производительности**
- Дополнительная задержка при обработке запросов
- Возможные узкие места при высокой нагрузке
- Сложность оптимизации для разных типов запросов

#### 4. **Сложность отладки**
- Трудно отследить проблемы в цепочке запросов
- Сложность диагностики проблем производительности
- Необходимость дополнительных инструментов мониторинга

#### 5. **Зависимость от технологии**
- Привязка к конкретной технологии Gateway
- Сложность миграции на другую платформу
- Зависимость от вендора решения

#### 6. **Проблемы с версионированием API**
- Сложность управления версиями API
- Необходимость обратной совместимости
- Проблемы с эволюцией API

### Сравнение с альтернативами

| Аспект | API Gateway | Direct Service Calls | Service Mesh |
|--------|-------------|---------------------|--------------|
| **Сложность** | Средняя | Низкая | Высокая |
| **Производительность** | Средняя | Высокая | Высокая |
| **Безопасность** | Высокая | Низкая | Средняя |
| **Масштабируемость** | Высокая | Низкая | Высокая |
| **Простота отладки** | Средняя | Высокая | Низкая |

## Тестирование

Интеграционные тесты показывают полный цикл работы API Gateway паттерна и помогают понять, как все компоненты работают вместе.

### Интеграционные тесты

#### Полный flow тест

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiGatewayIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private TestUserService testUserService;
    
    @Test
    void shouldRouteUserRequestToUserService() {
        // Given
        String userId = "user123";
        User expectedUser = new User(userId, "John Doe", "john@example.com");
        testUserService.setUser(expectedUser);
        
        // When
        ResponseEntity<User> response = restTemplate.getForEntity(
            "/api/users/" + userId, User.class);
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isEqualTo(expectedUser);
        assertThat(testUserService.getRequestCount()).isEqualTo(1);
    }
    
    @Test
    void shouldAggregateDashboardData() {
        // Given
        String userId = "user123";
        User user = new User(userId, "John Doe", "john@example.com");
        List<Order> orders = List.of(new Order("order1", userId));
        List<Recommendation> recommendations = List.of(new Recommendation("rec1"));
        
        testUserService.setUser(user);
        testOrderService.setOrders(orders);
        testRecommendationService.setRecommendations(recommendations);
        
        // When
        ResponseEntity<UserDashboardResponse> response = restTemplate.getForEntity(
            "/api/dashboard/" + userId, UserDashboardResponse.class);
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        UserDashboardResponse dashboard = response.getBody();
        assertThat(dashboard.getUser()).isEqualTo(user);
        assertThat(dashboard.getOrders()).isEqualTo(orders);
        assertThat(dashboard.getRecommendations()).isEqualTo(recommendations);
    }
    
    @Test
    void shouldHandleAuthentication() {
        // Given
        String invalidToken = "invalid-token";
        
        // When
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(invalidToken);
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        ResponseEntity<String> response = restTemplate.exchange(
            "/api/users/user123", HttpMethod.GET, entity, String.class);
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.UNAUTHORIZED);
    }
    
    @Test
    void shouldApplyRateLimiting() {
        // Given
        String userId = "user123";
        
        // When - делаем много запросов подряд
        for (int i = 0; i < 150; i++) {
            ResponseEntity<String> response = restTemplate.getForEntity(
                "/api/users/" + userId, String.class);
            
            // Then - после превышения лимита должны получить 429
            if (i >= 100) {
                assertThat(response.getStatusCode()).isEqualTo(HttpStatus.TOO_MANY_REQUESTS);
            }
        }
    }
}
```

#### TestUserService для тестирования

```java
@Component
@Primary
public class TestUserService {
    
    private User user;
    private int requestCount = 0;
    private boolean shouldFail = false;
    
    @GetMapping("/users/{userId}")
    public ResponseEntity<User> getUser(@PathVariable String userId) {
        requestCount++;
        
        if (shouldFail) {
            throw new RuntimeException("Test failure");
        }
        
        if (user != null && user.getId().equals(userId)) {
            return ResponseEntity.ok(user);
        }
        
        return ResponseEntity.notFound().build();
    }
    
    public void setUser(User user) {
        this.user = user;
    }
    
    public int getRequestCount() {
        return requestCount;
    }
    
    public void clear() {
        requestCount = 0;
        user = null;
        shouldFail = false;
    }
    
    public void setShouldFail(boolean shouldFail) {
        this.shouldFail = shouldFail;
    }
}
``` 