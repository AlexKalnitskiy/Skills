# Примеры конфигурации

## WriteSecretsFiles - создание файлов секретов

```csharp
using System.Security.Cryptography;
using System.Text.Json;
using Mindbox.Authentication.Abstractions;
using Mindbox.Authentication.AspNetCore;

namespace TestingHelpers;

public static class ConfigBuilder
{
    private static readonly Lazy<(RSA rsa, string publicKey)> _testKeyPair = new(GenerateKeyPair);
    private static readonly Lazy<(RSA rsa, string publicKey)> _testForDeleteKeyPair = new(GenerateKeyPair);

    public static RSA TestRsa => _testKeyPair.Value.rsa;
    public static string TestPublicKey => _testKeyPair.Value.publicKey;
    public static RSA TestForDeleteRsa => _testForDeleteKeyPair.Value.rsa;
    public static string TestForDeletePublicKey => _testForDeleteKeyPair.Value.publicKey;

    private static (RSA rsa, string publicKey) GenerateKeyPair()
    {
        var rsa = RSA.Create(2048);
        var publicKey = rsa.ExportRSAPublicKeyPem();
        return (rsa, publicKey);
    }

    public static void WriteSecretsFiles()
    {
        var testSecretsPath = Path.Combine(AppContext.BaseDirectory, "secrets");
        var testAuthSecretsPath = Path.Combine(AppContext.BaseDirectory, "auth-secrets");

        Directory.CreateDirectory(testSecretsPath);
        Directory.CreateDirectory(testAuthSecretsPath);

        var testSecretsFile = Path.Combine(testSecretsPath, "secrets.json");
        var testAuthSecretsFile = Path.Combine(testAuthSecretsPath, "auth.json");

        var data = new Dictionary<string, object>
        {
            ["Sentry"] = new Dictionary<string, string>
            {
                ["DSN"] = "https://cf68c4cc843681d@autobugs.mindbox.ru/2224"
            },
            ["WorkControl:RedisOptions"] = new Dictionary<string, string>
            {
                ["password"] = ""
            },
            ["ImgProxy"] = new Dictionary<string, string>
            {
                ["Key"] = "44bf2523efc2a6ed1293c5d1e81c89d127fc148e8b32350b2cb612c87d0fd64a",
                ["Salt"] = "3a4d8c65ac7e163b3e59b7b7845679b250770fb7f485c6162321eac9dc675ad3",
            },
            ["anonymizer"] = new Dictionary<string, string>
            {
                ["test-tenant"] = TestPublicKey,
                ["test-tenant.forDelete"] = TestForDeletePublicKey
            },
        };

        var authData = new Dictionary<string, object>
        {
            ["jwks:primary"] = new AuthJwtJsonWebKeyModel
            {
                Kid = "kid",
                Kty = "RSA",
                N = "n",
                E = "AQAB",
                Use = "sig"
            },
            ["tokens:nexus"] = new AuthJwtConfigToken
            {
                Token = "nexus-token"
            },
            ["tokens:universal-channel-runtime"] = new AuthJwtConfigToken
            {
                Token = "universal-channel-runtime-token"
            }
        };

        File.WriteAllText(
            testSecretsFile,
            JsonSerializer.Serialize(data));

        File.WriteAllText(
            testAuthSecretsFile,
            JsonSerializer.Serialize(authData));
    }
}
```

**Важно**: Проверь файл `.secrets` в проекте, чтобы понять, какие ключи нужны. Добавь нужные ключи и удали неиспользуемые.

## Initialization класс с AssemblyInitialize

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;

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

## Настройка .csproj для копирования appsettings

```xml
<ItemGroup>
  <Content Include="..\..\config\appsettings.*.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

Это обеспечит копирование `appsettings.common.json` и `appsettings.Development.json`.

## Настройка .csproj для копирования миграций (CQL)

```xml
<ItemGroup>
  <Content Include="..\..\cql\**\*">
    <Link>cql\%(RecursiveDir)%(Filename)%(Extension)</Link>
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

Это скопирует все файлы из директории `cql` в выходную директорию тестов.

## GetConfigOverridenByTestContainers - переопределение конфигурации

```csharp
public static IEnumerable<KeyValuePair<string, string?>> GetConfigOverridenByTestContainers()
{
    var configOverrides = new Dictionary<string, string?>
    {
        // Redis
        ["WorkControl:RedisOptions:EndPoints:0:Address"] = RedisContainer.GetRedisHost(),
        ["WorkControl:RedisOptions:EndPoints:0:Port"] = 
            RedisContainer.GetRedisPort().ToString(CultureInfo.InvariantCulture),
        
        // Kafka для всех кластеров
        // ...
    };

    // Переопределение Kafka для всех кластеров
    foreach (var kafka in Enum.GetNames<KafkaCluster>())
        configOverrides.Add($"KafkaBootstrapServers:{kafka}:bootstrap.servers", KafkaContainer.GetBootstrapServers());

    // Cassandra
    foreach (var cassandra in new[] { "Mailings" })
    {
        configOverrides.Add($"Cassandra:{cassandra}:NodeHosts:0", CassandraContainer.Hostname);
        for (var i = 1; i <= 10; i++)
            configOverrides.Remove($"Cassandra:{cassandra}:NodeHosts:{i}");
        configOverrides.Add($"Cassandra:{cassandra}:Datacenter", "dc1");
        configOverrides.Add($"Cassandra:{cassandra}:ConsistencyLevel", "One");
        configOverrides.Add(
            "Cassandra:Mailings:Port",
            CassandraContainer.GetMappedPublicPort(9042).ToString(CultureInfo.InvariantCulture));
    }

    // WireMock сервисы
    configOverrides.Add("Mailings:ServicesUrl", MailingsMicroservice.Url);
    configOverrides.Add("Nexus:ServicesUrl", TenantsMicroservice.Url);
    configOverrides.Add($"{WebPushGateOptions.SectionName}:{nameof(WebPushGateOptions.Host)}", WebPushGate.Url);

    return configOverrides;
}
```

## Использование ConfigBuilderExtensions в приложении

```csharp
using Microsoft.Extensions.Configuration;
using Mindbox.MailingsRuntime.Infrastructure;

public static class ConfigBuilderExtensions
{
    private static readonly PhysicalFileProvider _secretsFileProvider =
        new(Path.Combine(Environment.CurrentDirectory, "secrets"))
        {
            UsePollingFileWatcher = true,
            UseActivePolling = true
        };

    public static IConfigurationBuilder AddJsonsConfigsAndSecrets(
        this IConfigurationBuilder configBuilder,
        bool useAuthSecrets = true)
    {
        var contour = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
        var baseDirectory = AppDomain.CurrentDomain.BaseDirectory;

        configBuilder.AddKeyPerFile(Path.Combine(Directory.GetCurrentDirectory(), "ConfigValues"), true);
        configBuilder.AddJsonFile(Path.Combine(baseDirectory, "appsettings.common.json"), false);
        configBuilder.AddJsonFile(Path.Combine(baseDirectory, $"appsettings.{contour}.json"), false);
        configBuilder.AddJsonFile(_secretsFileProvider, "secrets.json", false, true);
        if (useAuthSecrets)
            configBuilder.AddAuthSecrets();

        return configBuilder;
    }
}
```
