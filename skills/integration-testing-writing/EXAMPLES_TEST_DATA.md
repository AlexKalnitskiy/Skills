# Примеры тестовых данных и пресетов

## Структура TestData с генераторами

```csharp
public static class TestData
{
    public static TestScenario Scenario1 => new()
    {
        Name = "Scenario1",
        InputGenerator = i => new InputDto { Id = i, Name = $"Test{i}" },
        ExpectedGenerator = (input, dateTime) => new ExpectedDto 
        { 
            Id = input.Id, 
            ProcessedAt = dateTime 
        }
    };

    public static TestScenario Scenario2 => new()
    {
        Name = "Scenario2",
        InputGenerator = i => new InputDto { Id = i, Type = "Special" },
        ExpectedGenerator = (input, dateTime) => new ExpectedDto 
        { 
            Id = input.Id, 
            Type = input.Type,
            ProcessedAt = dateTime 
        }
    };

    public static Dictionary<string, TestScenario> GetScenarios() => new()
    {
        [nameof(Scenario1)] = Scenario1,
        [nameof(Scenario2)] = Scenario2
    };
}

public record TestScenario
{
    public required string Name { get; init; }
    public required Func<int, InputDto> InputGenerator { get; init; }
    public required Func<InputDto, DateTime, ExpectedDto> ExpectedGenerator { get; init; }
}
```

## Использование в параметризованных тестах

```csharp
[TestMethod]
[Timeout(60000)]
[DataRow(nameof(TestData.Scenario1), "expected-topic")]
[DataRow(nameof(TestData.Scenario2), "expected-topic")]
public async Task ShouldProcessCorrectly(string scenarioKey, string targetTopic)
{
    //test
}
 

## Константы для тестов

```csharp
public static class TestConstants
{
    public const string TenantSystemName = "TestTenant";
    public const int DefaultMessagesCount = 100;
    public const int DefaultTimeoutSeconds = 60;
    public const string TestConsumerGroupPrefix = "test-consumer";
}

// Использование
const int messagesCount = TestConstants.DefaultMessagesCount;
var consumerGroupId = $"{TestConstants.TestConsumerGroupPrefix}-{scenarioKey}";
```

## Генерация тестовых данных с разными сценариями

```csharp
public static class TestDataGenerators
{
    public static CustomerDataResponse GenerateCustomerData(
        McsMailingMetadata mailing, 
        int index)
    {
        return new CustomerDataResponse
        {
            Metadata = new CustomerDataMetadata
            {
                TransactionalId = index,
                TenantSystemName = TestConstants.TenantSystemName
            },
            Response = new CustomerData
            {
                CustomerId = $"customer-{index}",
                TenantSystemName = TestConstants.TenantSystemName,
                CustomerIanaTimeZone = "Europe/Moscow"
            }
        };
    }

    public static ExpectedMessageDto GenerateExpected(
        McsMailingMetadata mailing,
        CustomerDataResponse input,
        DateTime dateTime)
    {
        return new ExpectedMessageDto
        {
            TransactionalId = input.Metadata.TransactionalId,
            CustomerId = input.Response.CustomerId,
            MailingId = mailing.InternalId,
            ProcessedAt = dateTime
        };
    }
}
```
