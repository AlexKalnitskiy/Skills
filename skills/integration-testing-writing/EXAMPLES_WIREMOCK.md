# Примеры работы с WireMock

## Простой HTTP endpoint мок

```csharp
using WireMock.RequestBuilders;
using WireMock.ResponseBuilders;
using WireMock.Server;

public static class SimpleWireMockExtensions
{
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
}
```

## Пример с GraphQL запросом (MailingsClientWireMockBuilder)

```csharp
using WireMock.RequestBuilders;
using WireMock.ResponseBuilders;
using WireMock.Server;
using WireMock.Matchers;
using Newtonsoft.Json;

namespace TestingHelpers.WireMocks;

public static class MailingsClientWireMockBuilder
{
    extension(WireMockServer server)
    {
        /// <summary>
        /// Given mailing will return if internalId and tenant (if specified) match
        /// </summary>
        public WireMockServer WithMailingMetadata(
            McsMailingMetadata originalMetadata,
            TenantSystemName? tenantSystemName = null)
        {
            var mailingRequest = Request.Create()
                .WithPath("/graphql")
                .WithBody(
                    new JsonPartialWildcardMatcher(
                        $$"""
                        {
                            "variables":
                                {
                                    "internalId": "{{originalMetadata.InternalId}}",
                                    "tenantSystemName": "{{tenantSystemName ?? "*"}}"
                                }
                        }
                        """))
                .UsingPost()
                .WithHeader("Accept", "application/graphql-response+json, application/json");

            var response = Response.Create()
                .WithHeader("Content-Type", "application/json")
                .WithBody(request => CreateResponse(request, originalMetadata));

            server.Given(mailingRequest).RespondWith(response);

            return server;
        }
    }

    private static string CreateResponse(IRequestMessage request, McsMailingMetadata originalMetadata)
    {
        // Логика создания ответа на основе GraphQL запроса
        // ...
        var response = new
        {
            data = new
            {
                mailing = new
                {
                    metadata = originalMetadata,
                    errors = (object?)null
                }
            }
        };
        return JsonConvert.SerializeObject(response);
    }
}
```

## Пример простого GET endpoint (WebPushClientWireMockBuilder)

```csharp
using WireMock.RequestBuilders;
using WireMock.ResponseBuilders;
using WireMock.Server;
using Newtonsoft.Json;

namespace TestingHelpers.WireMocks;

public static class WebPushClientWireMockBuilder
{
    extension(WireMockServer server)
    {
        public void AddWebPushClient(UserAgentWebPushImageSizeDto[] knownSizes)
        {
            var mailingRequest = Request.Create()
                .WithPath("/image-well-known-sizes")
                .UsingGet();

            var response = Response.Create()
                .WithHeader("Content-Type", "application/json")
                .WithBody(JsonConvert.SerializeObject(knownSizes));

            server.Given(mailingRequest).RespondWith(response);
        }
    }
}
```

## Пример настройки нескольких WireMock серверов в TestApp

```csharp
using WireMock.Server;

[TestClass]
public static class TestApp
{
    public static WireMockServer MailingsMicroservice { get; } = WireMockServer.Start();
    public static WireMockServer TenantsMicroservice { get; } = WireMockServer.Start();
    public static WireMockServer WebPushGate { get; } = WireMockServer.Start();

    [AssemblyInitialize]
    public static async Task InitializeAsync(TestContext context)
    {
        // Настройка MailingsMicroservice
        _ = MailingsPresets.Mailings.Aggregate(
            MailingsMicroservice, 
            (server, pair) => server.WithMailingMetadata(pair.Value));

        // Настройка WebPushGate
        WebPushGate.AddWebPushClient(
        [
            new UserAgentWebPushImageSizeDto
            {
                BrowserFamily = "Chrome",
                BrowserVersion = "100.0.0",
                OperatingSystemFamily = "Windows",
                OperatingSystemVersion = "10.0.0",
                Width = 1280,
                Height = 640
            }
        ]);

        // Настройка TenantsMicroservice
        SetupTenantsClientResponse();
    }

    private static void SetupTenantsClientResponse()
    {
        var tenantRequest = Request.Create()
            .WithPath($"v2/tenants/configuration?tenant={PresetsHelpers.TenantSystemName}&keys=locale")
            .UsingGet();

        var response = new
        {
            status = "Active",
            name = PresetsHelpers.TenantSystemName,
            systemName = PresetsHelpers.TenantSystemName,
            configuration = new Dictionary<string, string>
            {
                { "locale", "en-US" }
            }
        };

        var json = JsonConvert.SerializeObject(response);
        var tenantsResponse = Response.Create()
            .WithHeader("Content-Type", "application/json")
            .WithBody(json);

        TenantsMicroservice.Given(tenantRequest).RespondWith(tenantsResponse);
    }

    public static IEnumerable<KeyValuePair<string, string?>> GetConfigOverridenByTestContainers()
    {
        var configOverrides = new Dictionary<string, string?>
        {
            ["Mailings:ServicesUrl"] = MailingsMicroservice.Url,
            ["Nexus:ServicesUrl"] = TenantsMicroservice.Url,
            [$"{WebPushGateOptions.SectionName}:{nameof(WebPushGateOptions.Host)}"] = WebPushGate.Url
        };

        return configOverrides;
    }
}
```

## Использование WireMock в тестах

```csharp
[TestClass]
public class MyIntegrationTests
{
    private static WireMockServer _externalServiceMock = null!;

    [ClassInitialize]
    public static async Task Initialize(TestContext testContext)
    {
        _externalServiceMock = WireMockServer.Start();
        
        // Настройка моков
        _externalServiceMock.WithSomeEndpoint(new SomeDto { Id = 1 });
    }

    [ClassCleanup]
    public static void Cleanup()
    {
        _externalServiceMock.Dispose();
    }
}
```
