---
title: "Getting structured JSON from LLMs in C#"
date: 2025-07-25T10:00:00+00:00
draft: false
tags:
  - dotnet
  - csharp
  - AI
  - json
  - system.text.json
  - structured outputs
  - semantic kernel
---

Lately we've been trying to get more hands on with AI and started working on a project that uses a large language model (LLM) to extract specific information from documents. In order to use the output of the prompt in our code we needed the model to return JSON and deserialize that into a structured type.

While this worked well in many cases, we ran into issues where the generated JSON didn't match our expected structure and failed to deserialize. In this blog we'll look at our initial implementation and some possible solutions we found to prevent the deserialization issues.

We began by having the LLM extract information and return it to use as a JSON string. That string was then deserialized into a C# object for further use. For most documents, this approach worked just fine. However, we noticed that in some cases, the model would hallucinate and generate either properties that didn't exist or types we weren't expecting. This lead to deserialization issues causing us to have no data for that specific document. Unfortunately, tweaking the temperature parameter didn't prevent this from happening.

## JSON mode

Our first approach was to use JSON mode. This setting tells the LLM to return a valid JSON object, but it doesn’t enforce the structure beyond that. Here’s a basic code snippet showing how we implemented it. To keep the focus on how JSON mode is used we’ve stripped out the prompt and other details from the code snippet that felt unrelated to the problem.

```csharp
var azureOpenAIClient = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential());
var chatClient = azureOpenAIClient.GetChatClient(deploymentName);

ChatCompletionOptions options = new()
{
    // Omitted out other options...
    ResponseFormat = ChatResponseFormat.CreateJsonObjectFormat()
};

List<ChatMessage> messages = [
    new SystemChatMessage(prompt), // The prompt telling what information to retrieve from the document
    new AssistantChatMessage(example), // JSON example of how the output should look
    new UserChatMessage(documentContent) // Content of the document to process
];

var completion = await chatClient.CompleteChatAsync(messages, options);
var result = JsonSerializer.Deserialize<OutputModel>(completion.Value.Content.First().Text);
```

To increase the chances of having a valid JSON response we did two things. First, we used `ChatResponseFormat.CreateJsonObjectFormat()` to tell the model we expect JSON as the output. Together with this setting you have to specify in your prompt that JSON should be generated. Second, we included an `AssistantChatMessage` containing a valid JSON example of the expected output. It's filled with dummy data and serialized to a string before passed to the model.

The code to do that looks like this:
```csharp
static string CreateExample()
{
    var responseModel = new OutputModel
    {
        PersonNames = [
            "John Doe",
            "Doe, J.",
            "Doe, John",
        ],
        Addresses = [
            new() { Street = "123 Maple Street", City = "Springfield", PostalCode = "62704", Country = "USA" },
            new() { Street = "456 Queen St", City = "Toronto" },
            new() { Street = "789 Rue de Rivoli", City = "Paris", Country = "France" },
            new() { Street = "10 Downing Street", City = "London", PostalCode = "SW1A 2AA", Country = "UK" },
            new() { City = "Cupertino", PostalCode = "95014", Country = "USA" },
        ],
    };

    return JsonSerializer.Serialize(responseModel, options: new() { PropertyNameCaseInsensitive = true });
}

public record OutputModel
{
    public string[] PersonNames { get; set; } = [];
    public AddressModel[] Addresses { get; set; } = [];
}

public record AddressModel
{
	public string Street { get; set; } = "";
	public string City { get; set; } = "";
	public string PostalCode { get; set; } = "";
	public string Country { get; set; } = "";
}
```

Most of the time the LLM generated an output that was correct for us. For example:
`{"PersonNames":["Henk"],"Addresses":[{"City":"Amsterdam","Country":"Netherlands"}]}`
Unfortunately on some occasions it generated an output that was a bit different and would look like this:
`{"PersonNames": ["Henk"], "Addresses": ["Amsterdam"]}`

In that case, the `Addresses` property was a string array instead of an array of `AddressModel`, which led to deserialization errors.

## Structured outputs

Newer models introduced structured outputs, which is an improvement upon the JSON mode and solves this exact problem. While JSON mode guarantees valid JSON, but structured outputs go further by enforcing a specific schema.

To use this, we swapped `CreateJsonObjectFormat()` for `CreateJsonSchemaFormat()` and provided a JSON schema based on our expected model. Since .NET doesn't generate JSON schemas by default, we used the `NJsonSchema` NuGet package for the example code here:

```csharp
// dotnet add package NJsonSchema
var schema = JsonSchema.FromType<OutputModel>();
ChatCompletionOptions options = new()
{
    // Omitted out other options...
    ResponseFormat = ChatResponseFormat.CreateJsonSchemaFormat(
        jsonSchemaFormatName: nameof(OutputModel),
        jsonSchema: BinaryData.FromString(schema.ToJson())
    )
};
```

By using structured outputs we can now trust the LLM to always return a JSON string we can safely deserialize back in our code.

### Using semantic kernel

A different way to call LLMs in C# is by using Semantic Kernel. It provides extra abstractions that make using LLMs easier in code. Structured outputs also has better support as it no longer requires manual JSON schema generation.

Here's what that code looks like when rewritten for Semantic Kernel:
```csharp
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(
        deploymentName: deploymentName,
        endpoint: endpoint,
        credentials: new AzureCliCredential()
    )
    .Build();

var chatClient = kernel.GetRequiredService<IChatCompletionService>();

var messages = new ChatHistory();
messages.AddSystemMessage(prompt);
messages.AddAssistantMessage(example);
messages.AddUserMessage(documentContent);

OpenAIPromptExecutionSettings settings = new()
{
    ResponseFormat = typeof(OutputModel)
};

var completion = await chatClient.GetChatMessageContentAsync(messages, settings);
```

By passing `typeof(OutputModel)` into the `OpenAIPromptExecutionSettings`, Semantic Kernel automatically handled schema generation for us.

## Validating outputs

Structured outputs worked great, but not all models we used supported them. In those cases, we had to fall back on a more manual approach. We built a custom `JsonConverter` to validate and deserialize each property individually. If one value failed, we skipped it and kept the rest. This meant we would still lose some data, but we could display most of the generated output instead of nothing.

```csharp
internal class OutputModelConverter : JsonConverter<OutputModelConverter>
{
    public override OutputModelModel? Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        var model = new OutputModelModel();

        using var doc = JsonDocument.ParseValue(ref reader);

        var root = doc.RootElement;
        model.PersonNames = TryGetStringArray(root, "PersonNames") ?? [];
        model.Addresses = TryGetObjectArray<AddressModel>(root, "Addresses", options) ?? [];

        return model;
    }

    public override void Write(Utf8JsonWriter writer, NerResponseModel value, JsonSerializerOptions options)
    {
        JsonSerializer.Serialize(writer, value, options);
    }

    private static string[]? TryGetStringArray(JsonElement root, string propertyName)
    {
        try
        {
            if (root.TryGetProperty(propertyName, out var property) && property.ValueKind == JsonValueKind.Array)
                return property
                    .EnumerateArray()
                    .Where(e => e.ValueKind == JsonValueKind.String)
                    .Select(e => e.GetString()!)
                    .ToArray();
        }
        catch
        {
            // ...logging which property failed
        }
        return null;
    }

    private static T[]? TryGetObjectArray<T>(JsonElement root, string propertyName, JsonSerializerOptions options)
    {
        if (root.TryGetProperty(propertyName, out var property) && property.ValueKind == JsonValueKind.Array)
        {
            var list = new List<T>();
            foreach (var item in property.EnumerateArray())
            {
                try
                {
                    var obj = item.Deserialize<T>(options);
                    if (obj != null)
                        list.Add(obj);
                }
                catch
                {
                    // ...logging which property failed
                }
            }
            return list.Count > 0 ? [.. list] : null;
        }
        return null;
    }
}
```


## References

If you want to dive deeper into the topics discussed in this post, here are a few helpful links:

- https://platform.openai.com/docs/guides/structured-outputs?api-mode=responses#json-mode
- https://platform.openai.com/docs/guides/structured-outputs?api-mode=responses
- https://devblogs.microsoft.com/semantic-kernel/using-json-schema-for-structured-output-in-net-for-openai-models/
- https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/converters-how-to
- https://github.com/RicoSuter/NJsonSchema