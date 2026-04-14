# Spring Boot Cheat Sheet — 2V0-72.22

---

## ⚡ Быстрые формулы (запомни сначала эти)

| Концепция | Суть |
|---|---|
| `ApplicationContext` | eager, events, full features, расширяет BeanFactory |
| `BeanFactory` | минимум, lazy, базовый IoC |
| `@Configuration` | CGLIB proxy, singleton гарантирован |
| `prototype` | Spring управляет только созданием, **НЕ** destroy |
| `@Repository` | exception translation → DataAccessException |
| Constructor circular dependency | **FAIL** на старте |
| `@PreDestroy` | только singleton + shutdown (prototype игнорируется) |

```
BeanFactory (lazy)
    ↑
ApplicationContext (eager)
    ↑
ConfigurableApplicationContext
    ↑
WebApplicationContext
```

---

## 💉 Инъекция зависимостей

**Field injection:**
- ✅ меньше кода
- ❌ нельзя сделать поля `final`
- ❌ сложнее тестировать (нужен Spring контекст или reflection)
- ❌ скрывает зависимости

**Constructor injection:**
- ✅ безопасно, поля могут быть `final`
- ✅ легко тестировать (передаёшь mock в конструктор)
- ✅ явные зависимости видны
- ➖ больше кода (решается Lombok `@RequiredArgsConstructor`)

**Порядок поиска бина при `@Autowired`:**
1. Точное совпадение по типу
2. `@Primary`
3. `@Qualifier("name")`

> Если несколько кандидатов и нет `@Primary`/`@Qualifier` → `NoUniqueBeanDefinitionException`

---

## 🔄 Жизненный цикл бина

**Инициализация:** `Constructor` → `DI` → `@PostConstruct` → `afterPropertiesSet()` → `custom init-method`

> Запомни: **C → DI → Post → After → Custom**

**Уничтожение:** `@PreDestroy` → `destroy()` → `custom destroy-method`

> Запомни: **Pre → Destroy → Custom**

**`@PostConstruct` и `@PreDestroy`:**
- возвращают `void`
- без аргументов
- только singleton (для prototype `@PreDestroy` не вызывается)

**Два способа создать бин:**
1. `@Component` / `@Service` / `@Repository` / `@Controller`
2. `@Configuration` + `@Bean`

`@Lazy` — создать бин только при первом обращении (не при старте контекста)

---

## 🔭 Scopes

| Scope | Описание |
|---|---|
| `singleton` | один на весь контекст (default) |
| `prototype` | новый при каждом запросе; Spring **НЕ** управляет destroy |
| `request` | один на HTTP запрос (только web-aware контекст) |
| `session` | один на HTTP сессию (только web-aware контекст) |
| `application` | один на ServletContext (все пользователи, весь lifecycle) |
| `websocket` | один на WebSocket сессию (требует Spring WebSocket + STOMP) |

---

## 🚀 @SpringBootApplication = мета-аннотация

```java
@SpringBootApplication =
  @SpringBootConfiguration   // это @Configuration для Boot
  @EnableAutoConfiguration   // включает авто-конфигурацию
  @ComponentScan             // сканирует текущий пакет и sub-пакеты
```

**`@Configuration`:**
- meta-annotated с `@Component`
- CGLIB proxy → singleton-поведение `@Bean` методов гарантировано
- `@Configuration(proxyBeanMethods = false)` → без CGLIB, класс может быть `final`

> Почему `@Bean` методы не должны быть `final`? Spring должен переопределить их через CGLIB subclass.

---

## 🤖 Auto-Configuration (@Conditional механизм)

> Зависит от: **Classpath + Existing Beans + Configuration Properties**

| Аннотация | Когда активируется |
|---|---|
| `@ConditionalOnClass(X.class)` | класс есть в classpath |
| `@ConditionalOnMissingClass("x.X")` | класса **НЕТ** в classpath |
| `@ConditionalOnBean(X.class)` | бин есть в контексте |
| `@ConditionalOnMissingBean(X.class)` | бина **НЕТ** — default fallback |
| `@ConditionalOnProperty(name="x", havingValue="true")` | по значению property |
| `@ConditionalOnExpression("SpEL")` | сложная логика через SpEL |
| `@ConditionalOnWebApplication` | только для web-приложений |
| `@ConditionalOnNotWebApplication` | CLI / batch режим |
| `@ConditionalOnJava(JavaVersion.XX)` | по версии JDK |
| `@ConditionalOnResource("classpath:schema.sql")` | если файл существует |
| `@ConditionalOnSingleCandidate(X.class)` | если только один кандидат бина |

Упорядочить авто-конфигурации: `@AutoConfigureOrder` / `@AutoConfigureAfter` / `@AutoConfigureBefore`

---

## 📋 Приоритет Property Sources (от высшего к низшему)

1. DevTools global settings (`$HOME/.config/spring-boot`)
2. `@TestPropertySource` (в тестах)
3. `properties` атрибут `@SpringBootTest`
4. **Command-line arguments** ← самые важные в рантайме
5. `SPRING_APPLICATION_JSON` (inline JSON)
6. `ServletConfig` init parameters
7. `ServletContext` init parameters
8. JNDI (`java:comp/env`)
9. Java system properties (`System.getProperties()`)
10. OS environment variables
11. `RandomValuePropertySource` (`random.*`)
12. **Config data** (`application.properties` / `yml`) ← обычно здесь
13. `@PropertySource` на `@Configuration` классах
14. Default properties (`SpringApplication.setDefaultProperties()`)

**Порядок `application.properties` файлов** (от высшего приоритета):

| Приоритет | Путь |
|---|---|
| 1 | `/config/` рабочая директория |
| 2 | `/` рабочая директория |
| 3 | `classpath:/config/` |
| 4 | `classpath:/` ← стандартный `src/main/resources` |

```properties
@Value("${app.name}")                         # точечная инъекция
@ConfigurationProperties(prefix = "example")  # биндинг целого объекта
```

---

## 🎯 AOP

| Термин | Описание |
|---|---|
| Join point | точка выполнения (в Spring AOP — **всегда метод**) |
| Pointcut | предикат, который матчит join points (где применять) |
| Advice | что делать в join point (`@Before` / `@After` / `@Around` и т.д.) |
| Aspect | модуль = Pointcut + Advice (cross-cutting concerns) |

AfterReturning
Around

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {}
```

**Pointcut expressions:**

| Expression | Что матчит |
|---|---|
| `execution(...)` | выполнение методов |
| `within(com.xyz.service..*)` | все join points внутри пакета |
| `this(com.xyz.MyInterface)` | proxy является экземпляром типа |
| `target(com.xyz.MyClass)` | target объект является экземпляром типа |
| `args(java.io.Serializable)` | аргументы метода подходят по типу |
| `@annotation(com.xyz.Loggable)` | метод помечен аннотацией |

**JoinPoint как аргумент Advice:**
- `@Before` / `@After` / `@AfterReturning` / `@AfterThrowing` → `JoinPoint`
- `@Around` → `ProceedingJoinPoint` (нужно вызвать `.proceed()`!)

---

## 💳 Транзакции

```java
@Transactional(
    propagation  = Propagation.REQUIRED,  // default
    isolation    = Isolation.DEFAULT,
    timeout      = -1,
    readOnly     = false,
    rollbackFor  = RuntimeException.class,
    noRollbackFor = {}
)
```

**Propagation:**

| Тип | Если транзакция есть | Если нет | Создаёт? | Приостанавливает? | Exception? |
|---|---|---|---|---|---|
| `REQUIRED` | использует | создаёт | если нет | ❌ | ❌ |
| `REQUIRES_NEW` | приостанавливает и создаёт | создаёт | ✅ всегда | ✅ | ❌ |
| `SUPPORTS` | использует | без транзакции | ❌ | ❌ | ❌ |
| `MANDATORY` | использует | **exception** | ❌ | ❌ | ✅ если нет |
| `NEVER` | **exception** | без транзакции | ❌ | ❌ | ✅ если есть |
| `NOT_SUPPORTED` | приостанавливает | без транзакции | ❌ | ✅ | ❌ |
| `NESTED` | вложенная (savepoint) | как REQUIRED | вложенную | ❌ | ❌ |

> 🔥 **Создают новую:** REQUIRED (если нет), REQUIRES_NEW (всегда), NESTED (вложенную)
> ⛔ **Бросают exception:** MANDATORY (если нет транзакции), NEVER (если есть)
> ⏸ **Приостанавливают:** REQUIRES_NEW, NOT_SUPPORTED

**TransactionManager:**
- 1 БД → `JpaTransactionManager`
- 2+ БД → `JtaTransactionManager` (Two-Phase Commit: prepare → commit/rollback)

**TransactionTemplate** — когда нельзя использовать proxy:
- нужна транзакция внутри одного метода
- fine-grained контроль
- самовызов (self-invocation) — proxy не перехватывает

---

## 🗄️ Spring Data JPA

`spring-boot-starter-data-jpa` автоматически конфигурирует: `DataSource`, `EntityManagerFactory`, `JpaTransactionManager` (делает `HibernateJpaAutoConfiguration`)

`@EnableJpaRepositories` — сканирует пакет аннотированного класса и sub-пакеты

**Query method keywords:**

| Keyword | Возвращает |
|---|---|
| `find/read/get/query/search/streamBy` | сущности / коллекции |
| `existsBy` | `boolean` |
| `countBy` | число |
| `deleteBy` / `removeBy` | `void` или count |
| `...First<N>...` / `...Top<N>...` | ограничение результата |
| `...Distinct...` | уникальные результаты |

Ключевые слова: `And`, `Or`, `IsNull`, `IsNotNull`, `LessThan`, `GreaterThan`, `StartingWith`, `EndingWith`
Spring Data's query mechanism supports a rich vocabulary of keywords to express criteria in method names, including IsNull, 
Between, LessThan, GreaterThan, Like, In, and ExistsBy.

- `@Entity` → JPA (реляционная БД)
- `@NodeEntity` → Spring Data Neo4j (графовая БД)
- `@Param` не нужен для позиционных параметров (`?1`), нужен для именованных (`:name`)

---

## 🔌 JdbcTemplate

> Стартер: `spring-boot-starter-jdbc` (не `data-jpa`!)

`query` / `queryForObject` / `queryForList` → для read-only операций (рекомендуется)

**Callback интерфейсы:**

| Интерфейс | Назначение |
|---|---|
| `RowMapper` | строка ResultSet → объект |
| `ResultSetExtractor` | весь ResultSet → объект (полный контроль) |
| `RowCallbackHandler` | строка → void |
| `PreparedStatementCreator/Setter` | кастомный PreparedStatement |
| `CallableStatementCreator` | хранимые процедуры |

Возвращаемые типы: `T`, `List<T>`, `Map<String, Object>`, `List<Map<String, Object>>`

---

## 🌐 Spring MVC

```
HTTP запрос → DispatcherServlet → HandlerMapping → HandlerAdapter → Controller
                                                                        ↓
                                              View ← ViewResolver ← ModelAndView
```

**WebMvcAutoConfiguration включает:**
- ✅ Static resources
- ✅ HttpMessageConverters (Jackson, XML, etc.)
- ✅ Content negotiation
- ✅ Default ViewResolvers
- ✅ Jackson auto-config

**`@RequestMapping`:**
- по умолчанию **GET** (если не указан `method`)
- необязательный элемент называется `params`, **НЕ** `requestParams`
- можно несколько URL: `@RequestMapping({"/a", "/b"})`

**Параметры контроллера — что Spring инжектит:**

| Аннотация | Откуда |
|---|---|
| `@PathVariable` | `/orders/{id}` |
| `@RequestParam` | `/orders?id=15` |
| `@RequestBody` | тело запроса (JSON) |
| `@RequestHeader` | HTTP header |
| `@CookieValue` | Cookie |
| `@ModelAttribute` | Form data |
| `@AuthenticationPrincipal` | текущий пользователь (Spring Security) |
| `@SessionAttribute` | HTTP session |
| `HttpServletRequest/Response` | напрямую |
| `Principal`, `Locale`, `BindingResult` | напрямую |

**ViewResolvers:**

| Класс | Для чего |
|---|---|
| `InternalResourceViewResolver` | JSP |
| `ThymeleafViewResolver` | Thymeleaf |
| `BeanNameViewResolver` | ищет View-бин по имени |
| `FreeMarkerViewResolver` | FreeMarker |
| `ContentNegotiatingViewResolver` | делегирует другим по Accept header |

**Default HttpMessageConverters:** `ByteArrayHttpMessageConverter`, `StringHttpMessageConverter`, `ResourceHttpMessageConverter`, `MappingJackson2HttpMessageConverter` (если Jackson в classpath), `Jaxb2RootElementHttpMessageConverter` (если JAXB2), `FormHttpMessageConverter`

`@ResponseStatus` на классе → применится ко **всем** методам контроллера

---

## 🔒 Spring Security

**Основные концепции:**

| Термин | Описание |
|---|---|
| Principal | кто действует (user, device, system) |
| Authentication | подтверждение что principal существует |
| Authorization | что principal может делать |
| Authority | разрешение / роль |
| Secured Resource | что защищаем |

**Компоненты:**

| Компонент | Назначение |
|---|---|
| `UserDetails` | данные пользователя |
| `UserDetailsService` | загружает пользователя |
| `Authentication` | хранит результат аутентификации |
| `GrantedAuthority` | разрешения |
| `SecurityContext` | хранит Authentication (thread-local) |
| `SecurityContextHolder` | доступ к SecurityContext из любого места |

UserDetailsService
The core strategy interface used by the DaoAuthenticationProvider to retrieve user details from a data source (database, LDAP, etc.) given a username.

**Цепочка фильтров:**
```
Servlet Container → DelegatingFilterProxy → FilterChainProxy
→ SecurityFilterChain (по URL) → Spring Security Filters
```

> LDAP = identity store | Basic/Form/OAuth = authentication mechanism

**AuthenticationProvider:**
- `DaoAuthenticationProvider` → UserDetailsService + PasswordEncoder → Authentication
- `AnonymousAuthenticationProvider` → AnonymousAuthenticationToken
- 
  The AuthenticationProvider receives an unauthenticated Authentication object (e.g., UsernamePasswordAuthenticationToken) and 
- returns a fully authenticated one after successfully verifying the credentials and loading the user's authorities.
- AuthenticationProvider is a key component in the AuthenticationManager's process. The AuthenticationManager can delegate 
- to multiple providers (e.g., one for form login, one for LDAP) until one successfully authenticates the user.

**Получить текущего пользователя:**
```java
@AuthenticationPrincipal UserDetails user  // в параметре контроллера
SecurityContextHolder.getContext().getAuthentication()  // программно
```

**Spring Security 6+:**
```java
@EnableMethodSecurity  // флаги: securedEnabled, jsr250Enabled, prePostEnabled
requestMatchers()      // вместо deprecated antMatchers() и mvcMatchers()
```

**Аннотации метода:**

| Аннотация | Проверка роли | Аргументы метода | SpEL | Вызов бинов |
|---|---|---|---|---|
| `@Secured` | ✔ | ✖ | ✖ | ✖ |
| `@PreAuthorize` | ✔ | ✔ | ✔ | ✔ |
| `@PostAuthorize` | ✔ | ✔ | ✔ | ✔ |
| `@PreFilter` | — | фильтрует коллекцию до вызова | ✔ | ✔ |
| `@PostFilter` | — | фильтрует коллекцию после вызова | ✔ | ✔ |

```java
@PreAuthorize("#username == authentication.name")
```

**`@WithMockUser`** — для тестов: создаёт mock Authentication в SecurityContext.
- ❌ НЕ `@Mock` (Mockito, не знает о SecurityContext)
- ❌ НЕ `@MockBean` (заменяет бин, не трогает SecurityContext)

---
Clickjacking Prevention
CSRF (Cross-Site Request Forgery) Protection



## 🧪 Тесты

| JUnit версия | Аннотация |
|---|---|
| JUnit 4 | `@RunWith(SpringRunner.class)` |
| JUnit 5 | `@ExtendWith(SpringExtension.class)` |

> Slice-аннотации уже включают `@ExtendWith(SpringExtension.class)` автоматически.
> `@TestConfiguration` — НЕ включает. Только регистрирует бины.

**`@Mock` vs `@Spy`:**

| | `@Mock` | `@Spy` |
|---|---|---|
| Объект | полностью фейковый | реальный |
| Методы | ничего не делают (null/0) | выполняются |
| Стаббинг | нужно всё стаббить | можно выборочно |
| Синтаксис | `when().thenReturn()` | `doReturn().when()` |

**Slice-аннотации:**

| Аннотация | Загружает | Embedded DB | Rollback |
|---|---|---|---|
| `@SpringBootTest` | весь контекст | ❌ | ❌ (нужен `@Transactional`) |
| `@WebMvcTest` | только MVC, MockMvc | ❌ | ❌ |
| `@DataJpaTest` | JPA + Repositories | ✅ H2 | ✅ |
| `@JdbcTest` | JdbcTemplate + DataSource | ✅ | ✅ |
| `@JsonTest` | ObjectMapper + serializers | ❌ | ❌ |
| `@RestClientTest` | RestTemplate/WebClient + Jackson | ❌ | ❌ |

**`@SpringBootTest` webEnvironment:**

| Режим | Описание |
|---|---|
| `MOCK` | mock servlet (default) |
| `RANDOM_PORT` | реальный сервер на случайном порту |
| `DEFINED_PORT` | реальный сервер на `server.port` |
| `NONE` | без web контекста |

**`spring-boot-starter-test` включает:**
JUnit, Spring Test, Spring Boot Test, AssertJ, Hamcrest, Mockito, JSONassert, JsonPath, XMLUnit

**НЕ включает:** Testcontainers, RestAssured, WireMock, H2 (автоматически)

---

## 📊 Spring Actuator

```properties
# Включить shutdown
management.endpoint.shutdown.enabled=true
management.endpoints.web.exposure.include=shutdown

# Открыть все
management.endpoints.web.exposure.include=*
```

**Health статусы:**

| Статус | HTTP код |
|---|---|
| UP | 200 |
| DOWN | 503 |
| OUT_OF_SERVICE | 503 |
| UNKNOWN | 200 |

Web endpoints: `heapdump`, `jolokia`, `logfile`, `prometheus`

Метрики Micrometer: `Gauge`, `Timer`, `Counter`, `DistributionSummary`

`/actuator/metrics/{metricName}` теги: `method`, `status`, `uri`, `exception`, `outcome`

`/actuator/loggers` → список логгеров (по умолчанию JMX; для HTTP нужно включить явно)

Протоколы: HTTP/HTTPS, JMX, SSH, RMI

---

## 📦 JAR типы

| Тип | Содержимое | Executable |
|---|---|---|
| Original jar | только твой код | ❌ |
| Fat jar (uber jar) | код + все зависимости + embedded Tomcat | ✅ `java -jar app.jar` |
| Layered jar | слои: deps / loader / snapshot / app | ✅ + быстрый Docker rebuild |

---

## 🧩 Spring Boot Starters

| Стартер | Что подключает |
|---|---|
| `spring-boot-starter-web` | Spring MVC + Jackson + Validation + Embedded Tomcat |
| `spring-boot-starter-data-jpa` | JPA + Hibernate + DataSource + EntityManagerFactory |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit + Mockito + AssertJ + Hamcrest + JsonPath |
| `spring-boot-starter-jdbc` | JdbcTemplate + DataSource (без JPA) |
| `spring-boot-starter-hateoas` | HAL support + RepresentationModel + Jackson HAL module |

**DevTools (`spring-boot-devtools`):**
- авто-рестарт при изменении кода
- отключает кэш шаблонов (Thymeleaf, Freemarker и т.д.)
- включает LiveReload
- самый высокий приоритет для property sources

---

## 🗒️ Разное — важно на экзамене

**`BeanFactoryPostProcessor`** → метод `postProcessBeanFactory()`
- вызывается **после** загрузки bean definitions, **до** создания бинов
- пример: `PropertySourcesPlaceholderConfigurer`

**`BeanPostProcessor`** → вызывается до и после инициализации каждого бина
- методы: `postProcessBeforeInitialization()` / `postProcessAfterInitialization()`

`@Profile("prod")` — только для бинов (`@Component` / `@Bean`), **не** для полей

**SpEL:** интерпретируемо (default) или компилируемо (`SpelCompilerMode`). Используется в `@PreAuthorize`, `@Value`, `@ConditionalOnExpression`

**REST принципы (6):** Client–Server, Stateless, Cacheable, Uniform Interface, Layered System, Code-on-Demand (опционально)

**Паттерны микросервисов:** Saga pattern, Event-driven architecture, Outbox pattern

**Logging default:** Commons Logging → SLF4J → Logback. Поддерживает: Java Util Logging, Log4J2, Logback

🔗 [2V0-72.22 exam](https://www.vmware.com/education/certification/vcp-spring-exam.html)

---

## 🪤 Экзаменационные ловушки — часто путают

### 1. `@Configuration` vs `@Component`
`@Component` тоже создаёт бины, **НО** без CGLIB proxy. Если внутри `@Component` один `@Bean` метод вызывает другой — Spring **не** перехватит → получишь новый объект, не singleton!
`@Configuration` → CGLIB → вызов другого `@Bean` метода возвращает тот же singleton.

### 2. `@Transactional` self-invocation (самовызов)
Если метод внутри того же класса вызывает другой `@Transactional` метод — транзакция **НЕ** создастся! Proxy не перехватывает внутренние вызовы.
> Решение: `TransactionTemplate` или вынести метод в отдельный бин.

### 3. prototype + `@PreDestroy`
Spring вызывает `@PreDestroy` только для singleton бинов. Для prototype — **никогда**.

### 4. `@SpringBootTest` не делает rollback
- `@DataJpaTest` → делает rollback ✅
- `@SpringBootTest` → **НЕ** делает ❌ (нужно явно добавить `@Transactional`)

### 5. `@WebMvcTest` не загружает `@Service`
Сервисы нужно мокировать через `@MockBean`, иначе контекст не поднимется.

### 6. `@ExtendWith` не нужен со slice-аннотациями
`@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest` уже включают `SpringExtension`.
`@TestConfiguration` — **НЕ** включает. Только регистрирует бины.

### 7. MANDATORY vs NEVER — противоположности
- `MANDATORY` → нет транзакции = exception
- `NEVER` → есть транзакция = exception

### 8. `@RequestMapping` по умолчанию — GET
`params` называется `params`, **НЕ** `requestParams`.

### 9. `@Lazy` не спасает от constructor circular dependency
Circular dependency через constructor → **FAIL** всегда.
`@Lazy` помогает только при field/setter injection circular.

### 10. JdbcTemplate — это НЕ `data-jpa`
`spring-boot-starter-jdbc` — отдельный стартер. `data-jpa` тянет JPA/Hibernate; `jdbc` — только JdbcTemplate + DataSource.

### 11. `@PreFilter` vs `@PreAuthorize`
- `@PreAuthorize` → разрешает или запрещает вызов метода **целиком**
- `@PreFilter` → метод **всегда** вызывается, но коллекция фильтруется. Работает только если аргумент — коллекция или массив.

### 12. `@Around` — нужно вызвать `proceed()`
Если забудешь вызвать `joinPoint.proceed()` → метод **не** выполнится!
`@Before`/`@After` не требуют — они не оборачивают выполнение.

### 13. `BeanFactoryPostProcessor` vs `BeanPostProcessor`
- `BeanFactoryPostProcessor` → работает с bean **definitions** (до создания бинов), метод: `postProcessBeanFactory()`
- `BeanPostProcessor` → работает с **готовыми** бинами (до/после инициализации)

### 14. `@Profile` — только для бинов, не для полей
Нельзя поставить `@Profile` на поле класса. Только на `@Component` классе или `@Bean` методе.

### 15. `ContentNegotiatingViewResolver` сам не рендерит
Он только выбирает подходящий ViewResolver по `Accept` header. Рендерит делегат.

### 16. `@DataJpaTest` НЕ видит `@Service`
Если нужен Service — явно импортируй: `@Import(MyService.class)`

### 17. `SpringRunner` vs `SpringExtension`
- `SpringRunner` = JUnit 4 (`@RunWith`)
- `SpringExtension` = JUnit 5 (`@ExtendWith`)
- Spring Boot Test по умолчанию — JUnit 5. `SpringRunner` не подходит.

### 18. Порядок property sources
- Command-line arguments **ВЫШЕ** чем `application.properties`
- OS environment variables ниже command-line, но **ВЫШЕ** `application.properties`
- `@TestPropertySource` выше всего (в тестах) кроме DevTools

### 19. `@ConditionalOnMissingBean` — паттерн default
Spring использует `@ConditionalOnMissingBean` чтобы дать пользователю переопределить авто-конфигурацию своим бином.
> Пример: создашь свой `ObjectMapper` → Boot **НЕ** создаст свой.

### 20. `application.properties` порядок поиска
`/config/` > `/` > `classpath:/config/` > `classpath:/`
Файл в `/config/` папке имеет **высший** приоритет и перекрывает classpath.

---

## 🧠 Паттерны экзаменационных вопросов

| Если в вопросе есть... | Думай о... |
|---|---|
| "по умолчанию" | singleton scope, REQUIRED propagation, GET mapping |
| "circular dependency" | constructor = FAIL; setter/field = можно с `@Lazy` |
| "несколько DataSource" | `JtaTransactionManager` (не Jpa) |
| "без web layer" | `@DataJpaTest` или `@SpringBootTest(webEnv=NONE)` |
| "mock сервер" | `@SpringBootTest(webEnvironment=MOCK)` |
| "реальный сервер" | `RANDOM_PORT` или `DEFINED_PORT` |
| "только JPA компоненты" | `@DataJpaTest` |
| "только контроллер" | `@WebMvcTest` |
| "весь контекст" | `@SpringBootTest` |
| "фильтрация коллекции" | `@PreFilter` (не `@PreAuthorize`) |
| "cross-cutting concern" | AOP |
| "fine-grained транзакция" | `TransactionTemplate` |
| "default реализация бина" | `@ConditionalOnMissingBean` |
| "проверить наличие в classpath" | `@ConditionalOnClass` |
| "SpEL в security" | `@PreAuthorize` (не `@Secured`) |
| "embedded server" | `spring-boot-starter-web` (Tomcat внутри) |
| "metrics" | Micrometer + Actuator |
| "health check" | `/actuator/health` |


Relaxed binding ensures that properties defined in application.properties or environment variables can be bound flexibly to Java fields, 
supporting formats like 'my.property-name', 
'MY_PROPERTY_NAME', and 'my_property_name' for a field named 'propertyName'

@MockBean automatically scans the test's ApplicationContext and replaces any bean of the annotated field's type with a Mockito mock. 
This is crucial for isolating the component under test from its real dependencies.

Spring Security is implemented via a chain of Servlet Filters. 
Key filters include SecurityContextPersistenceFilter (context management), 
UsernamePasswordAuthenticationFilter (form login), and ExceptionTranslationFilter (error handling). 
The entire chain ensures that security checks are applied consistently to every request.

The key distinction is scope. 
@WebMvcTest is a slicing annotation that aggressively limits the loaded context for isolation. 
@AutoConfigureMockMvc is used when a full context is loaded (@SpringBootTest) but you still want the convenience of MockMvc injection and configuration.

Spring MVC integrates tightly with the Bean Validation standard. By annotating an argument (like a @RequestBody DTO) with @Valid or @Validated, 
Spring automatically runs the validation process using the configured Validator.

Spring MVC is highly flexible, supporting dozens of handler method argument types, including Servlet API objects (HttpServletRequest, HttpServletResponse), 
model containers (Model, Map), and security/validation results (Principal, BindingResult).

Spring Data supports a wide range of return types, including optional wrappers, streams, and collections, allowing flexibility in how data is retrieved.

@PostFilter is used to dynamically modify the returned collection or array, removing elements based on the security context.
@PreFilter is used to filter the incoming collection/array arguments of a method based on an expression.

Dependencies must still be manually mocked using @MockBean or Mockito.

@Secured is limited to role names (e.g., 'ROLE_ADMIN', 'ROLE_USER').

@ControllerAdvice is an enhancement to the controller mechanism, allowing developers to define components that contain cross-cutting concerns that 
apply to all controllers, primarily exception handling and model data population.

@Configuration classes hold the 'blueprints' for creating beans. The framework processes the @Bean methods within to create and configure bean instances.

While declarative dependency injection is preferred, programmatic lookups are sometimes necessary. This is done by obtaining a reference to the ApplicationContext 
(either by implementing ApplicationContextAware or BeanFactoryAware) and calling the getBean() method.

In Spring applications without Spring Boot, or when explicit configuration is required, @EnableAspectJAutoProxy is necessary. 
Spring Boot auto-configures this infrastructure if the relevant AspectJ libraries are on the classpath.

Ключевая мысль — @EnableTransactionManagement включает инфраструктуру прокси, которая перехватывает вызовы методов с @Transactional. 
По умолчанию используется JDK dynamic proxy (для интерфейсов) или CGLIB (для классов). Отсюда классический подводный вопрос: 
вызов @Transactional-метода внутри того же класса (self-invocation) транзакцию не создаст, 
потому что вызов не проходит через прокси. Это очень любят спрашивать.
Чистый Spring (без Boot) — вот тут да, без явного включения транзакционного управления @Transactional обрабатываться не будет. Но @EnableTransactionManagement — не единственный способ. Альтернативы:

XML-конфигурация — <tx:annotation-driven/> в XML делает ровно то же самое, что и @EnableTransactionManagement. Это эквивалентный способ активации.
Программный (императивный) подход — можно вообще обойтись без декларативных транзакций и использовать TransactionTemplate или напрямую PlatformTransactionManager. 
В этом случае ни @EnableTransactionManagement, ни @Transactional не нужны — вы управляете транзакциями вручную в коде.
AspectJ-режим — @EnableTransactionManagement(mode = AdviceMode.ASPECTJ) переключает с прокси на compile-time/load-time weaving. 
Это всё ещё та же аннотация, но принципиально другой механизм работы, и на экзамене могут спросить разницу.

HttpMessageConverters handle the serialization and deserialization between Java objects and the payload format 
(JSON via Jackson, XML via JAXB, etc.), essential for RESTful services.
Компоненты HttpMessageConverters играют важнейшую роль в создании REST API. Именно их аннотация @RequestBody использует для чтения входящих данных, а аннотация 
@ResponseBody — для записи исходящих данных в правильном типе содержимого (например, для преобразования списка объектов в массив JSON).

HandlerInterceptors — это важный механизм для реализации таких функций, как логирование, проверки безопасности и мониторинг группы обработчиков. 
Они находятся между DispatcherServlet и фактическим выполнением обработчика.
Объекты HandlerInterceptors предоставляют методы обратного вызова (preHandle, postHandle, afterCompletion), 
которые позволяют разработчикам внедрять сквозную логику в ключевых точках жизненного цикла обработки запроса.\

Spring Data's query derivation engine is highly expressive, supporting limiters (Top/First) and ordering (OrderBy),
as well as complex criteria using logical operators (And/Or) and comparison operators (LessThan/GreaterThan).

Weaving is the process of linking aspects with other application types or objects to create an advised object. 
Spring AOP typically uses runtime proxy-based weaving.

The two highest-precedence ways to activate profiles (outside of the code itself) are command-line arguments 
(--spring.profiles.active=dev,prod) and environment variables (SPRING_PROFILES_ACTIVE=dev,prod).

Spring MVC provides various annotations to extract data from different parts of the HTTP request: 
@PathVariable (URI template), @RequestParam (query string), @RequestHeader (HTTP headers), and @RequestBody (request body).

The @PathVariable annotation binds a variable from the URI template path (like {id}) to a method parameter (userId). 
@RequestParam is used for query parameters.

META-INF/spring.factories - This file contains a list of fully qualified class names for various extension points, most notably `EnableAutoConfiguration`, 
allowing Spring Boot to find and apply all auto-configuration classes across dependencies.

Как корректно внедрить значение из источника свойств приложения (например, application.properties) в поле Spring-бина?
Наиболее распространенный способ — использование аннотации `@Value`. Программный способ — использование объекта 
`Environment`, который доступен через автоматическое внедрение зависимостей (autowiring) или интерфейс 
`EnvironmentAware`. Оба метода допустимы для внедрения значений конфигурации.

JdbcTemplate's greatest contribution is abstracting away boilerplate JDBC code 
and translating technology-specific (checked) exceptions into Spring's unified, unchecked DataAccessException hierarchy, simplifying error handling.

Spring Boot Actuator provides production-ready features for monitoring and managing applications. /info provides static, general app data, while /beans 
provides runtime details of the application's components (beans). Other standard endpoints include /health, /metrics, and /env.

The method chain starting with http.authorizeRequests() or http.authorizeHttpRequests() is dedicated to 
configuring authorization, defining which users or roles can access specific URL patterns.

The InitializingBean and DisposableBean interfaces allow a bean to hook into its own lifecycle events after initialization and 
before destruction, respectively. Alternatively, @PostConstruct and @PreDestroy annotations can be used.

@ExceptionHandler maps an exception type to a handler method. @ResponseStatus maps an exception type directly to an HTTP status code. 
@ControllerAdvice is for applying handlers globally.

FilterSecurityInterceptor is a key Filter in the Security Filter Chain that protects the web layer, using the AccessDecisionManager 
to enforce authorization rules defined via `http.authorizeRequests()`.

The Propagation.NOT_SUPPORTED setting means the method is explicitly instructed not to run inside a transaction. 
If a calling transaction is active, it will be temporarily suspended before the method is executed.

The @EnableAutoConfiguration annotation, typically included in @SpringBootApplication, triggers the Spring Boot auto-configuration process, which uses conditional logic 
(e.g., @ConditionalOnClass) to automatically configure the application context.

@DataJpaTest is a slice test designed to integrate and test the data access components. 
It limits the context load and provides an in-memory database for convenience and speed.

@ModelAttribute is the key component for binding web request parameters (form fields, query parameters) 
to a simple Java object (POJO) in a conventional Spring MVC application.

Аннотация `@AutoConfigureMockMvc` — это часто используемая аннотация в сочетании с `@WebMvcTest` (или `@SpringBootTest`) 
для автоматической настройки 
MockMvc, позволяющая имитировать HTTP-запросы к контроллеру без необходимости запуска сервера.

The @Profile annotation is used to conditionally enable or disable beans and @Configuration classes based on the active profiles.

Method security with @PreAuthorize uses Spring AOP to apply proxies, running authorization checks 
based on SpEL before the method execution. It must be explicitly enabled.

PlatformTransactionManager decouples the application from the specific transaction infrastructure 
(e.g., JpaTransactionManager, DataSourceTransactionManager) by defining the standard commit/rollback/begin methods.

To access an Actuator endpoint, the request path must combine the management context path (`/manage`) with the endpoint ID (`/health`). 
Furthermore, the endpoint must be exposed using the `management.endpoints.web.exposure.include` property.

The CGLIB proxy intercepts inter-bean method calls to @Bean methods 
and consults the container to ensure that singleton instances are returned consistently.

@Valid (or the Spring-specific @Validated) triggers the validation process, 
using constraint annotations (like @NotNull, @Size) on the Item object's fields.

Profiles can be activated via system properties (often set by command line arguments) or by environment variables, 
which are automatically translated and loaded by Spring Boot's property source mechanism.

A critical behavior of Spring AOP's proxy mechanism is that calls from one method of the target object to another method on the same object 
(self-invocation) bypass the proxy entirely. The advice is therefore not applied to these internal calls.

Mocks are better suited for complex third-party objects that you only want to stub or verify.
A Mockito Spy is a wrapper around a real instance (a partial mock). 
It calls the real methods by default but allows the tester to verify calls and override specific methods. 
This is useful when you only need to mock a small part of a complex dependency.

Prototype beans are non-shared. A new instance is created for every request. Crucially,
Spring does not manage the destruction lifecycle for prototype beans, m
eaning `@PreDestroy` methods are generally not called.

SecurityContextHolder is a storage utility that provides access to the SecurityContext, 
which is crucial because it contains the Authentication object representing the application's 'who is currently logged in' state.

Spring's default transactional behavior only triggers a transaction rollback for unchecked exceptions (RuntimeException, Error). 
Checked exceptions, by default, allow the transaction to proceed to commit, even though the method throws the exception.

In Spring MVC, when validating an object (using @Valid or @Validated), the validation results are captured in a 
BindingResult argument, which must be declared immediately after the model object argument in the method signature.

The Spring Boot build plugin's most distinguishing feature is its ability to create a self-contained, 
executable jar file by unpacking and restructuring the original jar and embedding all necessary dependencies and the bootloader classes.

The Spring TestContext Framework caches application contexts to speed up testing. 
@DirtiesContext is used to explicitly tell the framework that the current context should not be reused for subsequent tests, 
ensuring test isolation by forcing a fresh context load.

The Advice annotations, such as @Before, @AfterReturning, @AfterThrowing, @After, and @Around, 
define the actual 'what' (the code) that runs at the intercepted method execution ('where').

SessionCreationPolicy.STATELESS is essential for building scalable, high-performance RESTful APIs,
as it avoids session memory overhead (scalability) and forces the client to present credentials (usually a token) with every request (fitting the REST model).

Method security requires two things: explicit activation via @EnableMethodSecurity (or legacy equivalent), 
and the target methods must belong to a bean that is managed by the Spring container so that an AOP proxy can be applied to intercept the method calls.

The BeanPostProcessor is a factory extension point. Its methods (`postProcessBeforeInitialization` and `postProcessAfterInitialization`) 
provide the ability to modify or wrap (proxy) a bean instance before it is fully ready for use.

View Resolvers (like InternalResourceViewResolver or ThymeleafViewResolver) translate the logical string name returned by the controller into an actual, 
renderable View object, often by prepending and appending prefixes and suffixes to the name.

Spring supports several ways to define initialization callbacks: the annotation-based @PostConstruct (preferred), 
the interface-based InitializingBean (Spring-specific), 
and the declarative init-method attribute (for @Bean definitions). These methods run after construction and DI.

The TargetSource abstracts the location and management of the target object behind an AOP proxy. For singletons, the TargetSource is trivial, 
but for more complex scenarios (like prototype or pooling), it plays a crucial role in providing the correct, fresh instance.

A BeanPostProcessor allows for custom logic to be applied to a bean instance (e.g., proxy creation for AOP, injecting resource loaders) 
both before any initialization methods (like @PostConstruct) and after, providing a hook into the bean's lifecycle before it's ready for use.

Matrix variables are a less common URI style but are supported by Spring MVC. 
@MatrixVariable allows Spring to parse specific name-value pairs embedded within a URI path segment.

По умолчанию большинство конечных точек Actuator не доступны через Интернет. Это свойство необходимо установить, 
чтобы явно указать, какие конечные точки должны быть доступны по протоколу HTTP (например, по адресу /actuator/health).

@ControllerAdvice is essential for building robust web applications by centralizing exception handling, preventing 
repetitive try-catch blocks in every controller method, and ensuring consistent responses across the application.

In modern Spring Security (since 5.7), WebSecurityConfigurerAdapter is deprecated and not strictly required for 
@EnableWebSecurity. @EnableWebSecurity enables the Spring Security filter chain and is essential for manual 
configuration in standard Spring apps.

Аннотация `@RestControllerAdvice` упрощает глобальную обработку исключений для REST API. 
Она гарантирует автоматическое преобразование пользовательских ответов об ошибках (обычно это объекты 
Java, содержащие подробную информацию об ошибке) в формат JSON/XML, что избавляет 
от необходимости вручную добавлять аннотацию `@ResponseBody` к каждому методу обработки исключений.

В Spring Data JPA auditing — это автозаполнение полей:
@CreatedDate
@LastModifiedDate
@CreatedBy
@LastModifiedBy

Spring Boot JARs are self-contained, including both the application code and the infrastructure (server and dependencies).