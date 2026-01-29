# Примеры работы с WebApplicationFactory

## WebApplicationBuilder - базовый билдер

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

namespace TestingHelpers.WebApplication;

public static class WebApplicationBuilder
{
    public static Action<IWebHostBuilder> Create() => b => { };

    extension(Action<IWebHostBuilder> apply)
    {
        public Action<IWebHostBuilder> WithSetting(string key, string? value) => builder =>
        {
            apply(builder);
            builder.UseSetting(key, value);
        };

        public Action<IWebHostBuilder> WithAppConfiguration(IEnumerable<KeyValuePair<string, string?>> initialData) => builder =>
        {
            apply(builder);
            builder.ConfigureAppConfiguration((context, configBuilder) =>
            {
                configBuilder.AddInMemoryCollection(initialData);
            });
        };

        public Action<IWebHostBuilder> WithTestServices(Action<IServiceCollection> servicesConfiguration) => builder =>
        {
            apply(builder);
            builder.ConfigureTestServices(servicesConfiguration);
            // Автоматически мокаем IAuthJwtClient
            builder.ConfigureTestServices(services =>
            {
                var authJwtClientMock = new Mock<IAuthJwtClient>();
                authJwtClientMock.Setup(
                    m => m.GetCustomTokenAsync(
                        It.IsAny<GetCustomTokenRequest>(),
                        It.IsAny<CancellationToken>()))
                    .ReturnsAsync(new GetCustomTokenResponse
                    {
                        Token = "token"
                    });

                var existingAuthJwtClient = services.SingleOrDefault(
                    d => d.ServiceType == typeof(IAuthJwtClient));
                if (existingAuthJwtClient is not null)
                    services.Remove(existingAuthJwtClient);

                services.AddSingleton<IAuthJwtClient>(_ => authJwtClientMock.Object);
            });
        };

        public Action<IWebHostBuilder> WithLogging(Action<ILoggingBuilder> loggingConfiguration) => builder =>
        {
            apply(builder);
            builder.ConfigureLogging(loggingConfiguration);
        };

        public WebApplicationFactory<T> Build<T>() where T : class
        {
            var factory = new WebApplicationFactory<T>().WithWebHostBuilder(apply);
            _ = factory.Server; // Инициализируем сервер
            return factory;
        }
    }
}
```

## WebApplicationBuilderExtensions - роли приложения

```csharp
using Microsoft.AspNetCore.Hosting;
using Mindbox.MailingsRuntime.Infrastructure;
using TestingHelpers.WebApplication;

namespace Worker.Tests.WebApplication;

public static class WebApplicationBuilderExtensions
{
    extension(Action<IWebHostBuilder> apply)
    {
        public Action<IWebHostBuilder> AsMailingsPipelineRole(string role) =>
            apply.WithSetting("HOST_CONFIGURATION:ROLE", role);

        public Action<IWebHostBuilder> AsManualMailingsRole() =>
            apply.WithSetting("HOST_CONFIGURATION:ROLE", ServiceRoles.ManualMailings);

        public Action<IWebHostBuilder> AsWorkControlRole() =>
            apply.WithSetting("HOST_CONFIGURATION:ROLE", ServiceRoles.WorkControlLeader);

        public Action<IWebHostBuilder> AsFrequencyPolicyRole() =>
            apply.WithSetting("HOST_CONFIGURATION:ROLE", ServiceRoles.FrequencyPolicy);
    }
}
```

## WebApplicationBuilderExtensions - дополнительные сервисы

```csharp
using Microsoft.Extensions.DependencyInjection;
using Moq;

namespace TestingHelpers.WebApplication;

public static class WebApplicationBuilderExtensions
{
    public static IServiceCollection WithSlaMailingsMockClient(
        this IServiceCollection collection,
        bool slaSupportedForAll = true,
        Action<Mock<ISlaMailingsClient>>? configure = null)
    {
        var slaMailingsClientMock = new Mock<ISlaMailingsClient>();
        slaMailingsClientMock
            .Setup(s => s.IsSlaSupportedAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(slaSupportedForAll);
        configure?.Invoke(slaMailingsClientMock);
        collection.AddTransient<ISlaMailingsClient>(sp => slaMailingsClientMock.Object);
        return collection;
    }

    public static IServiceCollection WithKafkaLimitCheckerMock(this IServiceCollection collection)
    {
        collection.AddTransient<IKafkaLimitsCheckerFactory>(sp =>
        {
            var mock = new Mock<IKafkaLimitsCheckerFactory>();
            mock.Setup(m => m.Create(It.IsAny<IKafkaSecuritySettings>())).Returns(Mock.Of<IKafkaLimitsChecker>());
            return mock.Object;
        });
        return collection;
    }

    public static IServiceCollection With(this IServiceCollection collection, Action<IServiceCollection>? setup)
    {
        setup?.Invoke(collection);
        return collection;
    }
}
```

## TestApp - полный пример

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Mindbox.MailingsRuntime.Infrastructure.Kafka;
using TestingHelpers;
using TestingHelpers.Containers;
using TestingHelpers.WebApplication;
using Testcontainers.Kafka;
using Testcontainers.Redis;
using WireMock.Server;

namespace Services.Tests;

[TestClass]
public static class TestApp
{
    public static KafkaContainer KafkaContainer { get; } = KafkaContainerExtensions.CreateKafka();
    public static RedisContainer RedisContainer { get; } = RedisContainerExtensions.CreateRedis();
    public static WireMockServer ExternalServiceMock { get; } = WireMockServer.Start();

    [AssemblyInitialize]
    public static async Task InitializeAsync(TestContext context)
    {
        ConfigBuilder.WriteSecretsFiles();
        Environment.SetEnvironmentVariable("ASPNETCORE_ENVIRONMENT", "development");
        await KafkaContainer.StartAsync();
        await RedisContainer.StartAsync();
        
        // Настройка WireMock
        ExternalServiceMock.WithSomeEndpoint(...);
    }

    [AssemblyCleanup]
    public static async Task CleanupAsync()
    {
        await KafkaContainer.DisposeAsync();
        await RedisContainer.DisposeAsync();
        ExternalServiceMock.Dispose();
    }

    public static IEnumerable<KeyValuePair<string, string?>> GetConfigOverridenByTestContainers()
    {
        var configOverrides = new Dictionary<string, string?>();

        // Переопределение Kafka для всех кластеров
        foreach (var kafka in Enum.GetNames<KafkaCluster>())
            configOverrides.Add($"KafkaBootstrapServers:{kafka}:bootstrap.servers", KafkaContainer.GetBootstrapServers());

        // Переопределение Redis
        configOverrides.Add("RedisOptions:EndPoints:0:Address", RedisContainer.GetRedisHost());
        configOverrides.Add("RedisOptions:EndPoints:0:Port", RedisContainer.GetRedisPort().ToString(CultureInfo.InvariantCulture));

        // Переопределение внешних сервисов
        configOverrides.Add("ExternalService:Url", ExternalServiceMock.Url);

        return configOverrides;
    }

    public static WebApplicationFactory<Program> RunServicesApp(
        Action<IServiceCollection>? setup = null) =>
        WebApplicationBuilder
            .Create()
            .WithAppConfiguration(GetConfigOverridenByTestContainers())
            .WithTestServices(services => services
                .With(s =>
                {
                    s.AddSingleton<IAuthorizationService, AlwaysAllowAuthorizationService>();
                })
                .With(setup))
            .WithLogging(l => l.SetMinimumLevel(LogLevel.Warning))
            .Build<Program>();
}
```

## TestApp с ролями (Worker.Tests)

```csharp
[TestClass]
public static class TestApp
{
    public static KafkaContainer KafkaContainer { get; } = KafkaContainerExtensions.CreateKafka();
    public static RedisContainer RedisContainer { get; } = RedisContainerExtensions.CreateRedis();
    public static WireMockServer MailingsMicroservice { get; } = WireMockServer.Start();

    public static WebApplicationFactory<Program> RunWorkControlApp(
        string? tenant = null,
        Action<IServiceCollection>? setup = null) =>
        WebApplicationBuilder
            .Create()
            .AsWorkControlRole()
            .WithAppConfiguration(GetConfigOverridenByTestContainers())
            .WithTestServices(services => services
                .With(setup))
            .WithLogging(l => l.SetMinimumLevel(LogLevel.Error))
            .Build<Program>();

    public static WebApplicationFactory<Program> RunManualMailingsApp(
        string? tenant = null,
        Action<IServiceCollection>? setup = null) =>
        WebApplicationBuilder
            .Create()
            .AsManualMailingsRole()
            .WithAppConfiguration(GetConfigOverridenByTestContainers())
            .WithTestServices(services => services
                .WithSlaMailingsMockClient()
                .With(setup))
            .WithLogging(l => l.SetMinimumLevel(LogLevel.Error))
            .Build<Program>();

    public static WebApplicationFactory<Program> RunMailingsPipelineApp(
        string? tenant = null,
        string? role = null,
        Action<IServiceCollection>? setup = null) =>
        WebApplicationBuilder
            .Create()
            .AsMailingsPipelineRole(role ?? ServiceRoles.MailingsPipelinePromotional)
            .WithAppConfiguration(GetConfigOverridenByTestContainers())
            .WithTestServices(services => services
                .WithSlaMailingsMockClient()
                .WithKafkaLimitCheckerMock()
                .WithFeatures(features =>
                {
                    if (tenant != null)
                    {
                        features.EnableTenantFeature(f => f.Email, tenant);
                        features.EnableTenantFeature(f => f.MobilePush, tenant);
                        features.EnableTenantFeature(f => f.Sms, tenant);
                        features.EnableTenantFeature(f => f.Viber, tenant);
                        features.EnableTenantFeature(f => f.WebPush, tenant);
                        features.EnableTenantFeature(f => f.NotificationsAndMessengers, tenant);
                    }
                })
                .With(setup))
            .WithLogging(l => l.SetMinimumLevel(LogLevel.Error))
            .Build<Program>();
}
```