# AGENTS.md - R3 Reactive Extensions

This document provides guidelines for agentic coding agents working in this repository.

## Project Overview

R3 is a modern reimplementation of Reactive Extensions (Rx) for .NET, supporting multiple platforms including Unity, Godot, Avalonia, WPF, WinForms, WinUI3, Stride, MAUI, MonoGame, Blazor, and Uno. The core library is in `src/R3/` and tests are in `tests/R3.Tests/`.

## Build Commands

### Building the Solution

```bash
# Build in Release mode (recommended for CI)
dotnet build -c Release

# Build in Debug mode
dotnet build -c Debug
```

### Running Tests

```bash
# Run all tests in Release mode
dotnet test -c Release

# Run all tests in Debug mode
dotnet test -c Debug

# Run a single test class
dotnet test --filter "FullyQualifiedName~SubjectTest"

# Run a single test method
dotnet test --filter "FullyQualifiedName~SubjectTest.Test"
```

### Creating Packages

```bash
# Create NuGet packages (Release mode)
dotnet pack -c Release

# Create packages with symbols
dotnet pack -c Release -p:IncludeSymbols=true -p:SymbolPackageFormat=snupkg
```

## Code Style Guidelines

### General Formatting

- **Indentation**: 4 spaces for C# files, 2 spaces for other files
- **Line endings**: LF (Unix-style)
- **Charset**: UTF-8 with BOM for C# files
- **Trailing whitespace**: Trimmed
- **Newlines**: Final newline at end of file
- **Braces**: All braces on new lines (K&R style via `csharp_new_line_before_open_brace = all`)

### Namespace Declarations

- Use file-scoped namespaces: `namespace R3;` (not block-scoped)
- Using directives go outside the namespace

```csharp
using System.Diagnostics;

namespace R3;
```

### Type Usage

- Use **explicit types** instead of `var` (this is enforced: `csharp_style_var_elsewhere = false`)
- Use **predefined types** (`int`, `string`, `bool`, etc.) instead of BCL types (`Int32`, `String`, `Boolean`)

```csharp
// Good
int count = 0;
string name = "test";
var subject = new Subject<int>(); // var is OK when type is apparent from constructor

// Avoid
var count = 0;
var name = "test";
```

### Naming Conventions

- **Public members** (properties, methods, fields, events): PascalCase
- **Private/internal fields**: `_camelCase` with underscore prefix
- **Static private fields**: `s_camelCase` with `s_` prefix
- **Constants**: PascalCase
- **Avoid `this.` prefix** unless absolutely necessary

```csharp
public class MyClass
{
    public int PublicProperty { get; }

    private readonly int _internalField;
    private static int s_counter;
    private const int MaxCount = 100;

    public void PublicMethod()
    {
        var local = _internalField;
    }
}
```

### Modifier Order

Follow suggested order: `public, private, protected, internal, static, extern, new, virtual, abstract, sealed, override, readonly, unsafe, volatile, async`

### Using Statements

- Sorting: Let IDE handle organize imports
- Prefer explicit type names when the type is clear from context

### Error Handling

- R3 uses `OnErrorResume` instead of stopping on error (core design philosophy)
- Use `Result` struct to check success/failure: `result.IsSuccess` / `result.IsFailure`
- For unhandled exceptions, use `ObservableSystem.RegisterUnhandledExceptionHandler`

```csharp
observer.OnCompleted(Result.Success);
observer.OnCompleted(Result.Failure(exception));
```

### Expression-Bodied Members

Disabled by default. Use explicit braces:

```csharp
// Avoid
public int Value => _value;

// Prefer
public int Value { get { return _value; } }
```

### Collection Expressions

Use collection expressions when appropriate:

```csharp
// Good
var list = new List<int> { 1, 2, 3 };
int[] array = [1, 2, 3];

// When chaining
list.ShouldBe([1, 2, 3]);
```

### Null Handling

- Use null-coalescing (`??`) and null-conditional (`?.`) operators
- Enable nullable reference types (project uses `<Nullable>enable</Nullable>`)

## Testing Guidelines

### Test Framework

- **xUnit v3** for testing
- **Shouldly** for assertions
- **Microsoft.Extensions.TimeProvider.Testing** for time-based tests

### Test Patterns

```csharp
using R3;
using Xunit;
using Shouldly;
using Microsoft.Extensions.Time.Testing;

public class SubjectTest
{
    [Fact]
    public void Test()
    {
        var subject = new Subject<int>();
        using var list = subject.ToLiveList();
        
        subject.OnNext(1);
        subject.OnNext(2);
        subject.OnCompleted();
        
        list.AssertEqual([1, 2]);
        list.AssertIsCompleted();
    }
}
```

### Time-Based Testing

Use `FakeTimeProvider` and `FakeFrameProvider` for time-sensitive operators:

```csharp
var fakeTime = new FakeTimeProvider();
var list = Observable.Timer(TimeSpan.FromSeconds(5), fakeTime).ToLiveList();

fakeTime.Advance(TimeSpan.FromSeconds(5));
list.AssertIsCompleted();
list.AssertEqual([Unit.Default]);
```

### Test Helper Extensions

Located in `tests/R3.Tests/_TestHelper.cs`:

- `AssertEqual<T>(params T[] expected)` - Assert list contents
- `AssertIsCompleted()` / `AssertIsNotCompleted()` - Assert completion status
- `AssertEmpty<T>()` - Assert empty list
- `Advance(this FakeTimeProvider, int seconds)` - Helper to advance time

## Key Architectural Patterns

### Observable/Observer Pattern

R3 uses custom interfaces, not `IObservable<T>`:

```csharp
public abstract class Observable<T>
{
    public IDisposable Subscribe(Observer<T> observer);
}

public abstract class Observer<T> : IDisposable
{
    public void OnNext(T value);
    public void OnErrorResume(Exception error);
    public void OnCompleted(Result result);
}
```

### Subscription Management

Always dispose subscriptions to prevent memory leaks:

```csharp
// Combine multiple subscriptions
var d = Disposable.Combine(sub1, sub2, sub3);
d.Dispose();

// Or use DisposableBuilder for dynamic additions
var builder = Disposable.CreateBuilder();
observable.Subscribe().AddTo(ref builder);
var subscription = builder.Build();
```

### TimeProvider and FrameProvider

- Use `TimeProvider` (not IScheduler) for time-based operations
- Use `FrameProvider` for frame-based operations (Unity, game loops)
- Set defaults via `ObservableSystem.DefaultTimeProvider` and `ObservableSystem.DefaultFrameProvider`

## Important Notes

1. **No `this.` prefix**: Avoid unless necessary for disambiguation
2. **Explicit types**: Don't use `var` except when type is immediately apparent from constructor
3. **Block-scoped namespaces**: Use file-scoped (`namespace X;`)
4. **Expression-bodied disabled**: Use explicit braces
5. **Tests use collection expressions**: `ShouldBe([1, 2, 3])` not `ShouldBe(new[] { 1, 2, 3 })`
