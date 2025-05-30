# Quick Start Guide

Welcome to the Zentient.Results Quick Start Guide! This page will help you get up and running with the library in minutes, covering the essential steps for installation and basic usage.

-----

## 1. Installation

First, add Zentient.Results to your .NET project. You can do this via the NuGet Package Manager Console or the .NET CLI:

```bash
dotnet add package Zentient.Results
```

Zentient.Results is compatible with .NET 6+ and later versions, including .NET 9.

-----

## 2. Core Concepts at a Glance

Before diving into code, here are the fundamental ideas:

  * **`Result<T>`:** Represents the outcome of an operation that produces a value of type `T`. It can be either a success (containing `T`) or a failure (containing error information).
  * **`Result` (Non-Generic):** Represents the outcome of an operation that doesn't produce a specific value (e.g., a void method). It's purely about success or failure.
  * **`ErrorInfo`:** A rich, structured object that provides details about why an operation failed (e.g., error code, message, category, additional data).
  * **`IResultStatus`:** Defines the high-level status of an operation, often aligning with HTTP status codes (e.g., 200 OK, 400 Bad Request). `ResultStatuses` provides common predefined statuses.

The core idea is to *return* results, not *throw* exceptions, for expected business outcomes.

-----

## 3. Basic Usage: Creating Results

Let's look at how to create instances of `Result<T>` for both success and failure scenarios.

### Creating a Success Result

When an operation completes successfully and produces a value:

```csharp
using Zentient.Results;

public class UserService
{
    public IResult<string> GetUserName(int userId)
    {
        // Simulate successful retrieval
        if (userId == 1)
        {
            return Result<string>.Success("Alice"); // Simple success
        }
        return Result<string>.Failure(
            default, // No value for failure
            new ErrorInfo(ErrorCategory.NotFound, "UserNotFound", "User not found."),
            ResultStatuses.NotFound
        );
    }
}
```

### Creating a Failure Result

When an operation fails, you use a `Failure` factory method, providing `ErrorInfo` and an `IResultStatus`.

```csharp
using Zentient.Results;

public class ProductService
{
    public IResult<Product> GetProduct(string productId)
    {
        if (string.IsNullOrWhiteSpace(productId))
        {
            return Result<Product>.Failure(
                default, // No product value for this failure
                new ErrorInfo(ErrorCategory.Validation, "InvalidProductId", "Product ID cannot be empty."),
                ResultStatuses.BadRequest // Use a standard status
            );
        }

        // Simulate not found scenario
        if (productId == "NON_EXISTENT_ID")
        {
            return Result<Product>.NotFound( // Convenience method for 404
                new ErrorInfo(ErrorCategory.NotFound, "ProductNotFound", $"Product {productId} not found.")
            );
        }

        // ... (simulate success path)
        return Result<Product>.Success(new Product { Id = productId, Name = "Example Product" });
    }
}

public class Product { public string Id { get; set; } public string Name { get; set; } }
```

### Creating a Non-Generic Result (for void operations)

For operations that just indicate success or failure without returning a value:

```csharp
using Zentient.Results;

public class EmailService
{
    public IResult SendWelcomeEmail(string emailAddress)
    {
        if (!emailAddress.Contains("@"))
        {
            return Result.Failure(
                new ErrorInfo(ErrorCategory.Validation, "InvalidEmail", "Email address format is invalid."),
                ResultStatuses.BadRequest
            );
        }
        // Simulate sending email
        Console.WriteLine($"Sending welcome email to {emailAddress}...");
        return Result.Success(); // Non-generic success
    }
}
```

-----

## 4. Basic Usage: Consuming Results

Once you have a `Result` object, you can check its state and access its contents.

```csharp
using Zentient.Results;
using System;

public class ApplicationFlow
{
    public void ProcessUserRequest()
    {
        var userService = new UserService(); // From previous example

        // Scenario 1: Successful operation
        IResult<string> successResult = userService.GetUserName(1);
        if (successResult.IsSuccess)
        {
            string userName = successResult.Value; // Access the value
            Console.WriteLine($"Successfully retrieved user: {userName}");
        }
        else
        {
            // This block won't be hit for successResult
            Console.WriteLine($"Operation failed: {successResult.Error}"); // Access the first error message
        }

        // Scenario 2: Failed operation
        IResult<string> failureResult = userService.GetUserName(99);
        if (failureResult.IsFailure)
        {
            Console.WriteLine($"Operation failed with status {failureResult.Status.Code}: {failureResult.Error}");
            // Access all errors:
            foreach (var error in failureResult.Errors)
            {
                Console.WriteLine($"- Error ({error.Category}/{error.Code}): {error.Message}");
            }
        }
        else
        {
            // This block won't be hit for failureResult
            Console.WriteLine($"Operation succeeded: {failureResult.Value}");
        }
    }
}
```

-----

## 5. Chaining Operations (Fluent API Introduction)

Zentient.Results provides a fluent API to chain operations, making your code concise and robust.

### `Map`: Transform the Value on Success

`Map` allows you to transform the *successful value* of a `Result<T>` into a `Result<U>` without changing the success/failure state.

```csharp
using Zentient.Results;
using System;

// Assume UserService.GetUserName(int) from above
public class UserService
{
    public IResult<string> GetUserName(int userId) { /* ... as before ... */ return Result<string>.Success("Alice"); }
}

public class ReportGenerator
{
    public IResult<int> GetUserNameLength(int userId)
    {
        return new UserService().GetUserName(userId)
            .Map(userName => userName.Length); // Map string to int (length) if successful
    }

    public void Run()
    {
        var lengthResult = GetUserNameLength(1);
        if (lengthResult.IsSuccess)
        {
            Console.WriteLine($"User name length: {lengthResult.Value}"); // Output: User name length: 5
        }

        var failedLengthResult = GetUserNameLength(0); // This would return a failure from GetUserName
        if (failedLengthResult.IsFailure)
        {
            Console.WriteLine($"Could not get user name length: {failedLengthResult.Error}");
        }
    }
}
```

### `Bind`: Chain Operations that Also Return a Result

`Bind` allows you to chain a new operation that also returns a `Result`. This is crucial for sequential operations where each step can fail.

```csharp
using Zentient.Results;
using System;

// Assume UserService.GetUserName(int) from above
public class UserService
{
    public IResult<string> GetUserName(int userId) { /* ... as before ... */ return Result<string>.Success("Alice"); }
}

public class ValidationService
{
    public IResult<string> ValidateName(string name)
    {
        if (name.Length > 10)
        {
            return Result<string>.Failure(
                name, // Can optionally pass the problematic value
                new ErrorInfo(ErrorCategory.Validation, "NameTooLong", "Name exceeds 10 characters."),
                ResultStatuses.BadRequest
            );
        }
        return Result<string>.Success(name);
    }
}

public class ProfileProcessor
{
    public IResult<string> ProcessUser(int userId)
    {
        var userService = new UserService();
        var validationService = new ValidationService();

        return userService.GetUserName(userId)
            .Bind(userName => validationService.ValidateName(userName)); // Only proceeds if GetUserName was successful
    }

    public void Run()
    {
        var result1 = ProcessUser(1); // User "Alice" (length 5)
        if (result1.IsSuccess)
        {
            Console.WriteLine($"Processed user: {result1.Value}"); // Output: Processed user: Alice
        }

        var result2 = ProcessUser(99); // Simulates user not found from GetUserName
        if (result2.IsFailure)
        {
            Console.WriteLine($"Failed to process user: {result2.Error}"); // Output: Failed to process user: User not found.
        }

        // If GetUserName returned a long name, ValidateName would fail.
    }
}
```

-----

## 6. Next Steps

You've now learned the absolute basics of Zentient.Results! To explore more advanced features and deeper architectural insights, please refer to these wiki pages:

  * **[Basic Usage Patterns](Basic-Usage-Patterns)**: More examples and common scenarios for creating and consuming results.
  * **[Structured Error Handling with `ErrorInfo`](Structured-Error-Handling-with-ErrorInfo-and-ErrorCategory)**: A detailed dive into the error model.
  * **[Chaining and Composing Operations](Chaining-and-Composing-Operations)**: Advanced usage of `Map`, `Bind`, `Then`, `OnSuccess`, `OnFailure`, and `Match`.
  * **[Integrating with ASP.NET Core](Integrating-with-ASP.NET-Core)**: Guidance on using Zentient.Results in web applications.
  * **[Architecture Overview](https://github.com/ulfbou/Zentient.Results/blob/main/ARCHITECTURE.md)**: Understand the library's design principles.
  * **[Contributing](https://github.com/ulfbou/Zentient.Results/blob/main/CONTRIBUTING.md)**: Learn how to contribute to the project.

For the full source code and README, visit the [Zentient.Results GitHub Repository](https://github.com/ulfbou/Zentient.Results).
