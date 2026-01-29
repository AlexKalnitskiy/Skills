---
name: integration-testing-writing
description: Пишет интеграционные тесты для .NET приложений с использованием Testcontainers, WireMock и FluentAssertions. Используется когда разработчик просит написать интеграционные тесты. Настраивает тестконтейнеры для Kafka, Redis, Cassandra, S3 (MinIO), SQL Database, мокирует HTTP-сервисы через WireMock, создает WebApplicationFactory для полных интеграционных тестов.
---

# Написание интеграционных тестов

## Процесс работы

### Шаг 1: Разработка сценария теста

Перед началом написания кода:
1. Найди код, который нужно протестировать.
2. Определи, что именно будет тестироваться. Это как правильно Rest Controller либо BackgroundTask, либо KafkaConsumerHost
3. Разработай план теста с описанием:
   - Что именно в приложении тестируется
   - Общий сценарий теста по пунктам
   - Как ты будешь отправлять входные данные. Либо делать Http-запросы к API, либо продюсить в топик кафки для консюма, либо еще что-то.
   - Какие результаты ожидаются и как ты будешь их проверять
   - Какие именно зависимости из конфига appsettings ты будешь заменять тестконтейнерами ил wiremock сервером.
4. **Предложи план пользователю и дождись подтверждения**

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

#### Критически важное правило: только публичный API

**ЗАПРЕЩЕНО** использовать `IServiceProvider`!!!

```csharp
// ❌ НЕПРАВИЛЬНО - прямое использование сервисов
var processor = _app.Services.GetRequiredService<ICustomerDataProcessor>();
await processor.ProcessAsync(...);
```

**Весь тест - это проверка публичного API приложения.** Приложение работает как черный ящик.

Если нужно проверить работу консюмера Kafka:
- ✅ **ПРАВИЛЬНО**: Продюсь сообщения в нужный топик Kafka, дождись обработки, проверь результаты в выходных топиках или базах данных
- ❌ **НЕПРАВИЛЬНО**: Получить консюмер через `GetRequiredService<IKafkaConsumer>()` и вызвать его напрямую

Пример правильного подхода:
```csharp
//Arrange
var inputMessages = GenerateTestMessages();

    //Act - работаем только через публичный API (Kafka топики)
    await _kafkaContainer.ProduceMessagesToTopicAsync("input-topic", inputMessages);

    // Ждем, пока все сообщения будут обработаны
    await _app.Services.WaitWhileTopicConsumedAsync(
        KafkaCluster.MailingsRuntime,
        new TopicName("input-topic"),
        ConsumerGroups.SomeGroup);

    //Assert - проверяем результаты через публичный API (выходные топики, БД)
    var outputMessages = _kafkaContainer.ConsumeMessages("output-topic", "test-consumer");
    Assertions.AssertMessagesAreSentCorrectly(outputMessages, expectedMessages, messagesCount);
```

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
