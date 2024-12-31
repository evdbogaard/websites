---
title: "Handling Nullable References with Deserialization in .NET 9"
date: 2024-12-31T08:00:00+01:00
draft: false
tags:
  - dotnet
  - csharp
  - dotnet9
  - json
  - system.text.json
---

.NET 9 introduced many features, including long-awaited improvements for JSON deserialization. These updates enhance handling nullable types, which helps avoid common issues like `NullReferenceException`.

Over the years, C# has provided tools to manage nullability, such as enabling nullable annotations and using the `required` modifier. However, deserialization often bypassed these safeguards. Let’s explore how .NET 9 addresses this problem.

## Nullable reference types

Before C# 8.0, `NullReferenceException` errors were common, especially with uninitialized strings. A string without a value defaults to `null`, creating potential runtime issues. Developers often assigned default values to avoid exceptions, but these solutions lacked visibility.

With C# 8.0, nullable reference types were introduced to improve safety by detecting null references. Enabling this feature requires adding `<Nullable>enable</Nullable>` to your project file.

Once enabled, the compiler issues warnings about potentially uninitialized properties. For example, the class below triggers warnings for `FirstName` and `LastName`, as there is no constructor to initialize them:

```C#
class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

To resolve these warnings, you can provide a default value or mark the property as optional:

```C#
class Person
{
    public string FirstName { get; set; } = ""; // Default value ensures it's never null
    public string? LastName { get; set; } // Optional, can still be null
}
```

## The `required` modifier

Setting default values isn't always ideal. In C# 11, the `required` modifier was introduced to enforce property initialization during object creation. A `required` property must be set, or the code won’t compile.

Setting a default value isn't always what we want. In C# 11 the required modifier was introduced which allows us to say "you cannot create a new object of this class, without at least specifying these fields/properties".
It allows us to get rid of the default value and now prevents code compilation if a required property isn't being set.

```C#
class Person
{
    public required string FirstName { get; set; }
    public required string LastName { get; set; }
}

// --- Example

var person = new Person
{
    FirstName = "John"
}; // Compilation error: 'LastName' is required
```

This modifier ensures objects are properly initialized.

## The problem with deserialization

By default, System.Text.Json ignores nullability when deserializing objects. Consider the example below:

```C#
class Person
{
    public required string FirstName { get; set; }
    public required string LastName { get; set; }
}

// ---

var json = "{\"firstName\":\"John\", \"lastName\":null}";
var person = JsonSerializer.Deserialize<Person>(json, options: JsonSerializerOptions.Web);
Console.WriteLine($"Person: {person.FirstName} {person.LastName}"); // Output: "Person: John "
person.LastName.ToUpper(); // Throws NullReferenceException
```

The deserializer creates a `Person` object with `LastName` set to `null`, even though this would not be possible with direct initialization. This behavior undermines the protections provided by nullable reference types and `required` modifiers.

## Respecting Nullable Annotations in .NET 9

.NET 9 addresses this issue with the `RespectNullableAnnotationsDefault` feature. This opt-in feature enforces nullability rules during deserialization, ensuring invalid objects aren’t created. Since it introduces breaking changes, it’s disabled by default but recommended for new projects.

Here's the updated behavior:

```C#
class Person
{
    public required string FirstName { get; set; }
    public required string LastName { get; set; }
}

// ---

var json = "{\"firstName\":\"John\", \"lastName\":null}";
var person = JsonSerializer.Deserialize<Person>(json, options: JsonSerializerOptions.Web);
// Throws JsonException: "The property or field 'lastName' on type 'Person' doesn't allow setting null values"
```

### Enabling `RespectNullableAnnotationsDefault`

To enable this feature globally, add the following to your project file:

```C#
<ItemGroup>
  <RuntimeHostConfigurationOption Include="System.Text.Json.Serialization.RespectNullableAnnotationsDefault" Value="true" />
</ItemGroup>
```

Alternatively, enable it for specific calls:
```C#
JsonSerializerOptions options = new() { RespectNullableAnnotations = true };
JsonSerializer.Deserialize<Person>(json, options: options);
```

## Conclusion

With .NET 9's `RespectNullableAnnotationsDefault` option, developers can better enforce property validity during deserialization. These improvements complement existing nullability tools.

## References

If you want to read more about changes to system.text.json in .NET 9 or nullable references in general, please check out the following links:

- https://devblogs.microsoft.com/dotnet/system-text-json-in-dotnet-9/
- https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references