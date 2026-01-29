---
name: integration-testing-writing
description: Пишет интеграционные тесты для .NET приложений с использованием Testcontainers, WireMock и FluentAssertions. ОБЯЗАТЕЛЬНО использовать этот скилл когда пользователь просит написать интеграционный тест, integration test, интеграционное тестирование, тест с контейнерами, тест с Kafka/Redis/Cassandra, тест с WireMock, WebApplicationFactory тест. Настраивает тестконтейнеры для Kafka, Redis, Cassandra, S3 (MinIO), SQL Database, мокирует HTTP-сервисы через WireMock, создает WebApplicationFactory для полных интеграционных тестов.
---

# Написание интеграционных тестов

## ⚠️ КРИТИЧЕСКИ ВАЖНЫЕ ПРАВИЛА

### Правило 1: ОБЯЗАТЕЛЬНОЕ составление плана ПЕРЕД написанием кода

**ПЕРЕД ЛЮБЫМ НАПИСАНИЕМ КОДА ТЫ ДОЛЖЕН:**

1. **Создать todo list** используя инструмент `todo_write` с задачами для:
   - Анализа кода, который нужно протестировать
   - Определения типа теста (REST Controller / BackgroundTask / KafkaConsumerHost)
   - Составления плана теста
   - Реализации инфраструктуры
   - Написания теста

2. **Составить детальный план теста** и представить его пользователю в виде структурированного списка:
   - Что именно в приложении тестируется (конкретный класс/метод/эндпоинт)
   - Общий сценарий теста по пунктам (Arrange → Act → Assert)
   - Как ты будешь отправлять входные данные (HTTP-запросы к API / продюсинг в Kafka топик / другой способ)
   - Какие результаты ожидаются и как ты будешь их проверять (выходные топики / БД / HTTP ответы)
   - Какие именно зависимости из конфига appsettings ты будешь заменять тестконтейнерами или WireMock сервером

3. **Дождаться явного подтверждения плана от пользователя**

**ЗАПРЕЩЕНО начинать писать код до подтверждения плана!**

### Правило 2: ЗАПРЕТ на использование IServiceProvider

**СТРОГО ЗАПРЕЩЕНО** использовать `IServiceProvider`, `GetRequiredService`, `GetService` или любые другие способы получения сервисов из DI контейнера!

```csharp
// ❌❌❌ АБСОЛЮТНО НЕПРАВИЛЬНО - ЗАПРЕЩЕНО!
var processor = _app.Services.GetRequiredService<ICustomerDataProcessor>();
await processor.ProcessAsync(...);

var consumer = _app.Services.GetRequiredService<IKafkaConsumer>();
await consumer.ConsumeAsync(...);

var repository = _app.Services.GetRequiredService<IRepository>();
var data = await repository.GetAsync(...);
```

**Весь тест - это проверка публичного API приложения. Приложение работает как черный ящик.**

✅ **ПРАВИЛЬНЫЙ подход:**
- Для REST API: делай HTTP-запросы через `HttpClient` из `WebApplicationFactory`
- Для Kafka консюмеров: продюсь сообщения в входной топик, дождись обработки, проверь результаты в выходных топиках или БД
- Для BackgroundTask: запусти задачу через публичный API (если есть) или через Kafka/другой входной канал

**Если ты видишь в коде использование `_app.Services` или `GetRequiredService` - это ОШИБКА! Исправь немедленно!**

## Процесс работы

### Шаг 1: Разработка сценария теста

**ВАЖНО: Этот шаг ОБЯЗАТЕЛЕН и должен выполняться ПЕРВЫМ!**

Перед началом написания кода:
1. Создай todo list с задачами для всего процесса
2. Найди код, который нужно протестировать
3. Определи, что именно будет тестироваться (REST Controller / BackgroundTask / KafkaConsumerHost)
4. Разработай план теста с описанием:
   - Что именно в приложении тестируется
   - Общий сценарий теста по пунктам
   - Как ты будешь отправлять входные данные (HTTP-запросы / Kafka топики / другой способ)
   - Какие результаты ожидаются и как ты будешь их проверять
   - Какие именно зависимости из конфига appsettings ты будешь заменять тестконтейнерами или WireMock сервером
5. **Представь план пользователю в структурированном виде и дождись подтверждения**

**Только после подтверждения плана приступай к реализации.**

### Шаг 2: Подготовка к тесту

## Тестовая инфраструктура

### Поиск существующей инфраструктуры

Перед созданием новой инфраструктуры:
1. Изучи тестовые проекты в репозитории
2. Ищи существующие тестовые классы или методы по работе с `WepApplicationFactory` или `Testcontainers`.
3. **Переиспользуй существующую инфраструктуру**, если она подходит
4. Создай необходимое или добавь новые методы, если считаешь нужным

### Основные компоненты инфраструктуры

1. **WebApplicationBuilder**
   - Билдер для создания `WebApplicationFactory`
   - Методы: `WithAppConfiguration`, `WithTestServices`, `WithLogging`

2. **ConfigBuilderExtensions** (`src/Infra/ConfigBuilderExtensions.cs`)
   - Экстеншены для настройки конфигурации
   - Метод `AddJsonsConfigsAndSecrets` загружает appsettings и secrets

3. **Пакеты NuGet**:
   - `Testcontainers.*` - для контейнеров (Kafka, Redis, Cassandra, PostgreSQL, MinIO)
   - `WireMock.Net` - для мокирования HTTP-сервисов
   - `FluentAssertions` - для ассертов
   - `Microsoft.AspNetCore.Mvc.Testing` - для WebApplicationFactory

### Настройка тестконтейнеров

Для каждого внешнего сервиса в конфигурации приложения создавай отдельный статический класс с экстеншенами в папке `Infrastructure` тестового проекта (например, `src/Tests/Infrastructure/`). Добавляй туда методы для работы с контейнером (создание топиков, очистка данных и т.д.).

**Пример структуры файлов:**
- `src/Tests/Infrastructure/KafkaContainerExtensions.cs`
- `src/Tests/Infrastructure/RedisContainerExtensions.cs`
- `src/Tests/Infrastructure/CassandraContainerExtensions.cs`

Пример для Kafka (`src/Tests/Infrastructure/KafkaContainerExtensions.cs`):
```csharp
public static class KafkaContainerExtensions
{
    public static KafkaContainer CreateKafka()
    {
        var container = new KafkaBuilder("confluentinc/cp-kafka:7.4.0")
            .Build();
        return container;
    }

    // Методы: GetBootstrapServers(), CreateTopicsAsync(), ProduceMessagesToTopicAsync(), ConsumeMessages()
}
```

**Полные примеры для всех контейнеров** (Kafka, Redis, Cassandra + Flyway, MinIO/S3, PostgreSQL) смотри в `EXAMPLES_CONTAINERS.md`

### Настройка WireMock для HTTP-клиентов

Для мокирования внешних HTTP-сервисов создавай отдельные экстеншены для настройки WireMock в папке `Infrastructure` тестового проекта (например, `src/Tests/Infrastructure/`):

```csharp
public static WireMockServer ExternalServiceMock { get; } = WireMockServer.Start();

public static WireMockServer WithSomeEndpoint(this WireMockServer server, SomeDto data)
{
    var request = Request.Create()
        .WithPath("/api/endpoint")
        .UsingPost();
    
    var response = Response.Create()
        .WithHeader("Content-Type", "application/json")
        .WithBody(JsonConvert.SerializeObject(data));
    
    server.Given(request).RespondWith(response);
    return server;
}
```

**Полные примеры** (простые HTTP endpoint моки, GraphQL запросы, настройка нескольких WireMock серверов) смотри в `EXAMPLES_WIREMOCK.md`

### Настройка .csproj файла

В тестовом проекте долджны быть скопированы appsettings:

```xml
<ItemGroup>
  <Content Include="..\..\config\appsettings.*.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

Это обеспечит копирование `appsettings.common.json` и `appsettings.Development.json`.

### Создание файла с секретами

Найди или создай тест с `AssemblyInitialize` и вызывай там метод для создания файла секретов (если его еще нет):

```csharp
[TestClass]
public sealed class Initialization
{
    [AssemblyInitialize]
    public static void InitializeAssembly(TestContext context)
    {
        Environment.SetEnvironmentVariable("ASPNETCORE_ENVIRONMENT", "development");
        ConfigBuilder.WriteSecretsFiles();
    }
}
```

Это создаст необходимые файлы `secrets/secrets.json` и `auth-secrets/auth.json` для запуска приложения.
Проверь, какие ключи нужны и добавь нужные. Удали неиспользуемые. Проверь для этого файл `.secrets`.

**Полный пример `WriteSecretsFiles`** смотри в `EXAMPLES_CONFIG.md`

### Метод GetConfigOverridenByTestContainers

Создавай статический метод, который переопределяет конфигурацию адресами тестконтейнеров. Переопредели нужные зависимости:

```csharp
public static IEnumerable<KeyValuePair<string, string?>> GetConfigOverridenByTestContainers()
{
    var configOverrides = new Dictionary<string, string?>();
    // Контейнеры
    configOverrides.Add("RedisOptions:EndPoints:0:Address", RedisContainer.GetRedisHost());
    configOverrides.Add("RedisOptions:EndPoints:0:Port", 
        RedisContainer.GetRedisPort().ToString(CultureInfo.InvariantCulture));
    
    // WireMock сервисы
    configOverrides.Add("ExternalService:Url", ExternalServiceMock.Url);
    
    return configOverrides;
}
```

**Полные примеры** с разными типами контейнеров смотри в `EXAMPLES_CONFIG.md`

### Создание WebApplicationFactory

Используй `WebApplicationBuilder` для создания `WebApplicationFactory`. Базовый пример:

```csharp
public static WebApplicationFactory<Program> RunApp() =>
    WebApplicationBuilder
        .Create()
        .WithAppConfiguration(GetConfigOverridenByTestContainers())
        .WithTestServices(services => services
            .With(setup))
        .WithLogging(l => l.SetMinimumLevel(LogLevel.Warning))
        .Build<Program>();
```

**Подробные примеры**: См. `EXAMPLES_WEBAPPLICATION.md`

## Структура тестового класса

### Инициализация контейнеров

**Важно**: Скоуп всех контейнеров должен быть **класс с тестами**, не метод. Используй `[ClassInitialize]`:

```csharp
[TestClass]
public class MyIntegrationTests
{
    private static WebApplicationFactory<Program> _app = null!;
    private static KafkaContainer _kafkaContainer = null!;
    private static RedisContainer _redisContainer = null!;
    private static WireMockServer _externalServiceMock = null!;
    
    [ClassInitialize]
    public static async Task Initialize(TestContext testContext)
    {   
        _kafkaContainer = KafkaContainerExtensions.CreateKafka();
        await _kafkaContainer.StartAsync();
        
        _redisContainer = RedisContainerExtensions.CreateRedis();
        await _redisContainer.StartAsync();
        
        _externalServiceMock = WireMockServer.Start();
        _externalServiceMock.WithSomeEndpoint(...);
        
        await _kafkaContainer.CreateTopicsAsync([...]);
        _app = TestApp.RunApp();
    }
    
    [ClassCleanup]
    public static async Task CleanupAsync()
    {
        await _app.DisposeAsync();
        await _kafkaContainer.DisposeAsync();
        await _redisContainer.DisposeAsync();
        _externalServiceMock.Dispose();
    }
}
```

**Полные примеры** инициализации с разными контейнерами смотри в `EXAMPLES_WEBAPPLICATION.md`

### Написание тестов

#### ⚠️ КРИТИЧЕСКИ ВАЖНО: Только публичный API - ЗАПРЕТ на IServiceProvider

**ПОВТОРЯЕМ: СТРОГО ЗАПРЕЩЕНО использовать `IServiceProvider`, `GetRequiredService`, `GetService`!**

```csharp
// ❌❌❌ АБСОЛЮТНО НЕПРАВИЛЬНО - ЗАПРЕЩЕНО!
var processor = _app.Services.GetRequiredService<ICustomerDataProcessor>();
await processor.ProcessAsync(...);

var consumer = _app.Services.GetRequiredService<IKafkaConsumer>();
await consumer.ConsumeAsync(...);

var repository = _app.Services.GetRequiredService<IRepository>();
var data = await repository.GetAsync(...);

// Даже для вспомогательных методов - НЕПРАВИЛЬНО!
var helper = _app.Services.GetRequiredService<ITestHelper>();
```

**Весь тест - это проверка публичного API приложения. Приложение работает как черный ящик.**

**Правильные подходы:**

1. **Для REST API:**
```csharp
// ✅ ПРАВИЛЬНО - через HttpClient
var client = _app.CreateClient();
var response = await client.PostAsJsonAsync("/api/endpoint", requestData);
response.StatusCode.Should().Be(HttpStatusCode.OK);
```

2. **Для Kafka консюмеров:**
```csharp
// ✅ ПРАВИЛЬНО - через публичный API (Kafka топики)
//Arrange
var inputMessages = GenerateTestMessages();

//Act - работаем только через публичный API (Kafka топики)
await _kafkaContainer.ProduceMessagesToTopicAsync("input-topic", inputMessages);

// Ждем, пока все сообщения будут обработаны (это вспомогательный метод, не получение сервиса!)
await WaitForTopicConsumptionAsync("input-topic", ConsumerGroups.SomeGroup);

//Assert - проверяем результаты через публичный API (выходные топики, БД)
var outputMessages = _kafkaContainer.ConsumeMessages("output-topic", "test-consumer");
Assertions.AssertMessagesAreSentCorrectly(outputMessages, expectedMessages, messagesCount);
```

3. **Для BackgroundTask:**
```csharp
// ✅ ПРАВИЛЬНО - запуск через публичный API или входной канал
// Если есть HTTP эндпоинт для запуска задачи:
var client = _app.CreateClient();
await client.PostAsync("/api/tasks/start", null);

// Или через Kafka топик, который триггерит задачу:
await _kafkaContainer.ProduceMessagesToTopicAsync("task-trigger-topic", triggerMessage);
```

**Если в коде теста есть `_app.Services.GetRequiredService` или `_app.Services.GetService` - это ОШИБКА! Исправь немедленно!**

#### Структура теста

1. **Декларативный, функциональный, чистый код**
2. **Используй FluentAssertions** для проверок
3. **Комментарии только для секций Arrange, Act, Assert** с кратким объяснением
4. **Работа только через публичный API** - никаких прямых вызовов сервисов
5. **Критически важно**: Используй HashSet для проверки сообщений из Kafka (см. `EXAMPLES_ASSERTIONS.md`)

#### Параметризация тестов

1. **Старайся писать один тест** и параметризовать его через `[DataRow]`
2. **Создавай структуры с тестовыми данными**, если нужно:

```csharp
public static class TestData
{
    public static TestScenario Scenario1 => new()
    {
        Name = "Scenario1",
        InputGenerator = i => new InputDto { Id = i },
        ExpectedGenerator = (input, dateTime) => new ExpectedDto { ... }
    };
}

public record TestScenario
{
    public required string Name { get; init; }
    public required Func<int, InputDto> InputGenerator { get; init; }
    public required Func<InputDto, DateTime, ExpectedDto> ExpectedGenerator { get; init; }
}
```

#### Константы и магические числа

1. **Избегай констант и магических чисел** прямо в тесте
2. **Проверь, есть ли файл с тестовыми константами**
3. **Если нет - создай его**:

```csharp
public static class TestConstants
{
    public const string TenantSystemName = "TestTenant";
    public const int DefaultMessagesCount = 100;
    public const int DefaultTimeoutSeconds = 60;
}
```

#### Timeout атрибут

**Всегда добавляй `[Timeout]` атрибут** тесту:
```csharp
[TestMethod]
[Timeout(60000)] // 60 секунд обычно достаточно
public async Task MyTest() { ... }
```

## Проверка и отладка

После написания теста:

1. **Убедись, что код компилируется**
2. **С разрешения пользователя запусти тест**
3. **Отлаживай, пока тест не пройдет**

Если тест падает:
- Проверь вывод теста
- Убедись, что все топики созданы
- Проверь, что WireMock настроен правильно
- Проверь, что конфигурация переопределена корректно
- Убедись, что секреты созданы

## Примеры

Все примеры кода находятся в отдельных файлах для удобства. Используй их как справочник при написании тестов:

- **`EXAMPLES_CONTAINERS.md`** - примеры работы с контейнерами:
  - Kafka (создание, топики, продюсинг, консюминг)
  - Redis (создание, очистка)
  - Cassandra + Flyway (с Network для связанных контейнеров)
  - MinIO/S3
  - PostgreSQL

- **`EXAMPLES_WIREMOCK.md`** - примеры настройки WireMock серверов:
  - Простые HTTP endpoint моки
  - GraphQL запросы
  - Настройка нескольких WireMock серверов
  - Использование в TestApp

- **`EXAMPLES_WEBAPPLICATION.md`** - примеры создания WebApplicationFactory:
  - WebApplicationBuilder
  - Роли приложения (AsWorkControlRole, AsMailingsPipelineRole, AsManualMailingsRole)
  - TestApp классы
  - WaitWhileTopicConsumedAsync

- **`EXAMPLES_ASSERTIONS.md`** - примеры проверок:
  - AssertMessagesAreSentCorrectly
  - AssertEventLogsMessages
  - AssertFrequencyPolicyAttemptsRegisteredAsync
  - Использование FluentAssertions

- **`EXAMPLES_TEST_DATA.md`** - примеры структуры тестовых данных:
  - TestData с генераторами
  - Параметризация через DataRow
  - MailingTestPreset
  - Константы для тестов

- **`EXAMPLES_CONFIG.md`** - примеры конфигурации:
  - WriteSecretsFiles
  - GetConfigOverridenByTestContainers
  - Настройка .csproj файлов
  - Initialization классы
