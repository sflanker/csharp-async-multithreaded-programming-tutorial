# Task Cancellation

Any function that returns a Task _should_ take a [`CancellationToken`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken). The purpose of this is to provide a generic mechanism to interrupt potentially long running IO operations (like a database query for example), even when that operation is deeply nested in a complex call hierarchy. Most async code **only needs to pass the `CancellationToken` to other async functions**. Code that actually performs low level IO (i.e. system calls to read/write data from a network or filesystem) needs to translate the `CancellationToken` based notification into termination of the IO operation. The one exception would be CPU intensive code. For example, in or function that computes PI it would be a good idea to check if cancellation has been requested:

```csharp
static Task<Decimal> DoSomethingCpuIntensive(
  Int32 iterations,
  CancellationToken cancellation) {
  
  return Task.Run(() => {
    var pi = 4m;
    for (var i = 0; i < iterations; i++) {
      // In synchronous CPU bound code it is necessary to check for
      // cancellation explicitly.
      cancellation.ThrowIfCancellationRequested();
      pi += (i % 2 == 0 ? -1 : 1) * 4m / (2 * i + 3);
    }
    
    return pi;
  });
}
```

Code that awaits other async operations is much simpler to add cancellation support to:

```csharp
static async Task DoSomethingAsync(
  Int32 count,
  CancellationToken cancellation) {
  
  for (var i = 0; i < count; i++) {
    await Task.Delay(500, cancellation);
    Console.Write(".");
  }

  await Task.Delay(500, cancellation);

  Console.WriteLine();
}
```

In order to control cancellation, you need to create a `CancellationTokenSource`, and then use some event to trigger cancellation. This simple example just uses a delay:

```csharp
public static async Task Main() {
  Console.WriteLine("Starting");
  using var cancellationSource = new CancellationTokenSource();
  
  // CPU Intensive task starts executing automatically
  var task = DoSomethingCpuIntensive(50000000, cancellationSource.Token);
  Console.Write("CPU Bound Task Created");
  
  // Do something IO related while computations are being executed
  var spinner = DoSomethingAsync(10, cancellationSource.Token);
  
  // Wait for the CPU bound task to complete, or the delay, whichever
  // comes first.
  var completed = await (await Task.WhenAny<Decimal>(
    task,
    Task.Delay(2000).ContinueWith(_ => -1m)
  ));
  
  if (completed > 0) {
    Console.WriteLine($"Done: {completed}");
  } else {
    Console.WriteLine($"Too Many Iterations");
  }

  // Cancel the spinner (and the CPU bound work if it is still running)
  cancellationSource.Cancel();

  // Wait for all tasks to finish
  try {
    await Task.WhenAll(new[] { spinner, task });
  } catch (OperationCanceledException) {
    // This exception is raised when a task is canceled, so it is expected
  }
}
```

## Timeouts

A `CancellationTokenSource` can also be created with a timeout. The `CancellationTonkenSource` will automatically be cancelled after the timeout has ellapsed (this can simplify the code above by obviating the need for the `Task.Delay` and `Task.WhenAny`).

## Disposable

The `CancellationTokenSource` type implementes `IDisposable` so it is important to declare it as a `using var` or with a `using` block. Disposing a `CancellationTokenSource` cleans up the resources created to apply the specified timeout. A `CancellationTokenSource` should only be disposed after all tasks using its `CancellationToken` have completed.

## Exercise

Create a program that simulates a virtual race between three competitors: "Red", "Green", and "Blue". Three Tasks should be started simultaneously and each should wait a random amount of time between 1 and 60 seconds before returning the name of one of the three competitors. Which ever function returns first should be announced as the winner. However, the spectators might get tired of waiting, so if the user presses \[Enter] before the end of the race all of the tasks should be cancelled immediately and the race should be called a draw.

### 🚨 Gotcha Alert

The `ReadLineAsync` method on `TextReader` (which is what `Console.In` is) does not have cancellation support on .Net 6.0 and even in .Net 8.0 it actually synchronously blocks the current thread 🤦‍♂️ (this shameful footgun is [documented here](https://learn.microsoft.com/en-us/dotnet/api/system.console.in?view=net-8.0#remarks)).

As a workaround you could simulate asynchrony via `Task.Yield` (see the Task Helpers section), or you could use the actual `Stream` instance for stdin. You can utilize the workaround below, or implement one yourself as part of the exercise.

<details>
<summary><strong>Stream based workaround</strong></summary>

```csharp
/// <summary>
///   Asynchronously reads the first line of input from stdin.
/// </summary>
/// <remarks>
///   This implementation discards any extra bytes that are read and
///   disposes the stdin Stream. It is not suitable for a use case that
///   requires reading multiple lines of input.
/// <remarks>
async Task<String> ActuallyReadLineAsync(CancellationToken cancellationToken) {
  var sb = new System.Text.StringBuilder();
  var buffer = new Byte[1024];
  using var stdin = Console.OpenStandardInput();
  while (true) {
    // Note: ConsoleStream doesn't correctly implement ReadAsync, it just
    // ignores the CancellationToken if it is passed in.
    var len = await stdin.ReadAsync(buffer, 0, 1024).WaitAsync(cancellationToken);
    if (len > 0) {
      var str = System.Text.Encoding.UTF8.GetString(buffer, 0, len);
      var newlineIx = str.IndexOf('\n');
      if (newlineIx >= 0) {
        sb.Append(str.Substring(0, newlineIx));
        break;
      } else {
        sb.Append(str);
      }
    } else {
      break;
    }
  }

  return sb.ToString();
}
```
</details>

### Solution

<details>
<summary><strong>Solution</strong></summary>

```csharp
public static async Task Main() {
  using var cts = new CancellationTokenSource();

  var rand = new Random();
  var raceTasks = new [] {
    Race("Red", rand.Next(1000, 60000), cts.Token),
    Race("Blue", rand.Next(1000, 60000), cts.Token),
    Race("Green", rand.Next(1000, 60000), cts.Token)
  };
  Console.Write("And they're off... Press [Enter] to cancel.");
  // See the "Gotcha Alert" section above for the implementation of this
  // function
  var inputTask = ActuallyReadLineAsync(cts.Token);

  var completedTask = await Task.WhenAny(raceTasks.Append(inputTask));
  var result = await completedTask;
  // Immediately cancel any tasks that are still pending (waiting for input or a delay)
  cts.Cancel();
  switch (result) {
    case "Red":
    case "Green":
    case "Blue":
      Console.WriteLine();
      Console.WriteLine($"And he winner is: {result}!");
      break;
    case "":
      Console.WriteLine("And it's a draw!");
      break;
    default:
      throw new NotSupportedException();
  }

  // Make sure all tasks have completed
  try {
    await Task.WhenAll(raceTasks.Append(inputTask));
  } catch (TaskCanceledException) {
    // This is expected
  }
}

public static async Task<String> Race(
  String competitor,
  Int32 delay,
  CancellationToken cancellationToken) {
  
  Console.WriteLine($"{competitor} Started (Delay: {delay}ms)");
  await Task.Delay(delay, cancellationToken);
  return competitor;
}
```
</details>
