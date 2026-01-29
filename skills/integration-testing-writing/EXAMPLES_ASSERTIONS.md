# Примеры проверок (Assertions)

## AssertMessagesAreSentCorrectly - проверка сообщений из Kafka (через Consumer, если надо проверить, что нужные сообщения попали в топик)

```csharp
using System.Text.Json;
using System.Text.Json.Nodes;
using FluentAssertions;

public static class Assertions
{
    public static void AssertMessagesAreSentCorrectl<T>(
        IEnumerable<T> actualMessages,
        IEnumerable<T> expectedMessages,
        int messagesCount,
        Func<T, object> keySelector)
    {
        var expectedDictionary = expectedMessages.ToDictionary(m => keySelector(m));
        var processed = new HashSet<object>();
        
        foreach (var actual in actualMessages)
        {
            var key = selector(actual);
            if (!expectedDictionary.TryGetValue(key, out var expectedMessage)) continue;

            actual.Should().BeEquivalentTo(expectedMessage);

            processed.Add(key);
            if (processed.Count == messagesCount) break;
        }

        processed.Count.Should().Be(messagesCount);
    }
}
```

**Использование:**

```csharp
//Assert
var actualMessages = _kafkaContainer.ConsumeMessages("output-topic", "test-consumer").Select(z => JsonSerializer.Deserialize<Dto>(z.Message));
var expectedMessages = inputMessages
    .Select(input => GenerateExpected(input))
    .Select(e => JsonSerializer.Serialize(e))
    .ToList();

Assertions.AssertMessagesAreSentCorrectly(
    actualMessages, 
    expectedMessages, 
    inputMessages.Count,
    m => m.Id);
```