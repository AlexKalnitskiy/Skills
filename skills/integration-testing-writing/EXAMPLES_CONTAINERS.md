# Примеры работы с контейнерами

## Kafka Container

```csharp
using Confluent.Kafka;
using Confluent.Kafka.Admin;
using Testcontainers.Kafka;

namespace TestingHelpers.Containers;

public static class KafkaContainerExtensions
{
    private const string BootstrapServersKey = "bootstrap.servers";

    public static KafkaContainer CreateKafka()
    {
        var container = new KafkaBuilder("confluentinc/cp-kafka:7.4.0").Build();
        return container;
    }

    public static string GetBootstrapServers(this KafkaContainer container) =>
        $"{container.Hostname}:{container.GetMappedPublicPort(9092)}";

    public static async Task CreateTopicsAsync(
        this KafkaContainer controller,
        IEnumerable<string> topicNames,
        Action<TopicSpecification>? factory = null)
    {
        using var client = new AdminClientBuilder(
            new Dictionary<string, string>
            {
                [BootstrapServersKey] = controller.GetBootstrapServers()
            }).Build();
        var topicSpecs = topicNames.Select(name =>
        {
            var spec = new TopicSpecification { Name = name, NumPartitions = 4, ReplicationFactor = 1 };
            factory?.Invoke(spec);
            return spec;
        }).ToList();
        try
        {
            await client.CreateTopicsAsync(topicSpecs);
        }
        catch (KafkaException)
        {
            // Топик уже существует, игнорируем
        }
    }

    public static async Task ProduceMessagesToTopicAsync(
        this KafkaContainer container,
        string topicName,
        IEnumerable<string> messages)
    {
        var producerConfig = new ProducerConfig
        {
            BootstrapServers = container.GetBootstrapServers(),
            Acks = Acks.All,
            MessageMaxBytes = 2097152
        };
        using var producer = new ProducerBuilder<string, string>(producerConfig).Build();
        await Parallel.ForEachAsync(
            messages,
            async (message, token) => await producer.ProduceAsync(
                topicName,
                new Message<string, string> { Value = message },
                token));
    }

    public static IEnumerable<string> ConsumeMessages(
        this KafkaContainer controller,
        string topic,
        string consumerGroupId,
        int messagesToConsume = int.MaxValue,
        TimeSpan? timeout = null)
    {
        var consumerConfig = new ConsumerConfig
        {
            BootstrapServers = controller.GetBootstrapAddress(),
            GroupId = consumerGroupId,
            AutoOffsetReset = AutoOffsetReset.Earliest,
            EnableAutoCommit = true
        };
        using var consumer = new ConsumerBuilder<string, string>(consumerConfig).Build();
        consumer.Subscribe(topic);
        var consumedCount = 0;
        while (consumedCount < messagesToConsume)
        {
            var result = consumer.Consume(timeout ?? TimeSpan.FromHours(24));
            if (result != null)
            {
                yield return result.Message.Value;
                consumedCount++;
            }
            else
            {
                break; // Timeout reached
            }
        }

        consumer.Close();
    }
}
```

## Redis Container

```csharp
using System.Net;
using StackExchange.Redis;
using Testcontainers.Redis;

namespace TestingHelpers.Containers;

public static class RedisContainerExtensions
{
    public static RedisContainer CreateRedis()
    {
        var container = new RedisBuilder("redis:7.0-alpine")
            .Build();

        return container;
    }

    public static string GetRedisHost(this RedisContainer container) => container.Hostname;

    public static int GetRedisPort(this RedisContainer container) => container.GetMappedPublicPort(6379);

    public static async Task FlushRedisAsync(this RedisContainer container)
    {
        var config = new ConfigurationOptions
        {
            EndPoints = new EndPointCollection(
                [new DnsEndPoint(container.GetRedisHost(), container.GetRedisPort())]),
            AllowAdmin = true,
            AbortOnConnectFail = false
        };

        await using var multiplexer = await ConnectionMultiplexer.ConnectAsync(config);
        await multiplexer.GetServer(multiplexer.GetEndPoints().First()).FlushAllDatabasesAsync();
    }
}
```

## Cassandra Container с Flyway

```csharp
using DotNet.Testcontainers.Builders;
using DotNet.Testcontainers.Containers;
using DotNet.Testcontainers.Networks;
using Testcontainers.Cassandra;

namespace TestingHelpers.Containers;

public static class CassandraContainerExtensions
{
    private const string CassandraHost = "cassandra";
    private const string DefaultDatacenter = "dc1";
    private const int CassandraPort = 9042;

    // Создаем Network для связи контейнеров
    private static readonly INetwork _network = new NetworkBuilder()
        .WithName($"cassandra-network-{Guid.NewGuid()}")
        .Build();

    public static CassandraContainer CreateCassandra()
    {
        var container = new CassandraBuilder()
            .WithNetwork(_network)
            .WithNetworkAliases(CassandraHost)
            .WithWaitStrategy(
                Wait.ForUnixContainer()
                    .UntilInternalTcpPortIsAvailable(CassandraPort)
                    .UntilMessageIsLogged("Startup complete"))
            .Build();
        return container;
    }

    public static IContainer CreateFlywayContainer() => new ContainerBuilder("flyway/flyway")
        .WithBindMount(Path.GetFullPath("./cql"), "/flyway/sql")
        .WithNetwork(_network)
        .WithCommand(
            $"-url={CreateCassandraConnectionString()}?localdatacenter={DefaultDatacenter}",
            "-configFiles=/flyway/sql/flyway.toml",
            "migrate")
        .WithWaitStrategy(
            Wait.ForUnixContainer()
                .UntilMessageIsLogged("Successfully applied"))
        .Build();

    private static string CreateCassandraConnectionString() => $"jdbc:cassandra://{CassandraHost}:{CassandraPort}";
}
```

**Важно**: Для использования Flyway нужно скопировать миграции в тестовом проекте:

```xml
<ItemGroup>
  <Content Include="..\..\cql\**\*">
    <Link>cql\%(RecursiveDir)%(Filename)%(Extension)</Link>
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

## MinIO/S3 Container

```csharp
using Testcontainers.MinIO;

namespace TestingHelpers.Containers;

public static class MinioContainerExtensions
{
    public static MinioContainer CreateMinio()
    {
        var container = new MinioBuilder("minio/minio:latest")
            .Build();
        return container;
    }

    public static string GetS3Url(this MinioContainer container) => 
        $"http://{container.Hostname}:{container.GetMappedPublicPort(9000)}";
}
```

## PostgreSQL Container

```csharp
using Testcontainers.PostgreSql;

namespace TestingHelpers.Containers;

public static class PostgreSqlContainerExtensions
{
    public static PostgreSqlContainer CreatePostgreSql()
    {
        var container = new PostgreSqlBuilder()
            .WithImage("postgres:15-alpine")
            .WithDatabase("testdb")
            .WithUsername("testuser")
            .WithPassword("testpass")
            .Build();
        return container;
    }

    public static string GetConnectionString(this PostgreSqlContainer container) =>
        container.GetConnectionString();
}
```

## Использование Network для связанных контейнеров

Когда контейнеры должны видеть друг друга (например, Cassandra и Flyway), создай Network:

```csharp
private static readonly INetwork _network = new NetworkBuilder()
    .WithName($"my-network-{Guid.NewGuid()}")
    .Build();

// Используй один и тот же Network для всех связанных контейнеров
var container1 = new ContainerBuilder()
    .WithNetwork(_network)
    .WithNetworkAliases("service1")
    .Build();

var container2 = new ContainerBuilder()
    .WithNetwork(_network)
    .WithNetworkAliases("service2")
    .Build();

// Теперь container2 может обращаться к container1 по имени "service1"
```
