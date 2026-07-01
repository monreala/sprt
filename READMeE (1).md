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
6. [Теория по темам модулей (для ревью)](#6-теория-по-темам-модулей-для-ревью)
   - [6.1 Value Objects (00)](#61-value-objects--00-rewards-common)
   - [6.2 Domain & JPA (01)](#62-domain-model-и-jpa--01-rewards-db)
   - [6.3 IoC / DI (10)](#63-ioc-и-dependency-injection--10-spring-intro)
   - [6.4 JavaConfig (12)](#64-javaconfig-configuration-bean--12-javaconfig-di)
   - [6.5 Стереотипы и Component Scan (16)](#65-стереотипные-аннотации-и-component-scan--16-annotations)
   - [6.6 AOP (22)](#66-aop--22-aop)
   - [6.7 TestContext и профили (24)](#67-testcontext-и-профили--24-test)
   - [6.8 JdbcTemplate (26)](#68-jdbctemplate--26-jdbc)
   - [6.9 Transactions (28)](#69-транзакции--28-transactions)
   - [6.10 Spring Boot overview (30)](#610-spring-boot-overview--30-jdbc-boot)
   - [6.11 Auto-config и Properties (32)](#611-auto-configuration-и-configurationproperties--32-jdbc-autoconfig)
   - [6.12 Свой стартер (33)](#612-собственный-стартерauto-configuration--33-autoconfig-helloworld)
   - [6.13 Spring Data JPA (34)](#613-spring-data-jpa--34-spring-data-jpa)
   - [6.14 Spring MVC (36)](#614-spring-mvc--36-mvc)
   - [6.15 REST WS (38)](#615-rest-ws--38-rest-ws)
   - [6.16 Testing Spring Boot (40)](#616-testing-spring-boot--40-boot-test)
   - [6.17 Spring Security (42)](#617-spring-security--42-security-rest)
   - [6.18 Actuator (44)](#618-actuator--44-actuator)

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

---

## 6. Теория по темам модулей (для ревью)

Ниже — концентрированная теория по каждой теме из [«Карты модулей»](#2-карта-модулей). Формат каждого пункта одинаковый: **суть → как устроено → ключевые сущности → пример из проекта (файл-указатель) → грабли**. Раздел рассчитан на подготовку к техническому ревью, поэтому темы поданы шире, чем требуется для прохождения самой лабы.

### 6.1. Value Objects — `00-rewards-common`

**Суть.** Value Object (VO) — маленький неизменяемый объект, чья идентичность определяется набором его полей (не по ссылке и не по суррогатному id). VO живут «в отрыве от жизненного цикла»: два `MonetaryAmount(10.00)` — это один и тот же объект с точки зрения бизнеса.

**Свойства настоящего VO:**
1. **Immutable.** Все поля `final` (или поле присваивается один раз в конструкторе), нет сеттеров. Изменение = возврат нового объекта.
2. **Value-based equality.** `equals`/`hashCode` считаются от значений полей, а не от identity. Без этого объект нельзя класть в `HashSet`, использовать как ключ `Map`, сравнивать в тестах.
3. **Self-validating.** Все инварианты (например, «процент не больше 100», «сумма с двумя знаками после запятой») проверяются в конструкторе, чтобы «плохое» состояние было в принципе непредставимо.
4. **Side-effect free operations.** Метод `add(other)` не мутирует `this`, а возвращает новый VO.

**VO vs Entity.**
| Признак | VO | Entity |
|---|---|---|
| Идентичность | по значению | по id (суррогатному или естественному) |
| Изменяемость | нет | да (внутри агрегата) |
| Жизненный цикл | нет | есть (создание, обновление, удаление) |
| Пример | `MonetaryAmount`, `Money`, `Address` | `Account`, `Order`, `Customer` |

**Пример из проекта.** `MonetaryAmount` (`00-rewards-common/.../common/money/MonetaryAmount.java`) хранит `BigDecimal` с фиксированным `scale = 2` и `RoundingMode.HALF_EVEN`, имеет корректный `equals/hashCode`. `Percentage` в конструкторе проверяет `0 ≤ value ≤ 1` и кидает `IllegalArgumentException`, если пришло что-то другое — это классический self-validation.

**Почему BigDecimal, а не double.** `double` теряет точность на копейках (`0.1 + 0.2 ≠ 0.3`), поэтому в финансовом коде всегда `BigDecimal` + явный `RoundingMode`. Это архитектурное решение уровня «нельзя иначе», а не вкусовщина.

**Грабли.**
- Забыть `equals/hashCode` — тесты неожиданно падают на сравнении.
- Использовать `BigDecimal.equals` там, где нужен `compareTo`: `new BigDecimal("2.0").equals(new BigDecimal("2.00"))` → `false`, но `compareTo` == 0. VO должен сравнивать нормализованные значения (`setScale`).
- Изменять внутреннее состояние через геттеры коллекций (возвращать `new HashSet<>(internal)` или `Collections.unmodifiableSet(...)`).

---

### 6.2. Domain Model и JPA — `01-rewards-db`

**Суть.** Domain Model — это Java-объекты, отражающие предметную область (счета, рестораны, начисления). JPA (Java Persistence API) — стандарт ORM, который сопоставляет эти объекты со строками в таблицах и обратно. Hibernate — самая распространённая реализация JPA.

#### 6.2.1. Anemic vs Rich domain

- **Anemic model.** Сущности — это структуры с геттерами/сеттерами, вся логика — в «сервисах». Обычно антипаттерн (Fowler, 2003).
- **Rich model.** Сущности сами знают правила: `Account.makeContribution(amount)` внутри проверяет инвариант (проценты бенефициаров == 100) и распределяет сумму. Сервис только оркестрирует, не считает.

В этом проекте — rich model. `RewardNetworkImpl` не знает, как считать долю каждого бенефициара — он делегирует это самому `Account`. Это делает бизнес-логику тестируемой без Spring и без БД.

#### 6.2.2. Ключевые аннотации JPA

| Аннотация | Что делает |
|---|---|
| `@Entity` | этот класс — сущность (persistable) |
| `@Table(name="T_ACCOUNT")` | явно указать имя таблицы (иначе — имя класса) |
| `@Id` | поле первичного ключа |
| `@GeneratedValue(strategy=IDENTITY\|SEQUENCE\|AUTO\|TABLE)` | как генерируется id |
| `@Column(name="...")` | явное имя колонки, `nullable`, `unique`, `length` |
| `@OneToMany`, `@ManyToOne`, `@ManyToMany`, `@OneToOne` | ассоциации |
| `@JoinColumn(name="ACCOUNT_ID")` | FK-колонка в дочерней таблице |
| `@Embeddable` / `@Embedded` | «встраивание» VO в родительскую таблицу (без своей таблицы) |
| `@AttributeOverride` | переопределить имя колонки, куда мапится поле `@Embeddable`-объекта |
| `@Access(FIELD\|PROPERTY)` | откуда JPA читает состояние — из поля или из геттера/сеттера |
| `@Transient` | не сохранять поле |
| `@Enumerated(STRING\|ORDINAL)` | как хранить enum (текстом или числом — практически всегда STRING) |
| `@Version` | поле для оптимистической блокировки |

**FetchType.**
- `EAGER` — грузится сразу вместе с родителем (по умолчанию для `*ToOne`).
- `LAZY` — грузится при первом обращении (по умолчанию для `*ToMany`). Требует открытой `PersistenceContext` — отсюда `LazyInitializationException`, когда работаешь с сущностью после закрытия сессии/транзакции.

**Cascade.** `PERSIST`, `MERGE`, `REMOVE`, `DETACH`, `REFRESH`, `ALL`. `orphanRemoval=true` — удалять дочерний объект, если его убрали из коллекции родителя.

#### 6.2.3. `@Embeddable` для value-объектов

`MonetaryAmount` и `Percentage` помечены `@Embeddable` — их поля лягут в колонки той же таблицы, что и владелец. Так VO переиспользуется в разных сущностях, не порождая лишних таблиц:

```java
// Beneficiary.java — 01-rewards-db/.../account/Beneficiary.java
@Embedded
@AttributeOverride(name = "value", column = @Column(name = "ALLOCATION_PERCENTAGE"))
private Percentage allocationPercentage;
```

`@AttributeOverride` нужен потому, что один и тот же VO встраивается в разные колонки (`ALLOCATION_PERCENTAGE`, `SAVINGS`), а поле внутри `Percentage`/`MonetaryAmount` всегда называется `value`.

#### 6.2.4. `@Access(PROPERTY)` — хитрый маппинг стратегии

Обычно JPA читает состояние через рефлексию поля (`FIELD`). Но иногда нужно сохранять не то, что лежит в поле, а результат вычисления. Классический трюк — маппинг стратегии `BenefitAvailabilityPolicy`:

```java
// Restaurant.java
@Access(AccessType.PROPERTY)
@Column(name = "BENEFIT_AVAILABILITY_POLICY")
protected String getDbBenefitAvailabilityPolicy() {
    if (benefitAvailabilityPolicy == AlwaysAvailable.INSTANCE) return "A";
    if (benefitAvailabilityPolicy == NeverAvailable.INSTANCE)  return "N";
    ...
}
protected void setDbBenefitAvailabilityPolicy(String code) { ... }
```

В поле — singleton-объект стратегии (`AlwaysAvailable.INSTANCE`), в колонке — один символ `'A'`/`'N'`. Пара геттер+сеттер работает как «переводчик».

#### 6.2.5. DDD-агрегат

`Account` — **агрегат-корень**: содержит коллекцию `Beneficiary`, снаружи никто не создаёт `Beneficiary` напрямую. Все инварианты («сумма процентов == 100», «есть ли валидное распределение») проверяются в самом `Account.makeContribution(amount)`. Это защищает целостность через языковые средства.

**Пакетно-приватные методы для ORM.** Например, `Account.restoreBeneficiary(...)` — лазейка для JDBC-репозиториев, скрытая от прикладного кода за счёт видимости пакета.

#### 6.2.6. Stub-репозитории

Идея: для каждого репозитория есть in-memory реализация с предзаполненными данными. Это позволяет писать чистые юнит-тесты без БД. `StubAccountRepository` создаёт один счёт «123456789» с двумя бенефициарами по 50%, и тест бизнес-логики становится тривиальным.

**Грабли JPA.**
- `LazyInitializationException` вне транзакции.
- N+1 запросов при итерации коллекции без `JOIN FETCH`/`@EntityGraph`.
- `EAGER`-ассоциации размножаются: пара `EAGER` `@ManyToOne` в цепочке — и SELECT собирает 10 таблиц.
- `@GeneratedValue(IDENTITY)` не даёт использовать батчи в Hibernate (нужен id сразу после `insert`).

---

### 6.3. IoC и Dependency Injection — `10-spring-intro`

**Суть.** IoC (Inversion of Control) — «зависимости должен получать объект, а не искать сам». DI (Dependency Injection) — конкретный механизм IoC: контейнер знает граф зависимостей и передаёт готовые объекты в конструктор/сеттер.

**Принцип Голливуда:** «Don't call us — we'll call you». Не бин ищет `AccountRepository`, а контейнер приносит его.

#### 6.3.1. Виды DI

| Способ | Как выглядит | Плюсы | Минусы |
|---|---|---|---|
| **Constructor** | параметры конструктора | сразу видно обязательные зависимости, поле может быть `final`, объект не создать в невалидном состоянии | много параметров = запах «god object» |
| **Setter** | публичный setter, часто с `@Autowired` | опциональные зависимости, циклы | объект может быть частично-инициализированным |
| **Field** | `@Autowired` прямо на поле | компактно | нельзя протестировать без рефлексии, поле не может быть `final` |

**Constructor injection — рекомендованный дефолт.** Именно поэтому `RewardNetworkImpl` в проекте так устроен:

```java
// 10-spring-intro-solution/.../RewardNetworkImpl.java
public RewardNetworkImpl(AccountRepository ar, RestaurantRepository rr, RewardRepository rewr) {
    this.accountRepository = ar;
    this.restaurantRepository = rr;
    this.rewardRepository = rewr;
}
```

Со Spring 4.3+ единственный конструктор считается точкой инъекции без явной `@Autowired`.

#### 6.3.2. Composition Root

Точка в коде, где собирается граф объектов. В 10-м модуле это делается вручную в `setUp()` теста:

```java
// RewardNetworkImplTests.setUp()
AccountRepository accountRepo = new StubAccountRepository();
RewardRepository  rewardRepo  = new StubRewardRepository();
rewardNetwork = new RewardNetworkImpl(accountRepo, restaurantRepo, rewardRepo);
```

Пока таких «сборок» три-четыре — это норма. Когда их станет 30, поверх сборки уже ставят контейнер (12-й модуль).

**Грабли.**
- Circular dependency между двумя конструкторами — контейнер не сможет инстанцировать ни один. Решение: сеттер/`@Lazy`/пересобрать код.
- `@Autowired` на поле = плохая тестируемость. Придётся `ReflectionTestUtils.setField(...)` или `@InjectMocks`.

---

### 6.4. JavaConfig (`@Configuration`/`@Bean`) — `12-javaconfig-di`

**Суть.** Способ описать граф бинов в Java-коде вместо XML. `@Configuration` помечает класс как «сборочный конвейер», `@Bean` — фабричный метод, который возвращает объект-бин.

#### 6.4.1. Как это работает

Spring оборачивает `@Configuration`-класс **CGLIB-прокси**. Прокси перехватывает вызовы `@Bean`-методов и, если бин уже создан, возвращает закешированный экземпляр вместо повторного вызова кода. Именно поэтому:

```java
// 12-javaconfig-dependency-injection-solution/.../RewardsConfig.java
@Bean public RewardNetwork rewardNetwork() {
    return new RewardNetworkImpl(accountRepository(), // ← НЕ создаёт новый объект
                                 restaurantRepository(),
                                 rewardRepository());
}
@Bean public AccountRepository accountRepository() { ... }
```

Здесь `accountRepository()` вызывается напрямую, но Spring возвращает тот же singleton, что и в остальных местах — граф склеивается корректно.

**`@Configuration(proxyBeanMethods = false)`** — оптимизация, отключает CGLIB. Работает, только если сам не вызываешь `@Bean`-методы внутри конфига.

#### 6.4.2. Scope бинов

| Scope | Когда создаётся | Пример |
|---|---|---|
| `singleton` (default) | один раз на контейнер | `RewardNetworkImpl`, `DataSource` |
| `prototype` | каждый вызов `getBean()` создаёт новый | stateful builder |
| `request` | один на HTTP-request | web-only |
| `session` | один на HTTP-сессию | web-only |
| `application` | один на `ServletContext` | web-only |
| `websocket` | один на WS-сессию | web-only |

**Инъекция prototype в singleton** — типичная ошибка: singleton закеширует один экземпляр prototype и не создаст новый. Решения: `@Lookup`, `ObjectProvider<T>`, `Provider<T>`, `proxyMode = TARGET_CLASS`.

#### 6.4.3. `@Import` и композиция конфигов

Правильно разделять инфраструктуру и прикладной слой:

```java
// TestInfrastructureConfig.java
@Configuration
@Import(RewardsConfig.class)
public class TestInfrastructureConfig {
    @Bean public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .addScript("classpath:rewards/testdb/schema.sql")
            .addScript("classpath:rewards/testdb/data.sql")
            .build();
    }
}
```

`DataSource` в тестах — embedded HSQL, в проде — из JNDI. Прикладной `RewardsConfig` не знает, откуда возьмётся `DataSource`, — просит через конструктор.

#### 6.4.4. Жизненный цикл бина

1. Instantiate (конструктор).
2. Populate properties (сеттеры/поля).
3. `BeanNameAware.setBeanName`, `BeanFactoryAware.setBeanFactory`, `ApplicationContextAware.setApplicationContext`.
4. `BeanPostProcessor.postProcessBeforeInitialization` (для всех бинов, всех процессоров).
5. `@PostConstruct` → `InitializingBean.afterPropertiesSet` → `@Bean(initMethod)`.
6. `BeanPostProcessor.postProcessAfterInitialization` — вот здесь Spring и оборачивает бин в AOP-прокси.
7. Готов к использованию.
8. При остановке контекста: `@PreDestroy` → `DisposableBean.destroy` → `@Bean(destroyMethod)`.

`BeanFactoryPostProcessor` (`BFPP`) в отличие от `BeanPostProcessor` работает не с бинами, а с их **определениями** (`BeanDefinition`) — до создания. Пример: `PropertySourcesPlaceholderConfigurer` подставляет `${prop}` в метаданные бинов.

---

### 6.5. Стереотипные аннотации и Component Scan — `16-annotations`

**Суть.** Вместо ручного `@Bean` — навесить аннотацию-стереотип на класс, а Spring сам найдёт его при сканировании classpath.

#### 6.5.1. Иерархия стереотипов

- `@Component` — «просто бин».
- `@Service` — бизнес-логика (семантика, для документации; технически = `@Component`).
- `@Repository` — доступ к данным. **Важное отличие:** активирует `PersistenceExceptionTranslationPostProcessor`, который перехватывает `SQLException`/`PersistenceException` и превращает их в единую иерархию `DataAccessException` (unchecked). Приложение везде ловит `DataAccessException`, не завися от конкретной технологии.
- `@Controller` — MVC/REST.
- `@Configuration` — сам является `@Component`, поэтому тоже подхватывается сканом.

Можно делать **свои** стереотипы через мета-аннотации: `@MyService = @Component + @Transactional + @Loggable`.

#### 6.5.2. Component Scan

```java
// 16-annotations-solution/.../config/RewardsConfig.java
@Configuration
@ComponentScan("rewards.internal")
public class RewardsConfig { }
```

- Без `basePackages` — сканирует пакет самого класса.
- Фильтры: `includeFilters`, `excludeFilters` (по аннотации, типу, регексу, AspectJ-выражению, кастомному `TypeFilter`).
- Не рекурсивно скановать корень (`com`) — Spring будет ползать по всему classpath.

#### 6.5.3. Автовайринг

- **`@Autowired`** (Spring) — по типу, при амбигьюити — по имени.
- **`@Qualifier("beanName")`** — снимает неоднозначность.
- **`@Primary`** — «если непонятно, бери меня».
- **`@Resource(name=...)`** (JSR-250) — по имени в первую очередь.
- **`@Inject`** (JSR-330) — почти как `@Autowired`, без `required`.
- **`Optional<Foo>`** или `@Autowired(required=false)` — если зависимости может не быть.
- **`Collection<Handler>`**, **`Map<String, Handler>`** — Spring соберёт все бины типа `Handler`.

Пример из проекта — сеттер:

```java
// JdbcAccountRepository.java (16-annotations-solution)
@Autowired
public void setDataSource(DataSource dataSource) { this.dataSource = dataSource; }
```

Плюс явное имя бина: `@Repository("accountRepository")` — чтобы тесты, которые ищут по имени, не сломались от переименований.

#### 6.5.4. Lifecycle callbacks

```java
// JdbcRestaurantRepository.java
@PostConstruct
public void populateCache() { /* прогрев кэша при старте */ }

@PreDestroy
public void clearCache() { /* очистка при остановке */ }
```

`@PostConstruct` вызывается **после** DI, но **до** возврата бина потребителю. `@PreDestroy` — при штатном закрытии контекста (`SIGTERM`, `context.close()`, `close()`-shutdown hook). При `kill -9` вызвать не удастся.

**Грабли.**
- В сеттере, вызываемом Spring до `@PostConstruct`, нельзя обращаться к другим бинам, если это field-injection — они ещё не проинжектились.
- `@PreDestroy` не срабатывает для prototype-бинов (Spring не отслеживает их после создания).
- Циклическая ссылка `A ↔ B` через конструктор — не разрешается, через сеттер — разрешается ценой временного показа полусобранного бина.

---

### 6.6. AOP — `22-aop`

**Суть.** AOP (Aspect-Oriented Programming) — способ вынести **cross-cutting concerns** (логирование, транзакции, метрики, безопасность) из бизнес-кода в отдельные модули (аспекты).

#### 6.6.1. Терминология AspectJ

| Термин | Смысл |
|---|---|
| **Join Point** | точка в потоке выполнения, где потенциально можно вмешаться (в Spring AOP это только вызов метода публичного бина). |
| **Pointcut** | предикат «какие join point'ы меня интересуют» — обычно текстовое выражение. |
| **Advice** | код, который выполняется на подходящем join point (`@Before`, `@Around`, ...). |
| **Aspect** | класс с pointcut'ами и советами (`@Aspect`). |
| **Weaving** | процесс «вплетения» аспекта в целевой код. Может быть compile-time, load-time (LTW) или runtime (Spring AOP). |
| **Target** | сам целевой объект. |
| **Proxy** | обёртка вокруг target, которую видит потребитель. |
| **Introduction** | добавить новый интерфейс/поле сущему бину (`@DeclareParents` — не в этой лабе). |

#### 6.6.2. Spring AOP vs полный AspectJ

- **Spring AOP** — proxy-based, runtime. Работает только для Spring-бинов и только на вызовах через прокси. Достаточно для 95% случаев.
- **Full AspectJ** — compile-time / load-time weaving. Может перехватывать `new`, `set field`, private/final-методы, любой Java-код. Нужен `aspectjweaver`, отдельный компилятор `ajc` или Java-агент.

Проект использует Spring AOP: подключён `spring-aspects` (позволяет писать `@Aspect`/`@Before`), а `@EnableAspectJAutoProxy` включает механизм создания прокси.

#### 6.6.3. Два вида прокси

- **JDK Dynamic Proxy** — если у target есть интерфейс. Прокси реализует тот же интерфейс, все вызовы через рефлексию.
- **CGLIB** — если интерфейса нет (или явно `proxyTargetClass = true`). Создаётся подкласс target с переопределёнными методами. **Не работает для `final`-классов и `final`-методов**, потому что их нельзя переопределить.

Spring Boot 2.x+ по умолчанию использует CGLIB для всех AOP (`spring.aop.proxy-target-class=true`).

#### 6.6.4. Синтаксис pointcut'ов

```
execution(<модификаторы>? <возвращаемый_тип> <пакет.класс.метод>(<аргументы>) <throws>?)
```

`execution(public * rewards.internal.*.*Repository.find*(..))` разбирается так:
- `public` — только публичные;
- `*` (после `public`) — любой возвращаемый тип;
- `rewards.internal.*.*Repository` — классы с суффиксом `Repository` в пакетах уровня `rewards.internal.<что-то>`;
- `find*` — имя метода начинается на `find`;
- `(..)` — любое число аргументов.

**Полезные designators помимо `execution`:**

| Designator | Смысл |
|---|---|
| `within(rewards..*)` | все join point'ы в пакете |
| `this(Foo)` | прокси реализует `Foo` |
| `target(Foo)` | таргет — экземпляр `Foo` |
| `args(String, ..)` | первый аргумент — `String` |
| `@annotation(Transactional)` | метод помечен `@Transactional` |
| `@within(...)`, `@target(...)`, `@args(...)` | по аннотации класса / target'a / аргументов |
| `bean(accountRepository)` | конкретный бин по имени |

Combinable: `execution(...) && @annotation(Tx) && args(String,..)`.

**Именованные pointcut'ы для переиспользования:**
```java
@Pointcut("execution(public * rewards.internal..*Repository.*(..))")
private void anyRepositoryMethod() {}
@Before("anyRepositoryMethod()")
public void log(JoinPoint jp) { ... }
```

#### 6.6.5. Виды advice

| Аннотация | Момент срабатывания | Аргумент | Может изменить поток? |
|---|---|---|---|
| `@Before` | до метода | `JoinPoint` | нет (но может кинуть исключение) |
| `@AfterReturning(returning="r")` | после успешного возврата | результат | нет |
| `@AfterThrowing(throwing="e")` | после исключения | исключение | нет (пробрасывает дальше) |
| `@After` | всегда (как `finally`) | `JoinPoint` | нет |
| `@Around` | оборачивает вызов | `ProceedingJoinPoint` | **да** (может не вызвать `proceed()`, изменить аргументы, подменить результат, поймать исключение) |

**`@Around` = самый мощный, но и самый опасный.** Если забыл `proceed()`, метод никогда не вызовется.

#### 6.6.6. Пример из проекта

```java
// 22-aop-solution/.../LoggingAspect.java
@Aspect @Component
public class LoggingAspect {
    @Before("execution(public * rewards.internal.*.*Repository.find*(..))")
    public void implLogging(JoinPoint jp) {
        logger.info("Before advice - " + jp.getTarget().getClass()
                    + "; Executing " + jp.getSignature().getName() + "()");
    }

    @Around("execution(public * rewards.internal.*.*Repository.update*(..))")
    public Object monitor(ProceedingJoinPoint pjp) throws Throwable {
        Monitor m = monitorFactory.start(createJoinPointTraceName(pjp));
        try { return pjp.proceed(); }
        finally { m.stop(); logger.info("Around advice - " + m); }
    }
}
```

`DBExceptionHandlingAspect` подписан именно на `RewardDataAccessException`: тип параметра advice — это ещё один фильтр, а не только pointcut.

#### 6.6.7. `@EnableAspectJAutoProxy`

Регистрирует `AnnotationAwareAspectJAutoProxyCreator` — это `BeanPostProcessor`, который на этапе `postProcessAfterInitialization` смотрит, попадает ли бин под какой-нибудь advisor. Если да — подменяет ссылку в контейнере на прокси.

Именно поэтому в контейнер попадает **прокси**, а не оригинал: снаружи никто не видит разницы (интерфейс тот же), но каждый вызов идёт через цепочку interceptors.

#### 6.6.8. Грабли

- **Self-invocation.** `this.methodB()` внутри бина идёт мимо прокси → advice не сработает. Лечится вынесением в другой бин, `AopContext.currentProxy()` (некрасиво) или weave.
- **`private`/`final`/`static`-методы** не перехватываются Spring AOP.
- **Order of aspects.** `@Order(N)` или `Ordered` — меньше значит «внешнее». Актуально, когда логирование должно быть снаружи транзакции.
- **Return type advice-параметра фильтрует по типу.** Если параметр `RuntimeException`, то `Error` пройдёт мимо.

---

### 6.7. TestContext и профили — `24-test`

**Суть.** Spring TestContext Framework — интеграция Spring с JUnit/TestNG, которая поднимает `ApplicationContext` один раз на весь testsuite и переиспользует его между тестами.

#### 6.7.1. Ключевые аннотации

| Аннотация | Смысл |
|---|---|
| `@ExtendWith(SpringExtension.class)` | подключить TestContext к JUnit 5 |
| `@ContextConfiguration(classes=...)` | указать конфиги |
| `@SpringJUnitConfig(Config.class)` | «два в одном» — короткая форма |
| `@ActiveProfiles("dev","jdbc")` | активные профили для этого класса |
| `@DirtiesContext` | пометить: этот тест «портит» контекст, пересоздать |
| `@Sql("classpath:x.sql")` | выполнить SQL до/после метода |
| `@TestPropertySource` | inline-свойства или файл |
| `@DynamicPropertySource` | (Spring 5.2+) свойство, вычисляемое во время старта — незаменимо для Testcontainers |

Пример:

```java
// 24-test-solution/.../DevRewardNetworkTests.java
@SpringJUnitConfig(classes = TestInfrastructureConfig.class)
@ActiveProfiles({ "local", "jdbc" })
public class DevRewardNetworkTests {
    @Autowired private RewardNetwork rewardNetwork;
    @Test public void rewardForDining() { ... }
}
```

#### 6.7.2. Context caching

TestContext хешит контекст по ключу = {классы конфига + профили + ресурсы + инициализаторы}. Если у двух тестовых классов ключ одинаковый — контекст переиспользуется. Это даёт **огромный прирост** скорости на больших тестах.

**Что ломает кеш:**
- `@DirtiesContext` — принудительно пересобрать.
- `@MockBean`/`@SpyBean` — Spring создаёт новый контекст для каждого уникального набора мок-типов.
- Разные `@ActiveProfiles` — разные контексты.
- Разные `@TestPropertySource` — разные контексты.

#### 6.7.3. Profiles

```java
@Repository @Profile("jdbc")
class JdbcAccountRepository { ... }

@Repository @Profile("stub")
class StubAccountRepository { ... }
```

Активные профили передаются: `@ActiveProfiles`, `spring.profiles.active=...`, env var `SPRING_PROFILES_ACTIVE`, cmd-arg `--spring.profiles.active=...`.

**Полезные приёмы:**
- `@Profile("!prod")` — «во всех, кроме prod».
- `@Profile({"dev", "test"})` — «в любом из».
- `spring.profiles.group.local=jdbc,embedded` (Boot 2.4+) — группы профилей.
- Стандартные Spring Boot профили: `default` (когда ничего не активно).

**Грабли.**
- Забыть `@Profile` на альтернативной реализации → два бина одного типа → `NoUniqueBeanDefinitionException`.
- Профиль в `application.properties` через `spring.profiles.active` в самом profile-specific файле не сработает (уже поздно).

---

### 6.8. JdbcTemplate — `26-jdbc`

**Суть.** `JdbcTemplate` — реализация Template Method: инкапсулирует бойлерплейт (открыть Connection, PreparedStatement, обработать ResultSet, закрыть, поймать SQLException), а изменяемую логику принимает через колбэки.

#### 6.8.1. Что делает бесплатно

1. Открывает `Connection` (через `DataSourceUtils`, который учитывает транзакцию).
2. Готовит `PreparedStatement`, подставляет параметры.
3. Обрабатывает `ResultSet` через `RowMapper`/`ResultSetExtractor`.
4. Закрывает всё в `finally`.
5. **Переводит** `SQLException` в иерархию `DataAccessException`:
   - `BadSqlGrammarException` (SQL syntax)
   - `DataIntegrityViolationException` (unique/FK)
   - `DuplicateKeyException`
   - `DeadlockLoserDataAccessException`
   - и так далее.

Иерархия унифицирована для JDBC, JPA, MyBatis, Mongo — поэтому в бизнес-коде везде ловится общий тип.

#### 6.8.2. Основные методы

| Метод | Возвращает | Пример |
|---|---|---|
| `update(sql, params...)` | `int` (rows affected) | INSERT/UPDATE/DELETE |
| `queryForObject(sql, RowMapper, args)` | одну строку | `SELECT ... WHERE id = ?` |
| `queryForObject(sql, Class, args)` | скалярное значение | `SELECT COUNT(*) FROM ...` |
| `query(sql, RowMapper, args)` | `List<T>` | несколько строк |
| `query(sql, ResultSetExtractor, args)` | что вернёт extractor | сложные joins 1:N |
| `queryForList(sql, args)` | `List<Map<String,Object>>` | ad-hoc |
| `batchUpdate(sql, List<Object[]>)` | `int[]` | пакетные INSERT |

#### 6.8.3. `RowMapper` vs `ResultSetExtractor`

- **`RowMapper<T>.mapRow(ResultSet, int)`** — вызывается для каждой строки, возвращает объект строки. Идеально для «плоских» SELECT'ов.
- **`ResultSetExtractor<T>.extractData(ResultSet)`** — работает со всем `ResultSet` целиком. Нужен, когда одна логическая сущность разбита на несколько строк (JOIN 1:N). В проекте `AccountExtractor` собирает `Account` из джойна с `T_ACCOUNT_BENEFICIARY` и `T_ACCOUNT_CREDIT_CARD`.
- **`RowCallbackHandler.processRow(ResultSet)`** — не возвращает объект, побочный эффект (например, стриминг в файл).

#### 6.8.4. Пример из проекта

```java
// 26-jdbc-solution/.../JdbcRewardRepository.java
jdbcTemplate.update(
    "insert into T_REWARD (CONFIRMATION_NUMBER, REWARD_AMOUNT, REWARD_DATE, "
  + "ACCOUNT_NUMBER, DINING_MERCHANT_NUMBER, DINING_DATE, DINING_AMOUNT) "
  + "values (?, ?, ?, ?, ?, ?, ?)",
    confirmationNumber, contribution.getAmount().asBigDecimal(),
    SimpleDate.today().asDate(), contribution.getAccountNumber(),
    dining.getMerchantNumber(), dining.getDate().asDate(),
    dining.getAmount().asBigDecimal());
```

#### 6.8.5. `NamedParameterJdbcTemplate`

Обёртка сверху: `insert into T_REWARD ... values (:num, :amount, :date)` + `Map<String,Object>` или `MapSqlParameterSource` / `BeanPropertySqlParameterSource`. Понятнее, чем `?, ?, ?`.

**Грабли.**
- Порядок аргументов должен совпадать с `?`. Ошибка ловится только в рантайме.
- `queryForObject` кидает `EmptyResultDataAccessException`, если строк 0, и `IncorrectResultSizeDataAccessException`, если >1. Оборачивать в try/catch или `Optional`.
- Забыть `DataSourceTransactionManager` при использовании транзакций — `JdbcTemplate` возьмёт свой connection мимо транзакции.

---

### 6.9. Транзакции — `28-transactions`

**Суть.** `@Transactional` — декларативный способ обозначить границу транзакции. Реально транзакцией управляет `PlatformTransactionManager`, а `@Transactional` — это лишь метаданные, которые перехватываются AOP-прокси (`TransactionInterceptor`).

#### 6.9.1. ACID (напомнить кратко)

- **A**tomicity — всё или ничего.
- **C**onsistency — переход из валидного состояния в валидное.
- **I**solation — параллельные транзакции не мешают друг другу (уровни ниже).
- **D**urability — коммит переживает падение.

#### 6.9.2. Как работает `@Transactional`

```java
// 28-transactions-solution/.../RewardNetworkImpl.java
@Transactional
public RewardConfirmation rewardAccountFor(Dining dining) { ... }
```

Что происходит при вызове через прокси:

1. `TransactionInterceptor` спрашивает у `TransactionSynchronizationManager`, есть ли активная транзакция.
2. По `propagation` решает: присоединиться / начать новую / отложить / упасть.
3. У `PlatformTransactionManager` берёт `TransactionStatus` — внутри это `connection.setAutoCommit(false)`.
4. Connection кладётся в `ThreadLocal` под ключом `DataSource`. `JdbcTemplate` через `DataSourceUtils.getConnection(ds)` найдёт именно этот connection.
5. Вызывается целевой метод.
6. Return без исключения → `commit`. Исключение → `rollback` (по правилам).

**`@EnableTransactionManagement`** регистрирует `TransactionInterceptor` + `InfrastructureAdvisorAutoProxyCreator`.

#### 6.9.3. Propagation (7 уровней)

| Уровень | Есть активная TX | Нет активной TX |
|---|---|---|
| `REQUIRED` (default) | присоединиться | создать новую |
| `REQUIRES_NEW` | приостановить внешнюю, создать свою | создать новую |
| `SUPPORTS` | использовать | без TX |
| `NOT_SUPPORTED` | приостановить, работать без TX | работать без TX |
| `MANDATORY` | использовать | `IllegalTransactionStateException` |
| `NEVER` | `IllegalTransactionStateException` | работать без TX |
| `NESTED` | savepoint внутри | новая (для JDBC — savepoint) |

`REQUIRES_NEW` реально важен, когда во внешнюю транзакцию нельзя вложить, а изменения нужно фиксировать независимо (лог аудита).

#### 6.9.4. Isolation (5 уровней)

- `READ_UNCOMMITTED` — dirty reads.
- `READ_COMMITTED` — no dirty, но possible non-repeatable read.
- `REPEATABLE_READ` — no non-repeatable, но phantom read.
- `SERIALIZABLE` — как последовательно.
- `DEFAULT` — что настроено в БД (обычно `READ_COMMITTED` для Postgres/Oracle, `REPEATABLE_READ` для MySQL InnoDB).

#### 6.9.5. Rollback rules

По умолчанию: **только** `RuntimeException` и `Error` вызывают rollback. Checked-исключения — **не** вызывают. Изменить: `@Transactional(rollbackFor = IOException.class)` или `noRollbackFor = MyException.class`.

**Практика:** декларативно указать `rollbackFor = Exception.class` в самом верхнем сервисе, если проект использует checked-исключения широко.

#### 6.9.6. Прочие атрибуты

- `readOnly = true` — hint (JDBC-драйверу и Hibernate). Hibernate не будет делать dirty check. НО: это не гарантирует, что физически нельзя записать.
- `timeout = 5` — секунд до принудительного rollback.

#### 6.9.7. Programmatic transactions

Иногда декларатив не подходит (нужно контролировать точку commit'а внутри метода):

```java
TransactionTemplate tx = new TransactionTemplate(txManager);
tx.execute(status -> {
    ...
    if (bad) status.setRollbackOnly();
    return result;
});
```

#### 6.9.8. Грабли

- **Self-invocation.** `this.method()` внутри бина мимо прокси → транзакция не открывается. Классический баг: `save(x)` без `@Transactional` вызывает `internalSave()` с `@Transactional` → ничего не открылось.
- **`@Transactional` на `private` / `protected`** — Spring AOP не перехватывает.
- **Разные `TransactionManager`'ы** (JPA + JDBC) — по умолчанию Spring возьмёт первый, для явного: `@Transactional("jpaTx")`.
- **Проверка**: включить `logging.level.org.springframework.transaction=TRACE` — видно begin/commit/rollback.
- **Checked exception без rollbackFor** — коммит и молчаливая потеря данных.

---

### 6.10. Spring Boot overview — `30-jdbc-boot`

**Суть.** Spring Boot = Spring Framework + convention over configuration + starters + auto-configuration + executable jar.

#### 6.10.1. `@SpringBootApplication` разбирается на

- `@SpringBootConfiguration` (по сути `@Configuration`).
- `@EnableAutoConfiguration` — включает автоконфигурацию.
- `@ComponentScan` — сканирует пакет самого класса и вложенные (**никогда** не кладите main-класс в корневой пакет).

#### 6.10.2. Что происходит в `SpringApplication.run(...)`

1. Создаётся `SpringApplication`, определяется тип приложения (SERVLET / REACTIVE / NONE) по classpath.
2. Загружаются `SpringApplicationRunListeners`.
3. Готовится `Environment` (свойства из `application.properties`, env vars, cmd-args, профили).
4. Печатается баннер.
5. Создаётся `ApplicationContext` (для web — `AnnotationConfigServletWebServerApplicationContext`).
6. Регистрируются инициализаторы (`ApplicationContextInitializer`).
7. Загружаются источники (`@SpringBootApplication`-класс).
8. `refresh()` — создаются все бины, срабатывает auto-configuration.
9. Запускаются `ApplicationRunner`/`CommandLineRunner`.
10. Приложение работает.

#### 6.10.3. Starters

Starter — это pom-модуль без кода, только зависимости. `spring-boot-starter-jdbc` тянет `spring-jdbc`, `HikariCP`, `spring-boot`, `spring-core`. Разработчику не нужно знать точные версии — они выравниваются через BOM `spring-boot-dependencies`.

Официальные — `spring-boot-starter-*`. Community — `something-spring-boot-starter` (обратный порядок).

#### 6.10.4. Executable JAR

Boot собирает **fat JAR** (не «uber», а свой формат `BOOT-INF/lib/`). Внутри — специальный ClassLoader (`LaunchedURLClassLoader`), который умеет читать jar-in-jar. Запуск: `java -jar app.jar` — работает без Tomcat, потому что встроенный Tomcat/Netty/Undertow идёт внутри.

#### 6.10.5. Пример из проекта

```java
// 30-jdbc-boot-solution/.../JdbcBootApplication.java
@SpringBootApplication
public class JdbcBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(JdbcBootApplication.class, args);
    }

    @Bean
    CommandLineRunner commandLineRunner(JdbcTemplate jdbcTemplate) {
        return args -> System.out.println("Hello, there are "
            + jdbcTemplate.queryForObject("SELECT count(*) FROM T_ACCOUNT", Long.class)
            + " accounts");
    }
}
```

`JdbcTemplate` создался «сам» — потому что Boot увидел `hsqldb` на classpath (`DataSourceAutoConfiguration`), поднял embedded DataSource и создал `JdbcTemplate` (`JdbcTemplateAutoConfiguration`).

**`CommandLineRunner` vs `ApplicationRunner`.** Оба выполняются после старта. Первый принимает `String[]`, второй — `ApplicationArguments` (умеет отличать опции `--name=value` от positional).

---

### 6.11. Auto-configuration и `@ConfigurationProperties` — `32-jdbc-autoconfig`

**Суть.** Auto-configuration = набор `@Configuration`-классов, которые применяются только при выполнении условий (класс на classpath, бина ещё нет и т.д.). Пользователь может **отменить** любое умолчание, объявив свой бин.

#### 6.11.1. Как auto-config подхватывается

- **Boot 2.x:** `META-INF/spring.factories`:
  ```
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfig
  ```
- **Boot 3.x:** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:
  ```
  com.example.MyAutoConfig
  ```

`@EnableAutoConfiguration` (внутри `@SpringBootApplication`) читает эти файлы и импортирует все перечисленные конфиги.

#### 6.11.2. `@ConditionalOn*` — семейство

| Аннотация | Условие |
|---|---|
| `@ConditionalOnClass` | класс есть в classpath |
| `@ConditionalOnMissingClass` | класса нет |
| `@ConditionalOnBean(type/name)` | в контексте уже есть такой бин |
| `@ConditionalOnMissingBean` | бина ещё нет — идеально для «умных дефолтов» |
| `@ConditionalOnProperty(name, havingValue)` | свойство имеет значение |
| `@ConditionalOnResource` | файл есть на classpath |
| `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | тип приложения |
| `@ConditionalOnExpression` | SpEL |
| `@ConditionalOnJava` | версия Java |

Порядок: `@AutoConfigureBefore(...)`, `@AutoConfigureAfter(...)`, `@AutoConfigureOrder(N)`.

#### 6.11.3. Как отладить

```
debug=true
```
в `application.properties` — Boot напечатает **Conditions Evaluation Report**: какие autoconfig'и сработали (positive), какие пропущены (negative) и почему (`OnClassCondition did not find required class 'X'`).

#### 6.11.4. Исключение конкретного autoconfig

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```
или в `application.properties`:
```
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```
(В проекте это как раз сделано в `32-jdbc-autoconfig-solution/application.properties`.)

#### 6.11.5. `@ConfigurationProperties`

Type-safe маппинг свойств на POJO с валидацией.

```java
// RewardsRecipientProperties.java
@ConfigurationProperties(prefix = "rewards.recipient")
public class RewardsRecipientProperties {
    private String name;
    private int age;
    // getters/setters
}
```

Активация:
```java
@SpringBootApplication
@EnableConfigurationProperties(RewardsRecipientProperties.class)
@Import(RewardsConfig.class)
public class RewardsApplication { ... }
```

**`application.properties`:**
```properties
rewards.recipient.name=John Doe
rewards.recipient.age=10
```

**Relaxed binding.** `rewards.recipient.name`, `REWARDS_RECIPIENT_NAME`, `rewards.recipient.Name`, `rewards.recipient.NAME`, `rewards.recipient.name` — Boot биндит все эти формы к одному полю. Для env vars — только upper-snake case.

**`@ConfigurationProperties` vs `@Value`.**

| | `@ConfigurationProperties` | `@Value("${x}")` |
|---|---|---|
| Type-safe | да, автоконверсия | нет (по типу поля) |
| Валидация | `@Validated` + JSR-303 | нет |
| Nested/List | да | сложно |
| SpEL | нет | да |
| IDE autocomplete | да (нужна spring-boot-configuration-processor) | нет |

#### 6.11.6. Порядок PropertySource'ов (упрощённо)

1. `SPRING_APPLICATION_JSON` env / cmd.
2. Command line arguments (`--server.port=8081`).
3. Env vars.
4. `application-{profile}.properties`.
5. `application.properties`.
6. `@PropertySource`.
7. Default properties.

Что выше — переопределяет то, что ниже.

---

### 6.12. Собственный стартер/auto-configuration — `33-autoconfig-helloworld`

**Суть.** Написать переиспользуемый модуль так, чтобы клиент положил зависимость — и всё «просто заработало» с разумными дефолтами.

#### 6.12.1. Разделение модулей

- **library** — бизнес-код, никаких Spring-аннотаций.
- **autoconfigure** — `@Configuration` с `@ConditionalOn*`.
- **starter** — pom без кода, тянет library + autoconfigure + другие зависимости.

В лабе три модуля: `hello-lib` (`HelloService`), `hello-starter` (autoconfig), `hello-app` (потребитель).

#### 6.12.2. Пример

```java
// hello-starter/.../HelloAutoConfig.java
@Configuration
@ConditionalOnClass(HelloService.class)
public class HelloAutoConfig {
    @ConditionalOnMissingBean(HelloService.class)
    @Bean
    HelloService helloService() {
        return new TypicalHelloService();
    }
}
```

+ регистрация:
```
# hello-starter/src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.starter.HelloAutoConfig
```

Правильный дизайн: `@ConditionalOnClass` — не «зависимость есть в pom», а «класс присутствует в classpath». Это позволяет starter'у гибко реагировать: положил hsqldb — embedded, положил jdbc-driver другой БД — не встраивается.

`@ConditionalOnMissingBean` — паттерн «дефолт, который можно перебить». Если пользователь объявил свой `@Bean HelloService`, autoconfig отступает.

#### 6.12.3. Тестирование auto-configuration

```java
new ApplicationContextRunner()
    .withUserConfiguration(HelloAutoConfig.class)
    .run(context -> assertThat(context).hasSingleBean(HelloService.class));
```

`ApplicationContextRunner` — облегчённый способ поднять контейнер с разными условиями и проверить, что autoconfig ведёт себя правильно (в том числе negative-кейсы).

#### 6.12.4. Соглашения об именовании

- Официальные Spring Boot: `spring-boot-starter-<name>`.
- Community: `<name>-spring-boot-starter`.
- Не хардкодьте версии внутри — используйте BOM.

---

### 6.13. Spring Data JPA — `34-spring-data-jpa`

**Суть.** Разработчик пишет интерфейс, Spring Data генерирует реализацию в рантайме.

#### 6.13.1. Иерархия репозиториев

```
Repository<T, ID>                         — маркер, ничего не даёт
   ↑
CrudRepository<T, ID>                     — save, findById, findAll, deleteById, count
   ↑
PagingAndSortingRepository<T, ID>         — findAll(Sort), findAll(Pageable)
   ↑
JpaRepository<T, ID>                      — flush, saveAndFlush, deleteInBatch, getById
```

Наследуйте только **нужный** уровень. В лабе намеренно взято минимальное `Repository<Account, Long>`:

```java
// 34-spring-data-jpa-solution/.../AccountRepository.java
public interface AccountRepository extends Repository<Account, Long> {
    Account findByCreditCardNumber(String creditCardNumber);
}
```

`Repository` не даёт никаких методов сам по себе — Spring Data сгенерирует только то, что явно объявлено. Это спасает от случайного `deleteAll()` в UI.

#### 6.13.2. Query derivation

Имя метода парсится: `find[Distinct]By<Property>[And|Or|Between|LessThan|Like|IgnoreCase|In]...[OrderBy...]`.

- `findByCreditCardNumber(String)` → JPQL `SELECT a FROM Account a WHERE a.creditCards.number = :p`.
- `findByNameContainingIgnoreCase(String)` → `WHERE lower(a.name) LIKE lower(concat('%', :p, '%'))`.
- `findTop3ByOrderByCreatedDesc()` → `LIMIT 3`.
- `existsByNumber(String)` → `SELECT count(...) > 0`.
- `deleteByNumber(String)` — модифицирующий, требует `@Modifying + @Transactional`.

#### 6.13.3. `@Query`

Когда имени не хватает:

```java
@Query("SELECT a FROM Account a WHERE size(a.beneficiaries) = :n")
List<Account> findByBeneficiaryCount(@Param("n") int n);
```

Native SQL: `@Query(value = "SELECT * FROM t_account WHERE ...", nativeQuery = true)`.

#### 6.13.4. Активация

```java
@SpringBootApplication
@EnableJpaRepositories(basePackages = "rewards.internal")
public class JpaApplication { ... }
```

С Spring Boot можно вообще без `@EnableJpaRepositories` — autoconfig подхватит.

#### 6.13.5. Прочее

- **Projections:** интерфейсные (`interface AccountSummary { String getName(); Long getEntityId(); }`) и класс-based DTO.
- **`Pageable`, `Page<T>`, `Slice<T>`, `Sort`** — стандартные типы для пагинации.
- **`Specifications`** — программные критерии (JPA Criteria обёртка).
- **`Auditing`** (`@CreatedDate`, `@LastModifiedBy`) — с `@EnableJpaAuditing`.
- **Транзакции по умолчанию:** `find*`-методы генерируются с `readOnly = true`; всё остальное — обычная транзакция. Прикладной сервис должен явно оборачивать бизнес-операции в `@Transactional`.

#### 6.13.6. Грабли

- **N+1 problem.** `findAll()` → потом в цикле `account.getBeneficiaries()` — по одному SELECT на каждый счёт. Решение: `@Query(... JOIN FETCH ...)`, `@EntityGraph(attributePaths = "beneficiaries")`.
- **`getById` (или `getReference`) vs `findById`.** Первый возвращает proxy без запроса — падает при обращении, если не существует. Второй — `Optional<T>` с реальным SELECT.
- **`save()` для detached-сущности** — делает `MERGE`, не `INSERT`. Может неожиданно перезаписать данные.
- **Lazy-геттеры вне транзакции** — `LazyInitializationException`.

---

### 6.14. Spring MVC — `36-mvc`

**Суть.** Реализация паттерна Front Controller. Один сервлет (`DispatcherServlet`) принимает все запросы и делегирует их подходящему `@Controller`.

#### 6.14.1. Поток запроса

1. HTTP → `DispatcherServlet.doDispatch`.
2. `HandlerMapping` находит `HandlerMethod` (метод контроллера) по URL/methods/headers.
3. `HandlerAdapter` вызывает метод, попутно резолвя аргументы (`@PathVariable`, `@RequestParam`, `@RequestBody`, `Model`, `Principal`, ...) через `HandlerMethodArgumentResolver`.
4. Метод возвращает `String` (имя view), `ModelAndView`, `ResponseEntity`, POJO (для REST).
5. Если это имя view — `ViewResolver` резолвит его в `View`, `View.render(model, ...)` пишет HTML в response.
6. Если это POJO/`@ResponseBody` — `HttpMessageConverter` сериализует в JSON/XML.
7. `HandlerExceptionResolver` перехватывает исключения (в том числе `@ExceptionHandler`).

Между шагами — цепочка `HandlerInterceptor` (`preHandle`, `postHandle`, `afterCompletion`).

#### 6.14.2. Аннотации

| Аннотация | Для чего |
|---|---|
| `@Controller` | обычный MVC-контроллер (view-based) |
| `@RestController` | `@Controller + @ResponseBody` на всех методах |
| `@RequestMapping` | базовый маппинг (path, method, headers, produces, consumes) |
| `@GetMapping / @PostMapping / @PutMapping / @DeleteMapping / @PatchMapping` | shortcut'ы |
| `@PathVariable` | `/users/{id}` |
| `@RequestParam` | `?name=X` |
| `@RequestHeader` | `Accept: ...` |
| `@RequestBody` | тело запроса → объект |
| `@ResponseBody` | объект → тело ответа (JSON) |
| `@ModelAttribute` | forms-биндинг, метод-«наполнитель модели» |
| `@SessionAttributes` | атрибуты, живущие в HTTP-сессии |

#### 6.14.3. Пример из проекта

```java
// 36-mvc-solution/.../AccountController.java
@RestController
public class AccountController {
    private final AccountManager accountManager;
    @Autowired
    public AccountController(AccountManager accountManager) { ... }

    @GetMapping("/accounts/{entityId}")
    public Account accountDetails(@PathVariable("entityId") long id) {
        return accountManager.getAccount(id);
    }

    @GetMapping("/accounts")
    public List<Account> accountList() { return accountManager.getAllAccounts(); }
}
```

Шаблон `index.html` использует Mustache; статические ресурсы — из `/static/`.

#### 6.14.4. ViewResolver

- `InternalResourceViewResolver` — JSP.
- `ThymeleafViewResolver`, `MustacheViewResolver`, `FreeMarkerViewResolver` — по одному имени и файлу шаблона.
- `ContentNegotiatingViewResolver` — по `Accept`-заголовку выбирает JSON/XML/HTML.

#### 6.14.5. WebMvcConfigurer

Точка расширения — реализация интерфейса:
- `addFormatters` — Converter, Formatter (String→SimpleDate, например).
- `addInterceptors` — HandlerInterceptors.
- `configureContentNegotiation` — MediaType strategy.
- `addResourceHandlers` — статика.
- `addViewControllers` — маппинг URL → view без контроллера.

**Грабли.**
- Забыть `@RequestBody` — Spring поймёт объект как биндинг form-полей.
- `@PathVariable("id") long id` — если параметр не число, `MethodArgumentTypeMismatchException` → 400.
- `redirect:/x` vs `forward:/x` — первый шлёт 302, второй — server-side dispatch.
- Стандартные валидаторы (`@Valid`) требуют starter-validation с Boot 2.3+.

---

### 6.15. REST WS — `38-rest-ws`

**Суть.** REST-контроллер возвращает **данные**, а не имя view. Сериализация в JSON/XML — задача `HttpMessageConverter`.

#### 6.15.1. Принципы REST (Fielding)

1. **Client-server.** Разделение UI и логики.
2. **Stateless.** Каждый запрос самодостаточен.
3. **Cacheable.** Ответы явно кешируемы или нет.
4. **Uniform interface.**
   - Resources идентифицируются URI (`/accounts/1`).
   - Representations (JSON/XML) отделены от resource.
   - Self-descriptive messages (Content-Type, Accept).
   - HATEOAS — навигация через ссылки.
5. **Layered system.** Прозрачные прокси.
6. **Code on demand** (опциональный).

#### 6.15.2. HTTP-методы и коды

| Метод | Значение | Idempotent? | Safe? |
|---|---|---|---|
| GET | получить | да | да |
| POST | создать/действие | нет | нет |
| PUT | заменить полностью | да | нет |
| PATCH | частично обновить | обычно нет | нет |
| DELETE | удалить | да | нет |

Типовые коды: 200, 201 (created + `Location`), 204 (no content), 400 (bad request), 401, 403, 404, 405, 409 (conflict), 500.

#### 6.15.3. `HttpMessageConverter`

Список конвертеров подключается автоматически по наличию библиотек:
- Jackson (`MappingJackson2HttpMessageConverter`) → JSON.
- `MappingJackson2XmlHttpMessageConverter` (при `jackson-dataformat-xml`) → XML.
- `StringHttpMessageConverter`, `FormHttpMessageConverter`, `ByteArrayHttpMessageConverter`.

`Accept: application/json` → JSON, `Accept: application/xml` → XML — content negotiation прозрачный.

#### 6.15.4. `ResponseEntity`

Полный контроль над status, headers, body:

```java
// 38-rest-ws-solution/.../AccountController.java
@PostMapping("/accounts")
public ResponseEntity<Void> createAccount(@RequestBody Account newAccount) {
    Account account = accountManager.save(newAccount);
    return entityWithLocation(account.getEntityId()); // строит 201 + Location
}
```

`ServletUriComponentsBuilder.fromCurrentRequestUri().path("/{id}").buildAndExpand(id).toUri()` — безопасно строит абсолютный URI даже за reverse proxy (если правильно настроен `ForwardedHeaderFilter`).

#### 6.15.5. Обработка ошибок

- **`@ExceptionHandler`** на самом контроллере — локально.
- **`@ControllerAdvice`** / **`@RestControllerAdvice`** — глобально.
- **`ResponseStatusException`** (Spring 5+) — кинуть с готовым статусом.
- **`ProblemDetail`** (Spring 6, RFC 7807) — стандартный JSON-формат ошибки: `type`, `title`, `status`, `detail`, `instance`.

#### 6.15.6. Валидация

```java
@PostMapping("/accounts")
public ResponseEntity<Void> create(@Valid @RequestBody AccountDto dto) { ... }
```
+ на DTO: `@NotBlank`, `@Email`, `@Size(min=1, max=100)`. Ошибка → `MethodArgumentNotValidException` → 400.

#### 6.15.7. Клиенты

- `RestTemplate` — синхронный, deprecated for new development, но всё ещё широко.
- `WebClient` — реактивный, подходит и для sync (`.block()`).
- `RestClient` (Spring 6.1+) — новый sync-клиент с fluent API.
- `HttpInterface` (Spring 6+) — декларативные клиенты вроде Feign.

**Грабли.**
- Забыть `produces = APPLICATION_JSON_VALUE` — Spring выберет XML, если `jackson-xml` в classpath.
- `PUT` с частичным телом — по семантике заменяет ВСЁ; для частичного — `PATCH`.
- Возвращать сущность JPA из контроллера — тянет всю ленивую графику, N+1, security-поля. Правильно — DTO.

---

### 6.16. Testing Spring Boot — `40-boot-test`

**Суть.** Три слоя тестов: юнит (без Spring) → slice (часть контекста) → integration (весь контекст, иногда с реальным HTTP).

#### 6.16.1. Юнит (без Spring)

Самый быстрый — простые Mockito или stub-объекты, никакого контейнера.

```java
// 40-boot-test-solution/.../AccountControllerTests.java
public class AccountControllerTests {
    private AccountController controller;
    @BeforeEach
    public void setUp() { controller = new AccountController(new StubAccountManager()); }
    @Test public void accountDetails() { ... }
}
```

Здесь контроллер вызывается напрямую — сериализация/маршрутизация не тестируется, только логика.

#### 6.16.2. `@WebMvcTest` — slice

Поднимает только web-слой: `DispatcherServlet`, `@Controller`, `HandlerMapping`, `HttpMessageConverter`, `ControllerAdvice`. **Все** сервисы/репозитории надо предоставить как `@MockBean`.

```java
// 40-boot-test-solution/.../AccountControllerBootTests.java
@WebMvcTest(AccountController.class)
public class AccountControllerBootTests {
    @Autowired private MockMvc mockMvc;
    @MockBean  private AccountManager accountManager;

    @Test
    public void accountDetails() throws Exception {
        given(accountManager.getAccount(anyLong())).willReturn(new Account("1234567890", "John Doe"));
        mockMvc.perform(get("/accounts/0")).andExpect(status().isOk());
    }
}
```

**MockMvc** не поднимает Tomcat — вызывает `DispatcherServlet` напрямую в памяти. Быстро.

#### 6.16.3. `@SpringBootTest` — полный интеграционный

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class FullIntegrationTests {
    @Autowired TestRestTemplate http;
    @LocalServerPort int port;
    ...
}
```

Варианты `webEnvironment`:
- `MOCK` (default) — только `MockMvc`, без сервера.
- `RANDOM_PORT` / `DEFINED_PORT` — реальный Tomcat.
- `NONE` — не web.

#### 6.16.4. Другие slice-аннотации

- `@DataJpaTest` — только JPA-слой, embedded БД, транзакция + rollback после каждого теста.
- `@JsonTest` — тестирование сериализации Jackson.
- `@RestClientTest` — тестирование `RestTemplate`-клиентов через `MockRestServiceServer`.

#### 6.16.5. `@MockBean` vs `@Mock`

- `@Mock` (Mockito) — просто создаёт мок, но не кладёт его в Spring-контекст.
- `@MockBean` — создаёт мок И заменяет им бин в контейнере. Именно то, что нужно для `@WebMvcTest`.
- **Внимание:** `@MockBean` инвалидирует context cache. Много `@MockBean` = много пересозданных контекстов = медленно.

#### 6.16.6. Test slices под капотом

Slice-аннотации отключают большинство autoconfig'ов через `@AutoConfigure*` мета-аннотации. Например, `@WebMvcTest` включает `WebMvcAutoConfiguration`, но НЕ включает `DataSourceAutoConfiguration`.

Если тесту не хватает какого-то autoconfig — добавить `@ImportAutoConfiguration(Xxx.class)` или `@Import`.

---

### 6.17. Spring Security — `42-security-rest`

**Суть.** Библиотека для аутентификации (кто ты) и авторизации (что тебе можно). Работает через servlet-фильтры и AOP.

#### 6.17.1. Аутентификация vs авторизация

- **Authentication.** Проверка личности (login+password → успех/провал).
- **Authorization.** Проверка прав (может ли этот user делать это действие).

#### 6.17.2. Security filter chain

При старте Spring Security регистрирует один сервлет-фильтр `springSecurityFilterChain` (`DelegatingFilterProxy`), внутри которого — **цепочка** фильтров:

1. `SecurityContextPersistenceFilter` — восстанавливает `SecurityContext` из сессии.
2. `WebAsyncManagerIntegrationFilter`.
3. `HeaderWriterFilter` — добавляет security-заголовки.
4. `CsrfFilter`.
5. `LogoutFilter`.
6. `UsernamePasswordAuthenticationFilter` (для form-login).
7. `BasicAuthenticationFilter` — HTTP Basic.
8. `BearerTokenAuthenticationFilter` — JWT/OAuth2 (когда подключено).
9. `RequestCacheAwareFilter`.
10. `SecurityContextHolderAwareRequestFilter`.
11. `AnonymousAuthenticationFilter` — если не аутентифицирован, создаёт anonymous-principal.
12. `SessionManagementFilter`.
13. `ExceptionTranslationFilter` — ловит `AccessDeniedException` / `AuthenticationException` и превращает в 401/403.
14. `FilterSecurityInterceptor` / `AuthorizationFilter` (Spring Security 6) — финальная проверка прав.

#### 6.17.3. Ключевые компоненты

- **`SecurityContextHolder`** — `ThreadLocal` с текущим `SecurityContext`.
- **`Authentication`** — «этот запрос принадлежит такому-то principal с такими-то authorities».
- **`UserDetailsService`** — «дай пользователя по имени» (для form/basic).
- **`UserDetails`** — сам пользователь: имя, пароль (закодированный), authorities.
- **`GrantedAuthority`** — право («ROLE_USER», «READ_ACCOUNTS»).
- **`PasswordEncoder`** — как хешируется пароль. Практически всегда `DelegatingPasswordEncoder`, распознающий префиксы `{bcrypt}`, `{pbkdf2}`, ....

#### 6.17.4. Конфигурация в проекте

```java
// 42-security-rest-solution/.../RestSecurityConfig.java
@Configuration
@EnableMethodSecurity
public class RestSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers(HttpMethod.GET,    "/accounts/**").hasAnyRole("USER","ADMIN","SUPERADMIN")
                .requestMatchers(HttpMethod.PUT,    "/accounts/**").hasAnyRole("ADMIN","SUPERADMIN")
                .requestMatchers(HttpMethod.POST,   "/accounts/**").hasAnyRole("ADMIN","SUPERADMIN")
                .requestMatchers(HttpMethod.DELETE, "/accounts/**").hasRole("SUPERADMIN")
                .anyRequest().denyAll())
            .httpBasic(withDefaults())
            .csrf(CsrfConfigurer::disable);
        return http.build();
    }
    // ... UserDetailsService, PasswordEncoder
}
```

`SecurityFilterChain` как бин — современный способ (заменил `WebSecurityConfigurerAdapter` в Spring Security 6).

#### 6.17.5. `hasRole` vs `hasAuthority`

- `hasRole("ADMIN")` — проверяет, есть ли authority с префиксом `ROLE_` → `ROLE_ADMIN`.
- `hasAuthority("ROLE_ADMIN")` — то же самое, но без магического префикса.
- `User.roles("ADMIN")` — на самом деле добавляет `ROLE_ADMIN` в GrantedAuthorities.

#### 6.17.6. CSRF

**CSRF (Cross-Site Request Forgery)** — атака, когда чужой сайт делает от твоего имени POST на банк, пользуясь твоей активной сессионной кукой.

**Правило:**
- Есть куки/сессия → CSRF **включён**, каждый небезопасный запрос требует token.
- Stateless REST (Bearer token в `Authorization`) → CSRF можно выключить.

Отсюда `csrf(CsrfConfigurer::disable)` в конфиге REST-модуля — тут нет сессии.

#### 6.17.7. Method security

`@EnableMethodSecurity` (заменил `@EnableGlobalMethodSecurity`) включает:

- **`@PreAuthorize("hasRole('ADMIN')")`** — до вызова метода.
- **`@PostAuthorize("returnObject.owner == authentication.name")`** — после (можно проверить result).
- **`@Secured("ROLE_ADMIN")`** — старый стиль.
- **`@RolesAllowed`** — JSR-250.

Работают через AOP (те же грабли: self-invocation, private-методы).

#### 6.17.8. Session management

`http.sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))` — не создавать HttpSession. Обязательно для JWT/OAuth2 Resource Server.

**Грабли.**
- Забыть `PasswordEncoder` → `IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"`.
- Отключить CSRF, оставив кукинг-сессию → уязвимость.
- `permitAll()` до `authenticated()` — порядок в цепочке matcher'ов важен, первый совпавший выигрывает.
- Использовать `hasRole("ROLE_ADMIN")` — двойной префикс: Spring добавит ещё один `ROLE_`, получится `ROLE_ROLE_ADMIN`.

---

### 6.18. Actuator — `44-actuator`

**Суть.** Стандартизированные endpoints для observability: здоровье, метрики, конфигурация, мэппинги, дампы.

#### 6.18.1. Стандартные endpoints

| Endpoint | Что показывает |
|---|---|
| `/actuator/health` | «up/down» + компоненты (диск, БД, ping, custom) |
| `/actuator/info` | build-info, git-info, кастомные `info.*` |
| `/actuator/metrics` | список метрик через Micrometer |
| `/actuator/metrics/{name}` | значение конкретной метрики (с тегами) |
| `/actuator/env` | ConfigurableEnvironment |
| `/actuator/configprops` | все `@ConfigurationProperties`-бины |
| `/actuator/beans` | граф бинов |
| `/actuator/mappings` | все RequestMapping'и |
| `/actuator/conditions` | Conditions Evaluation Report |
| `/actuator/loggers` (+ POST) | посмотреть / изменить уровни логирования в рантайме |
| `/actuator/threaddump` | thread dump |
| `/actuator/heapdump` | скачать hprof |
| `/actuator/prometheus` | текстовый экспорт метрик |
| `/actuator/httptrace` | последние N HTTP-запросов (2.x deprecated, → HttpExchanges в 3.x) |

#### 6.18.2. Exposure

По умолчанию через HTTP наружу открыты только `/health` и `/info`. Всё остальное — доступно, но требует явного включения:

```properties
# 44-actuator-solution/.../application.properties
management.endpoints.web.exposure.include=*
management.info.java.enabled=true
management.info.env.enabled=true
info.restaurant.location=New York
info.restaurant.discountPercentage=10
```

Отдельно — jmx exposure (`management.endpoints.jmx.exposure.*`).

#### 6.18.3. Health-группы

Позволяют группировать чеки для разных целей (например, K8s liveness vs readiness):

```properties
management.endpoint.health.group.system.include=diskSpace,db
management.endpoint.health.group.web.include=ping
management.endpoint.health.group.application.include=restaurantHealthCheck,restaurantHealthCheck2
management.endpoint.health.group.application.status.order=NO_RESTAURANTS,DOWN,UP
```

`status.order` — какой статус «побеждает» при агрегации: если хоть один компонент даёт `NO_RESTAURANTS`, вся группа помечена так же.

#### 6.18.4. Кастомный `HealthIndicator`

```java
// RestaurantHealthCheck.java
@Component
public class RestaurantHealthCheck implements HealthIndicator {
    private final RestaurantRepository restaurantRepository;
    public RestaurantHealthCheck(RestaurantRepository r) { this.restaurantRepository = r; }

    @Override
    public Health health() {
        Long count = restaurantRepository.getRestaurantCount();
        return count > 0
             ? Health.up().withDetail("restaurantCount", count).build()
             : Health.status("NO_RESTAURANTS").build();
    }
}
```

Имя бина (`restaurantHealthCheck`) автоматически становится ключом в `/health`. Свои статусы (`NO_RESTAURANTS`) допустимы, но нужно указать их место в `status.order`.

#### 6.18.5. Кастомный `@Endpoint`

```java
// RestaurantCustomEndpoint.java
@Component
@Endpoint(id = "restaurant")
public class RestaurantCustomEndpoint {
    Map<String, String> map = new HashMap<>();
    ...
    @ReadOperation  public Map<String, String> readOperation() { return map; }
    @WriteOperation public Map<String, String> writeOperation(String key, String value) { ... }
    @DeleteOperation public Map<String, String> deleteOperation() { map.clear(); return map; }
}
```

Маппинг operation → HTTP:
- `@ReadOperation` → GET.
- `@WriteOperation` → POST.
- `@DeleteOperation` → DELETE.

Параметр операции с `@Selector` — путь: `/actuator/restaurant/{selector}`.

Есть также специализированные: `@WebEndpoint` (только HTTP), `@JmxEndpoint` (только JMX), `@ServletEndpoint` (полный контроль над Servlet).

#### 6.18.6. Метрики (Micrometer)

Micrometer — «SLF4J для метрик»: единый API, back-end подставляется зависимостью (Prometheus, Datadog, JMX, CloudWatch, StatsD, ...).

Типы метров:
- **Counter** — монотонно растущий счётчик (`http.server.requests`).
- **Gauge** — мгновенное значение (`jvm.memory.used`).
- **Timer** — распределение продолжительности (avg, max, percentiles).
- **DistributionSummary** — распределение произвольных значений (размер payload'а).
- **LongTaskTimer** — активные долгие задачи.

Пример:
```java
@Autowired MeterRegistry registry;

Counter c = registry.counter("orders.created", "type", "urgent");
c.increment();

Timer t = registry.timer("db.query", "table", "T_ACCOUNT");
t.record(() -> jdbc.query(...));
```

`@Timed` (aspect) — декларативно.

#### 6.18.7. Actuator и Security

Все endpoints, кроме `/health` и `/info`, могут содержать чувствительную инфу (env, beans, heapdump). В прод-конфиге:
- отделяют management port (`management.server.port=9091`) — доступ только из внутренней сети;
- закрывают через Spring Security (`.requestMatchers("/actuator/**").hasRole("ADMIN")`).

**Грабли.**
- `management.endpoints.web.exposure.include=*` в проде без security → утечка секретов через `/actuator/env`.
- `heapdump` может занять гигабайты — не оставлять открытым.
- Кастомные `HealthIndicator`'ы не должны делать долгих запросов — health-endpoint дёргается часто (k8s каждые 10 сек).

---

## Что дальше

Если после прогона по этой теории остались нечёткие места — переходите к конкретному модулю в разделе [3. Разбор каждого задания](#3-разбор-каждого-задания) и сравнивайте `<модуль>` (задание) с `<модуль>-solution` (эталон). Раздел [4. Сквозные темы](#4-сквозные-темы) сшивает AOP, транзакции и auto-configuration в одну ментальную модель — «прокси + BeanPostProcessor + condition».
