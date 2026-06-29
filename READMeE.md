# Core Spring & Spring Boot — Lab Projects

Это учебный набор лаб по курсам **Core Spring** и **Spring Boot**. Все модули лежат в каталоге [`lab/`](./lab) и почти каждый идёт парой:

- модуль без суффикса — **задание** со скелетом кода и `TODO`-комментариями;
- модуль с суффиксом `-solution` — **эталонное решение**.

Каждый модуль крутится вокруг одной и той же доменной модели — **Reward Network**, программы лояльности «оплата ужина → начисление на счёт → распределение по бенефициарам». Это удобно: бизнес-логика общая, меняется только тот слой Spring/Spring Boot, который изучается в конкретной лабе.

Сборка:
- **Maven:** `cd lab && ./mvnw clean verify`
- **Gradle:** `cd lab && ./gradlew build`
- БД во всех модулях — embedded HSQL, ничего ставить локально не нужно (для модулей с JPA — Hibernate, тоже встроенный).

Импорт в IDE: открыть `lab/pom.xml` (Maven) или `lab/build.gradle` (Gradle) как корневой проект.

---

## Содержание

1. [Бизнес-логика модели Reward Network](#1-бизнес-логика-модели-reward-network)
2. [Карта модулей](#2-карта-модулей)
3. [Разбор каждого задания](#3-разбор-каждого-задания)
   - [10 — Spring Intro](#модуль-10--10-spring-intro)
   - [12 — JavaConfig & DI](#модуль-12--12-javaconfig-dependency-injection)
   - [16 — Annotations & ComponentScan](#модуль-16--16-annotations)
   - [22 — AOP (детально)](#модуль-22--22-aop--spring-aop-детально)
   - [24 — Test & Profiles](#модуль-24--24-test)
   - [26 — JdbcTemplate](#модуль-26--26-jdbc)
   - [28 — Transactions (детально)](#модуль-28--28-transactions-детально)
   - [30 — JDBC Boot](#модуль-30--30-jdbc-boot-solution)
   - [32 — JDBC Autoconfig](#модуль-32--32-jdbc-autoconfig)
   - [33 — Custom Auto-Configuration](#модуль-33--33-autoconfig-helloworld)
   - [34 — Spring Data JPA](#модуль-34--34-spring-data-jpa)
   - [36 — Spring MVC](#модуль-36--36-mvc)
   - [38 — REST WS](#модуль-38--38-rest-ws)
   - [40 — Boot Test](#модуль-40--40-boot-test)
   - [42 — Security REST](#модуль-42--42-security-rest)
   - [44 — Actuator](#модуль-44--44-actuator)
4. [Сквозные темы: как работают AOP, транзакции и автоконфигурация под капотом](#4-сквозные-темы)
5. [Как читать репозиторий](#5-как-читать-репозиторий)

---

## 1. Бизнес-логика модели Reward Network

### 1.1. Что моделирует приложение

Клиент сети ресторанов оплачивает ужин кредитной картой. Сеть лояльности (`RewardNetwork`):

1. Находит счёт клиента по номеру карты.
2. Находит ресторан по «merchant number».
3. Считает бенефит (часть суммы) по правилам ресторана.
4. Распределяет бенефит между бенефициарами счёта (например, между членами семьи) согласно их процентным долям.
5. Сохраняет подтверждение транзакции (`RewardConfirmation`) с уникальным номером.

### 1.2. Главные классы

Они живут в `00-rewards-common` (примитивы домена) и `01-rewards-db` (агрегаты, репозитории, сервис):

| Класс / интерфейс | Тип | Назначение |
|---|---|---|
| `rewards.RewardNetwork` | интерфейс-фасад | главная точка входа: `RewardConfirmation rewardAccountFor(Dining dining)` |
| `rewards.internal.RewardNetworkImpl` | сервис | оркестрирует всю бизнес-операцию |
| `rewards.Dining` | value object | факт оплаты в ресторане (сумма, номер карты, мерчант, дата) |
| `rewards.internal.account.Account` | JPA `@Entity` / **агрегат** | счёт клиента; содержит набор `Beneficiary`; знает, как распределять контрибьюции |
| `rewards.internal.account.Beneficiary` | JPA `@Entity` | один получатель доли: имя, `Percentage`, `MonetaryAmount savings` |
| `rewards.internal.restaurant.Restaurant` | JPA `@Entity` | ресторан с `benefitPercentage` и `BenefitAvailabilityPolicy` |
| `rewards.internal.restaurant.BenefitAvailabilityPolicy` | стратегия | политика «когда положен бенефит». Реализации `AlwaysAvailable`/`NeverAvailable` хранятся в одной колонке (`'A'`/`'N'`) и маппятся через JPA `@Access(PROPERTY)` |
| `rewards.AccountContribution` | value object | сводка о начислении на счёт + `Set<Distribution>` на каждого бенефициара |
| `rewards.AccountContribution.Distribution` | value object | сумма для одного бенефициара + его % + текущий итог `totalSavings` |
| `rewards.RewardConfirmation` | value object | возвращаемое подтверждение с уникальным номером |
| `common.money.MonetaryAmount` | value object | деньги (внутри `BigDecimal`), поддерживает `add`, `multiplyBy(Percentage)` |
| `common.money.Percentage` | value object | %, поддерживает суммирование с проверкой «не больше 100» |
| `common.datetime.SimpleDate` | value object | упрощённая дата |
| `*Repository` | DAO | три репозитория: `AccountRepository`, `RestaurantRepository`, `RewardRepository`. Есть реализации **Stub**, **Jdbc** и **Jpa** (в зависимости от модуля) |
| `accounts.AccountManager` (+ `JpaAccountManager`) | сервис | CRUD над `Account` для веб-модулей (MVC, REST, Security, Actuator) |

### 1.3. Алгоритм `RewardNetworkImpl.rewardAccountFor(Dining)`

Эталонная последовательность (см. `10-spring-intro-solution`):

```java
public RewardConfirmation rewardAccountFor(Dining dining) {
    Account account       = accountRepository.findByCreditCard(dining.getCreditCardNumber());
    Restaurant restaurant = restaurantRepository.findByMerchantNumber(dining.getMerchantNumber());

    MonetaryAmount amount             = restaurant.calculateBenefitFor(account, dining);
    AccountContribution contribution  = account.makeContribution(amount);

    accountRepository.updateBeneficiaries(account);
    return rewardRepository.confirmReward(contribution, dining);
}
```

Что происходит внутри:

1. **`Restaurant.calculateBenefitFor(account, dining)`**
   ```java
   if (benefitAvailabilityPolicy.isBenefitAvailableFor(account, dining)) {
       return dining.getAmount().multiplyBy(benefitPercentage);
   } else {
       return MonetaryAmount.zero();
   }
   ```
   Политика `BenefitAvailabilityPolicy` — это паттерн Strategy. Хранится в колонке одним символом и через JPA-аксессоры превращается в singleton `AlwaysAvailable.INSTANCE` или `NeverAvailable.INSTANCE`.

2. **`Account.makeContribution(amount)`**
   ```java
   if (!isValid()) {
       throw new IllegalStateException("invalid beneficiary allocations");
   }
   Set<Distribution> distributions = distribute(amount);
   return new AccountContribution(getNumber(), amount, distributions);
   ```
   - `isValid()` проверяет, что сумма процентов всех бенефициаров ровно 100%.
   - `distribute(amount)` для каждого бенефициара считает `amount.multiplyBy(beneficiary.getAllocationPercentage())`, делает `beneficiary.credit(distributionAmount)` и собирает `Distribution`.

3. **`accountRepository.updateBeneficiaries(account)`** — сохраняет новые `savings` бенефициаров (для `Jdbc`-варианта это `UPDATE T_ACCOUNT_BENEFICIARY ...`, для `Jpa` это просто merge через `EntityManager`).

4. **`rewardRepository.confirmReward(contribution, dining)`** — `INSERT` в `T_REWARD`, читая следующий номер из последовательности `S_REWARD_CONFIRMATION_NUMBER`. Возвращает `RewardConfirmation`.

### 1.4. Схема БД (HSQL embedded)

`01-rewards-db/src/main/resources/rewards/testdb/schema.sql`:

```
T_ACCOUNT(ID, NUMBER, NAME)
  └─ 1:N  T_ACCOUNT_CREDIT_CARD(ID, ACCOUNT_ID, NUMBER)
  └─ 1:N  T_ACCOUNT_BENEFICIARY(ID, ACCOUNT_ID, NAME, ALLOCATION_PERCENTAGE, SAVINGS)

T_RESTAURANT(ID, MERCHANT_NUMBER, NAME, BENEFIT_PERCENTAGE, BENEFIT_AVAILABILITY_POLICY)

T_REWARD(ID, CONFIRMATION_NUMBER, REWARD_AMOUNT, REWARD_DATE,
         ACCOUNT_NUMBER, DINING_MERCHANT_NUMBER, DINING_DATE, DINING_AMOUNT)

S_REWARD_CONFIRMATION_NUMBER  (sequence)
DUAL_REWARD_CONFIRMATION_NUMBER (вспом. таблица для HSQL)
```

Все остальные модули **переиспользуют** эти же сущности, меняя только обвязку.

### 1.5. Ключевые архитектурные решения

- **Rich domain.** Бизнес-логика живёт в сущностях (`Account.makeContribution`, `Restaurant.calculateBenefitFor`), а не в «толстых» сервисах. Сервис только оркестрирует.
- **Value objects.** `MonetaryAmount` и `Percentage` неизменяемы и имеют `equals/hashCode`, что устраняет ошибки округления и упрощает тестирование.
- **Stub-реализации.** Для каждого репозитория есть Stub-вариант — это позволяет писать чистые юнит-тесты и постепенно вводить инфраструктуру.
- **Package-private методы для ORM.** Например, `Account.restoreBeneficiary(...)` и `Restaurant.getDbBenefitAvailabilityPolicy()` — это «лазейки» для репозитория, скрытые от прикладного кода.

---

## 2. Карта модулей

```
00-rewards-common        общие value-объекты (MonetaryAmount, Percentage, SimpleDate)
01-rewards-db            домен + JPA + stub-репозитории + общая схема БД

10-spring-intro          первое знакомство: реализовать RewardNetworkImpl
12-javaconfig-di         @Configuration + @Bean, конструкторный DI
16-annotations           @Component / @Repository / @Autowired / @ComponentScan
22-aop                   Spring AOP: логирование, мониторинг, обработка исключений
24-test                  Spring TestContext + @ActiveProfiles
26-jdbc                  JdbcTemplate вместо «голого» JDBC
28-transactions          @Transactional + @EnableTransactionManagement
30-jdbc-boot             миграция конфигурации на Spring Boot
32-jdbc-autoconfig       @SpringBootApplication, @ConfigurationProperties
33-autoconfig-helloworld написание собственного стартера/auto-configuration
34-spring-data-jpa       репозитории Spring Data JPA
36-mvc                   Spring MVC + Mustache (HTML view)
38-rest-ws               REST API (@RestController, ResponseEntity, content negotiation)
40-boot-test             @SpringBootTest / @WebMvcTest / MockMvc
42-security-rest         Spring Security: HTTP Basic, роли, SecurityFilterChain
44-actuator              health/info/metrics + кастомный endpoint
```

Логика возрастает от **Spring Core → AOP → JDBC → Transactions → Spring Boot → Boot autoconfig → Data/Web/Test/Security/Actuator**.

---

## 3. Разбор каждого задания

### Модуль 10 — `10-spring-intro`

**Цель:** познакомиться с бизнес-логикой и сделать первую реализацию `RewardNetwork` без Spring'а вообще.

**Ключевые классы:**
- `RewardNetwork` (интерфейс), `RewardNetworkImpl` (с тремя пустыми полями репозиториев и пустым `rewardAccountFor`).
- `StubAccountRepository`, `StubRestaurantRepository`, `StubRewardRepository` — in-memory заглушки с предзаполненными данными (например, счёт «123456789» с двумя бенефициарами 50/50).
- `RewardNetworkImplTests` — JUnit-тест, в котором руками собирается граф.

**TODO студента:**
- `TODO-07/08`: реализовать `rewardAccountFor(...)` — собственно ту 6-строчную последовательность.
- `TODO-10`: убрать `@Disabled` с теста, написать `setUp()` который руками создаёт три stub-репозитория и инжектит их в `RewardNetworkImpl`.

**Что в `-solution`:** реализация из раздела 1.3. Тест проверяет, что у `StubAccountRepository` дёрнули `findByCreditCard("1234123412341234")`, обновили бенефициаров и получили подтверждение с суммой `8.00 * 50% = 4.00`.

**Зачем модуль:** показать, что без контейнера всё работает. Дальше в лабе 12 ровно эту сборку перенесут на Spring.

---

### Модуль 12 — `12-javaconfig-dependency-injection`

**Цель:** научиться объявлять бины через JavaConfig и собирать граф зависимостей в `ApplicationContext`.

**Ключевые классы:**
- `config.RewardsConfig` — `@Configuration` с четырьмя `@Bean`-методами:
  ```java
  @Configuration
  public class RewardsConfig {
      private final DataSource dataSource;
      public RewardsConfig(DataSource dataSource) { this.dataSource = dataSource; }

      @Bean public RewardNetwork rewardNetwork() {
          return new RewardNetworkImpl(accountRepository(),
                                       restaurantRepository(),
                                       rewardRepository());
      }
      @Bean public AccountRepository accountRepository()       { return new JdbcAccountRepository(dataSource); }
      @Bean public RestaurantRepository restaurantRepository() { return new JdbcRestaurantRepository(dataSource); }
      @Bean public RewardRepository rewardRepository()         { return new JdbcRewardRepository(dataSource); }
  }
  ```
- `config.TestInfrastructureConfig` — отдельный `@Configuration`, который `@Import(RewardsConfig.class)` и поднимает HSQL через `EmbeddedDatabaseBuilder`:
  ```java
  @Bean public DataSource dataSource() {
      return new EmbeddedDatabaseBuilder()
              .addScript("classpath:rewards/testdb/schema.sql")
              .addScript("classpath:rewards/testdb/data.sql")
              .build();
  }
  ```
- `RewardNetworkTests` — поднимает контекст в `@BeforeEach` (`SpringApplication.run(TestInfrastructureConfig.class)`), достаёт `RewardNetwork` через `getBean`.

**TODO студента:** написать всё содержимое `RewardsConfig` и в тесте — `setUp()`/`tearDown()` + `@BeforeEach`/`@AfterEach`.

**Важные нюансы:**
- Вызов `accountRepository()` внутри `rewardNetwork()` **не** создаёт новый экземпляр: `@Configuration` оборачивается CGLIB-прокси, который кеширует результат `@Bean`-метода (singleton scope).
- Дочерний контейнер (`TestInfrastructureConfig`) держит инфраструктуру (DataSource), а `RewardsConfig` — прикладной слой. Это правильное разделение: в проде `DataSource` придёт из JNDI, в тесте — из embedded.

---

### Модуль 16 — `16-annotations`

**Цель:** заменить ручное объявление бинов на сканирование компонентов и инжекцию по аннотациям.

**Ключевые изменения:**
- На `JdbcAccountRepository`, `JdbcRestaurantRepository`, `JdbcRewardRepository` навешивается `@Repository("accountRepository")` (имя бина задаётся явно ради тестов).
- `DataSource` инжектится:
  - в `JdbcAccountRepository` — через `@Autowired` на setter;
  - в `JdbcRestaurantRepository` — через сочетание конструктора и `@Autowired` setter (показывает варианты);
  - и так далее.
- `JdbcRestaurantRepository` дополнительно использует lifecycle callbacks:
  ```java
  @PostConstruct
  public void populateRestaurantCache() { ... }   // прогрев кэша при старте

  @PreDestroy
  public void clearRestaurantCache()    { ... }   // очистка при остановке
  ```
- В `RewardsConfig` всё ручное конструирование заменяется одной строкой:
  ```java
  @Configuration
  @ComponentScan("rewards.internal")
  public class RewardsConfig { }
  ```

**TODO студента:** расставить `@Repository`, `@Autowired`, `@PostConstruct`, `@PreDestroy`, `@ComponentScan`.

**Тонкие моменты:**
- `@ComponentScan` без `basePackages` сканирует пакет самого `@Configuration`-класса.
- Если бин один — `@Autowired` на конструкторе можно опустить (Spring 4.3+). В лабе используется явная аннотация ради наглядности.
- `@Repository` ещё и **переводит** `SQLException`/JDBC-исключения в иерархию `DataAccessException` благодаря `PersistenceExceptionTranslationPostProcessor`.

---

### Модуль 22 — `22-aop` — **Spring AOP (детально)**

**Цель:** вынести сквозные задачи (логирование, мониторинг времени, обработку ошибок) из бизнес-кода в отдельные **аспекты**, не трогая репозитории.

#### 3.1. Что такое Spring AOP

Spring AOP — это **proxy-based** AOP: фреймворк создаёт прокси-объект вокруг бина и подменяет им оригинал в контексте. Все вызовы извне идут через прокси, который перехватывает их и пропускает через цепочку **advice** (логику аспекта) перед/после/вместо настоящего метода.

Под капотом:
- если у целевого класса есть интерфейс — используется **JDK Dynamic Proxy** (`java.lang.reflect.Proxy`);
- если нет — **CGLIB**: создаётся подкласс целевого класса (метод-перехватчик в подклассе вызывает super).
- `@EnableAspectJAutoProxy(proxyTargetClass = true)` принудительно включает CGLIB.

Spring AOP перехватывает **только публичные вызовы между бинами через прокси**. Это значит:
- self-invocation (`this.find(...)` из самого репозитория) НЕ попадёт под аспект — прокси-обёртки нет;
- private/final-методы не перехватываются (CGLIB не может переопределить final);
- `static`-методы не перехватываются никогда.

Если нужна более тонкая работа — есть полноценный AspectJ с weave-time или load-time weaving, но в этой лабе используется именно Spring AOP.

#### 3.2. Реализованные аспекты

**`LoggingAspect`** — логирование поиска + измерение времени обновлений через JAMon.

```java
@Aspect
@Component
public class LoggingAspect {

    private static final Logger LOG = LoggerFactory.getLogger(LoggingAspect.class);

    // --- 1. Логирование ДО вызова любого find*() ---
    @Before("execution(public * rewards.internal.*.*Repository.find*(..))")
    public void implLogging(JoinPoint jp) {
        LOG.info("'{}' invoked with {}",
                 jp.getSignature().getName(),
                 Arrays.toString(jp.getArgs()));
    }

    // --- 2. Замер времени вокруг любого update*() ---
    @Around("execution(public * rewards.internal.*.*Repository.update*(..))")
    public Object monitor(ProceedingJoinPoint pjp) throws Throwable {
        Monitor monitor = MonitorFactory.start(pjp.toShortString());
        try {
            return pjp.proceed();       // выполнить оригинальный метод
        } finally {
            monitor.stop();             // даже если кинется исключение
            LOG.info("JAMon stats: {}", monitor);
        }
    }
}
```

**`DBExceptionHandlingAspect`** — централизованная обработка ошибок доступа к БД.

```java
@Aspect
@Component
public class DBExceptionHandlingAspect {
    private static final Logger LOG = LoggerFactory.getLogger(DBExceptionHandlingAspect.class);

    @AfterThrowing(
        value    = "execution(public * rewards.internal.*.*Repository.*(..))",
        throwing = "e")
    public void implExceptionHandling(RewardDataAccessException e) {
        LOG.error("Repository failure", e);
    }
}
```

#### 3.3. Разбор pointcut-выражения

`execution(public * rewards.internal.*.*Repository.find*(..))`

| Часть | Смысл |
|---|---|
| `execution(...)` | сработать на **выполнении** метода (а не `call`, который Spring AOP не поддерживает) |
| `public` | только публичные методы |
| `*` (после `public`) | любой возвращаемый тип |
| `rewards.internal.*.*Repository` | классы с суффиксом `Repository` внутри подпакетов `rewards.internal.*` (одного уровня) |
| `.find*` | имя метода начинается с `find` |
| `(..)` | любое количество аргументов любых типов |

Полезные приёмы pointcut'ов:
- `..` в пути пакета: `rewards.internal..*Repository` — любая глубина вложенности.
- `args(arg1, arg2)` — связать аргументы метода с параметрами advice'а.
- `@annotation(org.springframework.transaction.annotation.Transactional)` — все методы с этой аннотацией.
- `within(rewards.internal..*)` — все вызовы внутри пакета.
- Можно именовать pointcut'ы и переиспользовать:
  ```java
  @Pointcut("execution(public * rewards.internal..*Repository.*(..))")
  private void anyRepositoryMethod() {}

  @Before("anyRepositoryMethod()")
  public void log(JoinPoint jp) { ... }
  ```

#### 3.4. Типы advice

| Аннотация | Когда срабатывает | Что доступно | Может изменить ход выполнения? |
|---|---|---|---|
| `@Before` | до целевого метода | `JoinPoint` (имя, аргументы) | нет, но может бросить исключение |
| `@After` | всегда после (как `finally`) | `JoinPoint` | нет |
| `@AfterReturning(returning="r")` | после успешного возврата | результат | нет, но может «подсмотреть» |
| `@AfterThrowing(throwing="e")` | после исключения | исключение | нет |
| `@Around` | оборачивает метод | `ProceedingJoinPoint` | **да** — можно не вызывать `proceed()`, изменить аргументы или результат |

`@Around` — самый мощный, но и самый «опасный»: легко забыть `proceed()` и навсегда заблокировать вызов.

#### 3.5. Активация AOP

```java
@Configuration
@ComponentScan("rewards.internal.aspects")
@EnableAspectJAutoProxy
public class AspectsConfig { }
```

`@EnableAspectJAutoProxy` подключает `AnnotationAwareAspectJAutoProxyCreator` — это `BeanPostProcessor`, который при инициализации каждого бина смотрит, попадает ли он под какой-то pointcut, и если да — оборачивает прокси.

В `pom.xml` модуля:
- `spring-aspects` — поддержка AspectJ-аннотаций;
- `jamon` — библиотека для измерения времени (используется в `@Around` советe).

#### 3.6. Что делает студент

- **Шаг 02–03 (LoggingAspect):** `@Aspect`, `@Component`, `@Before` с правильным pointcut и параметром `JoinPoint`.
- **Шаг 07–08 (Around-advice):** написать pointcut `update*(..)`, обернуть `pjp.proceed()` в `try/finally`, не забыть `throws Throwable`.
- **Шаг 10–11 (DBExceptionHandlingAspect):** `@AfterThrowing(value=..., throwing="e")`, тип параметра — конкретное исключение `RewardDataAccessException`, чтобы advice срабатывал только на нужном типе.
- **Шаг 04 (AspectsConfig):** `@ComponentScan("rewards.internal.aspects")` + `@EnableAspectJAutoProxy`.

**Финальный эффект:** в логах появляются строки «`findByCreditCard invoked with [1234...]`», измеряется время `update*()` через JAMon, а ошибки БД централизованно логируются. Сами репозитории остаются нетронутыми — это и есть «cross-cutting concern», вынесенный аспектом.

---

### Модуль 24 — `24-test`

**Цель:** заменить ручное управление контекстом в тестах на Spring TestContext Framework, познакомиться с профилями.

**Ключевые изменения:**

1. Тест:
   ```java
   @SpringJUnitConfig(TestInfrastructureConfig.class)
   @ActiveProfiles({"local","jdbc"})
   class RewardNetworkTests {
       @Autowired RewardNetwork rewardNetwork;
       @Test void testRewardForDining() { ... }
   }
   ```
   Никаких `@BeforeEach setUp()` — контекст кешируется TestContext'ом между тестами одного класса.

2. Репозитории помечаются профилями:
   ```java
   @Repository @Profile("jdbc")
   class JdbcAccountRepository implements AccountRepository { ... }

   @Repository @Profile("stub")
   class StubAccountRepository implements AccountRepository { ... }
   ```

3. Конфигурации инфраструктуры тоже по профилям:
   ```java
   @Configuration @Profile("local")
   class TestInfrastructureLocalConfig { @Bean DataSource dataSource() { ... } }

   @Configuration @Profile("jndi")
   class TestInfrastructureJndiConfig  { @Bean DataSource dataSource() { ... } }
   ```

**Что в `-solution`:** три параллельных тестовых класса с разными `@ActiveProfiles`:
- `DevRewardNetworkTests` — `{"local","jdbc"}` (embedded HSQL);
- `ProductionRewardNetworkTests` — `{"jndi","jdbc"}` (JNDI-DataSource);
- `StubRewardNetworkTests` — `{"stub"}` (без БД).

**Полезные знания:**
- `@DirtiesContext` — пометить, что тест «грязнит» контекст и его нужно пересоздать.
- `@Sql("classpath:my-sql.sql")` — выполнить SQL до/после метода.
- TestContext кеширует контексты по ключу (классы конфига + профили + ресурсы), поэтому один и тот же контекст переиспользуется во всех тестах с одинаковым сочетанием. Это даёт огромный прирост скорости тестов.

---

### Модуль 26 — `26-jdbc`

**Цель:** переписать «голый JDBC» на `JdbcTemplate`.

**Было (фрагмент `JdbcRewardRepository`):**
```java
Connection con = null;
PreparedStatement ps = null;
try {
    con = dataSource.getConnection();
    ps = con.prepareStatement("insert into T_REWARD ...");
    ps.setString(1, confirmationNumber);
    // ... ещё 6 setXxx
    ps.executeUpdate();
} catch (SQLException e) {
    throw new RuntimeException(e);
} finally {
    if (ps != null) try { ps.close(); } catch (SQLException ignored) {}
    if (con != null) try { con.close(); } catch (SQLException ignored) {}
}
```

**Стало:**
```java
jdbcTemplate.update(
    "insert into T_REWARD (CONFIRMATION_NUMBER, REWARD_AMOUNT, REWARD_DATE, " +
    "ACCOUNT_NUMBER, DINING_MERCHANT_NUMBER, DINING_DATE, DINING_AMOUNT) " +
    "values (?, ?, ?, ?, ?, ?, ?)",
    confirmationNumber, contribution.getAmount().asBigDecimal(),
    SimpleDate.today().asDate(), contribution.getAccountNumber(),
    dining.getMerchantNumber(), dining.getDate().asDate(),
    dining.getAmount().asBigDecimal());
```

Что бесплатно даёт `JdbcTemplate`:
- открытие/закрытие `Connection` и `PreparedStatement`;
- перевод `SQLException` → `DataAccessException` (иерархия unchecked, см. `BadSqlGrammarException`, `DataIntegrityViolationException`, ...);
- удобные методы: `query(sql, RowMapper)`, `queryForObject(sql, RowMapper, args)`, `queryForList`, `update`.

**RowMapper для выборок** (используется в репозиториях счёта/ресторана):
```java
class AccountExtractor implements ResultSetExtractor<Account> { ... }   // для join'ов 1:N
class RowMapper<T> { T mapRow(ResultSet rs, int rowNum); }              // для одной строки
```

---

### Модуль 28 — `28-transactions` — **Декларативные транзакции (детально)**

**Цель:** управлять транзакциями декларативно через `@Transactional` поверх AOP-прокси.

**Минимальная конфигурация:**
```java
@Configuration
@EnableTransactionManagement
public class RewardsConfig {

    @Bean public PlatformTransactionManager transactionManager(DataSource ds) {
        return new DataSourceTransactionManager(ds);
    }
    // ... @Bean DataSource, репозитории, RewardNetworkImpl
}
```

**Использование:**
```java
@Service
public class RewardNetworkImpl implements RewardNetwork {

    @Transactional
    public RewardConfirmation rewardAccountFor(Dining dining) { ... }
}
```

#### 3.7. Что происходит под капотом

`@EnableTransactionManagement` регистрирует `TransactionInterceptor` (AOP-advice) и `InfrastructureAdvisorAutoProxyCreator`. Любой бин, у которого хотя бы один метод помечен `@Transactional` (или сам класс), оборачивается прокси.

Когда прикладной код вызывает `rewardNetwork.rewardAccountFor(...)`:

1. Прокси проверяет, есть ли уже активная транзакция (см. `TransactionSynchronizationManager`).
2. По правилам **propagation** решает: создать новую, переиспользовать существующую, отложить, или бросить ошибку.
3. У `PlatformTransactionManager` запрашивается `TransactionStatus` (внутри начинается JDBC-транзакция: `connection.setAutoCommit(false)`).
4. Connection кладётся в `ThreadLocal` — `DataSourceUtils.getConnection(ds)` (которым пользуется `JdbcTemplate`) увидит его и не откроет новый.
5. Вызывается целевой метод.
6. Если он вернулся нормально — `commit`. Если бросил `RuntimeException`/`Error` — `rollback`. Checked-исключения по умолчанию **не** откатывают транзакцию (это можно изменить через `rollbackFor`).

#### 3.8. Атрибуты `@Transactional`

| Атрибут | Что значит | Значение по умолчанию |
|---|---|---|
| `propagation` | как ведёт себя относительно уже существующей транзакции | `REQUIRED` |
| `isolation` | уровень изоляции БД | `DEFAULT` (из БД) |
| `timeout` | секунд до автоматического rollback | без таймаута |
| `readOnly` | hint драйверу, что писать не будем | `false` |
| `rollbackFor` / `noRollbackFor` | какие исключения вызывают rollback | только `RuntimeException`/`Error` |

**Уровни propagation:**
- `REQUIRED` (по умолчанию) — присоединиться или создать новую;
- `REQUIRES_NEW` — приостановить внешнюю, начать собственную (commit/rollback независимо);
- `SUPPORTS` — если есть транзакция, использовать; нет — выполнить без неё;
- `NOT_SUPPORTED` — приостановить активную, выполнить без транзакции;
- `MANDATORY` — должна быть активная (иначе исключение);
- `NEVER` — не должно быть активной (иначе исключение);
- `NESTED` — savepoint внутри существующей.

#### 3.9. Подводные камни

- **Self-invocation.** Если внутри одного бина метод `A()` без `@Transactional` вызывает метод `B()` с `@Transactional`, прокси не сработает — вызов `this.B()` идёт мимо обёртки. Лечится либо вынесением в другой бин, либо `AopContext.currentProxy()` (некрасиво), либо AspectJ weave.
- **`@Transactional` на `private`/`protected`** — у Spring AOP не работает (метод не перехватывается).
- **Checked-исключения** не откатывают по умолчанию — надо `rollbackFor = Exception.class`, если хочется обратное.

В лабе студент:
- ставит `@EnableTransactionManagement` на конфиг,
- объявляет `PlatformTransactionManager`,
- расставляет `@Transactional` на `rewardAccountFor`,
- в тестах исследует поведение propagation и rollback'ов.

---

### Модуль 30 — `30-jdbc-boot-solution`

**Цель:** показать миграцию того же `JdbcTemplate`-кода на Spring Boot. Только `-solution` — это «как должно выглядеть».

**Что поменялось:**

- `pom.xml`:
  ```xml
  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.7.5</version>
  </parent>
  <dependencies>
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-jdbc</artifactId>
      </dependency>
      <dependency>
          <groupId>org.hsqldb</groupId>
          <artifactId>hsqldb</artifactId>
          <scope>runtime</scope>
      </dependency>
  </dependencies>
  ```

- Точка входа:
  ```java
  @SpringBootApplication
  public class JdbcBootApplication {
      public static void main(String[] args) { SpringApplication.run(JdbcBootApplication.class, args); }

      @Bean
      CommandLineRunner runner(JdbcTemplate jt) {
          return args -> System.out.println("Accounts: " + jt.queryForObject("select count(*) from T_ACCOUNT", Long.class));
      }
  }
  ```

- **Никакой явной конфигурации не нужно.** Boot сам:
  - находит `hsqldb` на classpath и создаёт embedded `DataSource` (`DataSourceAutoConfiguration`);
  - создаёт `JdbcTemplate` (`JdbcTemplateAutoConfiguration`);
  - создаёт `PlatformTransactionManager` (`DataSourceTransactionManagerAutoConfiguration`);
  - выполняет `schema.sql` и `data.sql` из classpath (`DataSourceInitializerAutoConfiguration`).

Это демонстрирует основную идею Boot: **convention over configuration**.

---

### Модуль 32 — `32-jdbc-autoconfig`

**Цель:** углубиться в авто-конфигурацию и научиться использовать `@ConfigurationProperties`.

**Главные изменения относительно 30:**

- `RewardsApplication`:
  ```java
  @SpringBootApplication
  @EnableConfigurationProperties(RewardsRecipientProperties.class)
  @Import(RewardsConfig.class)
  public class RewardsApplication {

      @Bean
      CommandLineRunner one(JdbcTemplate jt) {
          return args -> log.info("Accounts: {}",
              jt.queryForObject("SELECT count(*) FROM T_ACCOUNT", Long.class));
      }

      @Bean
      CommandLineRunner two(RewardsRecipientProperties props) {
          return args -> log.info("Recipient: {}, age {}", props.getName(), props.getAge());
      }
  }
  ```

- `RewardsRecipientProperties`:
  ```java
  @ConfigurationProperties(prefix = "rewards.recipient")
  public class RewardsRecipientProperties {
      private String name;
      private int age;
      private String gender;
      private String hobby;
      // getters/setters
  }
  ```

- `application.properties`:
  ```properties
  rewards.recipient.name=John Doe
  rewards.recipient.age=10
  rewards.recipient.gender=Male
  rewards.recipient.hobby=Tennis

  logging.level.config=DEBUG
  # пример: отключить авто-конфиг DataSource и предоставить свой
  spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
  ```

**Полезные приёмы:**
- `debug=true` в `application.properties` — Boot напечатает **Conditions Evaluation Report**: какие auto-configuration сработали, какие нет и почему (отсутствие класса, `@ConditionalOnMissingBean` и т.д.).
- Можно отключить отдельные авто-конфиги через `spring.autoconfigure.exclude` или `@SpringBootApplication(exclude = {...})`.
- Property binding поддерживает relaxed-форматы: `rewards.recipient.name` = `REWARDS_RECIPIENT_NAME` = `rewards.recipient.Name`.

---

### Модуль 33 — `33-autoconfig-helloworld`

**Цель:** написать собственный «стартер» — переиспользуемую auto-configuration.

**Структура:**

```
hello-lib       — интерфейс + дефолтная реализация
hello-starter   — auto-configuration + регистрация
hello-app       — приложение-потребитель
```

`hello-lib`:
```java
public interface HelloService { String sayHello(String name); }

public class TypicalHelloService implements HelloService {
    public String sayHello(String name) { return "Hello, " + name; }
}
```

`hello-starter`:
```java
@Configuration
@ConditionalOnClass(HelloService.class)
public class HelloAutoConfig {

    @Bean
    @ConditionalOnMissingBean(HelloService.class)
    public HelloService helloService() { return new TypicalHelloService(); }
}
```

Регистрация (для Spring Boot 2.x):
```
src/main/resources/META-INF/spring.factories
---
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.starter.HelloAutoConfig
```

Для Spring Boot 3.x:
```
src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
---
com.starter.HelloAutoConfig
```

`hello-app`:
```java
@SpringBootApplication
public class HelloApplication {
    public static void main(String[] a) { SpringApplication.run(HelloApplication.class, a); }

    @Bean CommandLineRunner runner(HelloService svc) { return args -> System.out.println(svc.sayHello("world")); }
}
```

**Самые важные условные аннотации:**

| Аннотация | Когда применяется |
|---|---|
| `@ConditionalOnClass` | класс есть в classpath |
| `@ConditionalOnMissingClass` | класса в classpath нет |
| `@ConditionalOnBean` | в контексте уже есть бин такого типа |
| `@ConditionalOnMissingBean` | бина такого типа в контексте ещё нет — главный приём для «разумных значений по умолчанию» |
| `@ConditionalOnProperty` | свойство имеет конкретное значение |
| `@ConditionalOnWebApplication` | приложение web (Servlet/Reactive) |

**Идея:** стартер задаёт умолчания, но пользователь всегда может перебить — объявить свой `@Bean HelloService` и `@ConditionalOnMissingBean` уйдёт с дороги.

---

### Модуль 34 — `34-spring-data-jpa`

**Цель:** заменить ручные JPA-репозитории на сгенерированные Spring Data.

**Что меняется:**

```java
// БЫЛО (JpaAccountRepository.java)
public class JpaAccountRepository implements AccountRepository {
    @PersistenceContext EntityManager em;
    public Account findByCreditCard(String cc) {
        return em.createQuery("select a from Account a join a.creditCards c where c.number = :cc", Account.class)
                 .setParameter("cc", cc).getSingleResult();
    }
    // ... все остальные методы вручную
}

// СТАЛО — только интерфейс, реализации нет вообще
public interface AccountRepository extends Repository<Account, Long> {
    Account findByCreditCardNumber(String number);   // ← сгенерируется JPQL по имени метода
    List<Account> findAll();
    Account findByNumber(String number);
    Account save(Account account);
}
```

В конфиге включается:
```java
@SpringBootApplication
@EnableJpaRepositories(basePackages = "rewards.internal")
public class JpaApplication { ... }
```

**Как Spring Data строит запросы по имени:**
- `findByCreditCardNumber(String)` → `WHERE creditCardNumber = :p`;
- `findByNumberAndName(String, String)` → `WHERE number = :p1 AND name = :p2`;
- `findByNameContainingIgnoreCase(String)` → `WHERE LOWER(name) LIKE LOWER(CONCAT('%', :p, '%'))`;
- `findTop3ByOrderByNameAsc()` — `LIMIT 3 ORDER BY name`.

Для нестандартных запросов — `@Query("...")`.

**Преимущества:** ноль бойлерплейта, единый API, поддержка проекций, `Page<T>` для пагинации, `@Modifying` для UPDATE/DELETE.

---

### Модуль 36 — `36-mvc`

**Цель:** познакомиться со Spring MVC и шаблонизатором (в лабе используется Mustache).

**Ключевые классы:**

```java
@Controller
public class AccountController {

    private final AccountManager accountManager;
    public AccountController(AccountManager m) { this.accountManager = m; }

    @GetMapping("/accounts")
    public String list(Model model) {
        model.addAttribute("accounts", accountManager.getAllAccounts());
        return "accounts/list";     // имя view — рендерится через ViewResolver
    }

    @GetMapping("/accounts/{id}")
    public String view(@PathVariable Long id, Model model) {
        model.addAttribute("account", accountManager.getAccount(id));
        return "accounts/details";
    }
}
```

**Зависимости:** `spring-boot-starter-web` + `spring-boot-starter-mustache`.

**Шаблоны:** `src/main/resources/templates/index.html` + Bootstrap-стили в `static/`.

**Чем MVC отличается от REST (модуль 38):**
- MVC-контроллер возвращает **имя view** (или `ModelAndView`), а движок рендерит HTML.
- REST-контроллер возвращает **данные** (объект) — `HttpMessageConverter` сериализует их в JSON/XML по `Accept`-заголовку.

В лабе студент: расставляет `@Controller`/`@RequestMapping`/`@GetMapping`/`@PathVariable`, добавляет `Model`-параметры, пишет шаблоны.

---

### Модуль 38 — `38-rest-ws`

**Цель:** сделать полноценный REST API с CRUD-операциями.

**Ключевые приёмы:**

```java
@RestController
@RequestMapping("/accounts")
public class AccountController {

    @GetMapping
    public List<Account> all() { return accountManager.getAllAccounts(); }

    @GetMapping("/{id}")
    public Account one(@PathVariable Long id) { return accountManager.getAccount(id); }

    @PostMapping
    public ResponseEntity<Void> create(@RequestBody Account a) {
        Account saved = accountManager.save(a);
        URI location = ServletUriComponentsBuilder
                .fromCurrentRequestUri()
                .path("/{id}").buildAndExpand(saved.getEntityId()).toUri();
        return ResponseEntity.created(location).build();
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { accountManager.delete(id); }

    // Маппинг ошибок:
    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public void handleNotFound() {}

    @ExceptionHandler(DataIntegrityViolationException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public void handleConflict() {}
}
```

**Что важно:**
- `@RestController` = `@Controller` + `@ResponseBody` на всех методах.
- `ResponseEntity` даёт полный контроль над статусом, заголовками и телом.
- `ServletUriComponentsBuilder` собирает безопасный `Location` относительно текущего запроса.
- **Content negotiation**: один и тот же endpoint может отдавать JSON и XML — зависит от `Accept`-заголовка и подключённых конвертеров (Jackson, Jackson-XML).
- Иерархия для бенефициаров: `POST /accounts/{id}/beneficiaries` создаёт нового, `DELETE /accounts/{id}/beneficiaries/{name}` удаляет и ребалансирует %.

---

### Модуль 40 — `40-boot-test`

**Цель:** показать пирамиду тестирования в Spring Boot.

**Три уровня:**

1. **Юнит-тест без Spring** — `StubAccountManager`, `new AccountController(stub)`, прямой вызов методов. Самые быстрые.

2. **`@WebMvcTest` — slice-тест web-слоя.**
   ```java
   @WebMvcTest(AccountController.class)
   class AccountControllerBootTests {

       @Autowired MockMvc mockMvc;
       @MockBean   AccountManager accountManager;

       @Test
       void getAccount() throws Exception {
           given(accountManager.getAccount(1L)).willReturn(new Account("1", "Keith"));

           mockMvc.perform(get("/accounts/1").accept(MediaType.APPLICATION_JSON))
                  .andExpect(status().isOk())
                  .andExpect(jsonPath("$.name").value("Keith"));
       }
   }
   ```
   Поднимается только web-слой, остальные бины заменены `@MockBean`.

3. **`@SpringBootTest` — полный интеграционный тест.**
   ```java
   @SpringBootTest(webEnvironment = RANDOM_PORT)
   class FullIntegrationTests {
       @Autowired TestRestTemplate http;
       @Test void list() { ... http.getForObject("/accounts", Account[].class) ... }
   }
   ```

**Полезные аннотации:** `@DataJpaTest` (поднимает только JPA-слой, embedded БД, откатывает после теста), `@JsonTest`, `@RestClientTest`.

**Важный нюанс:** в `@WebMvcTest` нужен именно `@MockBean` — голый `@Mock` (Mockito) не попадёт в Spring-контекст.

---

### Модуль 42 — `42-security-rest`

**Цель:** защитить REST API через Spring Security.

**`RestSecurityConfig`:**
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity     // если хочешь @PreAuthorize/@PostAuthorize
public class RestSecurityConfig {

    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(reg -> reg
                .requestMatchers(HttpMethod.GET,    "/accounts/**").hasAnyRole("USER","ADMIN","SUPERADMIN")
                .requestMatchers(HttpMethod.POST,   "/accounts/**").hasAnyRole("ADMIN","SUPERADMIN")
                .requestMatchers(HttpMethod.PUT,    "/accounts/**").hasAnyRole("ADMIN","SUPERADMIN")
                .requestMatchers(HttpMethod.DELETE, "/accounts/**").hasRole("SUPERADMIN")
                .anyRequest().authenticated())
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable());
        return http.build();
    }

    @Bean
    public UserDetailsService users() {
        var users = User.builder().passwordEncoder(PasswordEncoderFactories.createDelegatingPasswordEncoder()::encode);
        return new InMemoryUserDetailsManager(
            users.username("user").password("user").roles("USER").build(),
            users.username("admin").password("admin").roles("USER","ADMIN").build(),
            users.username("superadmin").password("superadmin").roles("USER","ADMIN","SUPERADMIN").build());
    }
}
```

**Что важно:**
- `SecurityFilterChain` — современный способ конфигурации (заменил `WebSecurityConfigurerAdapter` в Spring Security 6).
- Под капотом — цепочка фильтров (`SecurityContextPersistenceFilter`, `BasicAuthenticationFilter`, `ExceptionTranslationFilter`, `FilterSecurityInterceptor` / `AuthorizationFilter`...).
- CSRF выключен — это нормально для **stateless REST** (нет сессии в куках). Для браузерных приложений с куки-сессией CSRF оставлять.
- HTTP Basic — простейшая аутентификация; для прода обычно JWT или OAuth2 Resource Server.
- `@PreAuthorize("hasRole('ADMIN')")` — alternative для метод-секьюрити, работает через AOP (как и `@Transactional`).

---

### Модуль 44 — `44-actuator`

**Цель:** добавить мониторинг и observability через Spring Boot Actuator.

**Из коробки:** `/actuator/health`, `/actuator/info`, `/actuator/metrics`, `/actuator/env`, `/actuator/mappings`, ... — управление через свойства:

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus,restaurant
management.endpoint.health.show-details=always

# Кастомные info.* свойства попадут в /actuator/info
info.app.name=Rewards
info.app.version=1.0.0

# Health-группы
management.endpoint.health.group.system.include=diskSpace,db
management.endpoint.health.group.web.include=ping
```

**Свой `HealthIndicator`:**
```java
@Component
public class RestaurantHealthCheck implements HealthIndicator {
    private final RestaurantRepository repo;

    public Health health() {
        try {
            long count = repo.count();
            return Health.up().withDetail("restaurants", count).build();
        } catch (Exception ex) {
            return Health.down(ex).build();
        }
    }
}
```

**Свой `@Endpoint`:**
```java
@Component
@Endpoint(id = "restaurant")
public class RestaurantCustomEndpoint {

    @ReadOperation
    public Map<String, Object> all() { ... }   // GET /actuator/restaurant

    @WriteOperation
    public void add(String merchant) { ... }   // POST /actuator/restaurant

    @DeleteOperation
    public void remove(String merchant) {...}  // DELETE /actuator/restaurant
}
```

**Метрики** работают через Micrometer — поддерживает Prometheus, Datadog, JMX и другие back-end'ы из коробки.

---

## 4. Сквозные темы

### 4.1. Как Spring AOP, `@Transactional` и `@Async` связаны

Это всё — proxy-based аспекты с одним механизмом:

1. `BeanPostProcessor` (`AnnotationAwareAspectJAutoProxyCreator`, `InfrastructureAdvisorAutoProxyCreator`) при создании бина проверяет: попадает ли он под какой-то advisor?
2. Если да — оборачивает прокси и кладёт прокси в контейнер вместо бина.
3. Все вызовы извне идут через прокси → через цепочку interceptors (`TransactionInterceptor`, `AspectJAroundAdvice`, ...) → в целевой объект.

Из этого вытекает общий набор «граблей»:
- **self-invocation** (`this.method()`) ничего не перехватывает;
- private/final/static-методы не перехватываются;
- порядок советов задаётся `@Order`/`Ordered` (важно, например, когда логирование должно идти снаружи транзакции).

### 4.2. Эволюция конфигурации

```
XML beans
   ↓
@Configuration + @Bean          (модуль 12)
   ↓
@ComponentScan + @Component     (модуль 16)
   ↓
@SpringBootApplication +
auto-configuration              (модули 30–33)
```

`@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`. Авто-конфиги выбираются по содержимому classpath через `@ConditionalOn*`.

### 4.3. Эволюция доступа к данным

```
SQL + Connection/PreparedStatement   (до модуля 26)
   ↓
JdbcTemplate                          (модуль 26)
   ↓
+ @Transactional                      (модуль 28)
   ↓
JPA EntityManager + Hibernate         (внутри 01-rewards-db и модулей 28+)
   ↓
Spring Data JPA Repository<T,ID>      (модуль 34)
```

### 4.4. Слои тестирования

```
plain unit (без Spring)        — секунды
@WebMvcTest / @DataJpaTest      — slice context, mock'и
@SpringBootTest                 — полный контекст, иногда с реальным сервером
```

Эта пирамида явно реализована в модулях 24 и 40.

---

## 5. Как читать репозиторий

1. Открой `01-rewards-db` — там вся бизнес-модель, особенно `Account.java`, `Restaurant.java`, `RewardNetworkImpl.java`.
2. Дальше иди **по номерам**: каждый модуль добавляет один слой Spring/Spring Boot поверх той же модели.
3. В каждом модуле сначала смотри версию **без** `-solution`:
   - в `src/main/java` будут классы с комментариями `// TODO-XX:` — это шаги задания;
   - в `src/test/java` — тест, который нужно сделать зелёным.
4. Когда задание сделано (или зашёл в тупик) — сверься с одноимённым модулем с суффиксом `-solution`.
5. Команды:
   ```
   cd lab
   ./mvnw -pl 22-aop-solution test          # запустить один модуль
   ./mvnw clean verify                      # собрать всё
   ```

Удачи!
