# Exception Handling

## Faulted Tasks, `await`, and the `Task.Exception` property

When an exception is raised by async code the Task that it returned will be in the Faulted state. When a Task in the faulted state is awaited the exception will be re-raised, preserving the original stack trace. An exception would also be raised by a synchronously blocking method or property such as `Task.Wait()` or `Task.Result`, but it would be wrapped in a `AggregateException`.

Tasks that represent multiple other tasks running in parallel may be in a Faulted state with multiple exceptions (when using `Task.WhenAll` or `Parallel.ForEachAsync` for example). When such a `Task` is awaited only one of the exceptions will actually be raised. If you need to examine all exceptions you can inspect the Task's `Exception` property (which is of type `AggregateException`).

If a `Task` is never awaited, any exceptions throw by that task will have no affect on the program (unlike exceptions raised in normal threads, which would crash the process).

<details>
  <summary><h3>Example</h3></summary>

```csharp
public static async Task Main() {
  try {
    await ErrorExample(500);
  } catch (Exception e) {
    Console.WriteLine($"{e.GetType().Name} throw by await");
  }

  try  {
    Console.WriteLine(ErrorExample(500).Result.ToString());
  } catch (Exception e) {
    Console.WriteLine($"{e.GetType().Name} throw by Result");
  }

  Task whenAllTask = null;
  try {
    await (whenAllTask = Task.WhenAll(
      ErrorExample(500),
      ErrorExample(501)
    ));
  } catch (Exception e) {
    Console.WriteLine($"{e.GetType().Name} throw by await WhenAll");

    Console.WriteLine($"But there were {whenAllTask.Exception.InnerExceptions.Count} exceptions in the task state");
  }

  try {
    await Parallel.ForEachAsync(
      new[] { 203, 500, 501 },
      async (val, cancellation) => {
        Console.WriteLine(await ErrorExample(val));
      }
    );
  } catch (Exception e) {
    Console.WriteLine($"{e.GetType().Name} throw by await Parallel.ForEach");
  }
}

private static async Task<String> ErrorExample(Int32 delay) {
  await Task.Delay(delay);
  if (delay % 2 == 0) {
    throw new ArgumentException("Vim is better.");
  } else if (delay % 3 == 0) {
    throw new ArgumentException("Emacs is better.");
  } else {
    return "I don't think that's what ArgumentException is for.";
  }
}
```
</details>

## Synchronously Raised Exception

It is important to be aware that just because a method returns a Task, that doesn't mean it can't also throw an exception synchronously. Some methods validate inputs synchronously and then return a Task that is the result of calling another method without using `await` and that could result in an exception being throw synchronously.

<details>
  <summary><h3>Example</h3></summary>
  
```csharp
public static async Task Main() {
  Task exampleTask;
  try {
    exampleTask = ErrorExample(100, throwBefore: true);
  } catch (Exception e) {
    Console.WriteLine($"Caught Exception {e.GetType().Name}: {e.Message}");
    exampleTask = ErrorExample(100, throwAfter: true);
  }

  try {
    Console.WriteLine("Awaiting Task");
    await exampleTask;
  } catch (Exception e) {
    Console.WriteLine($"Caught Exception {e.GetType().Name}: {e.Message}");
  }
}

private static Task ErrorExample(
  Int32 delay,
  Boolean throwBefore = false,
  Boolean throwAfter = false) {

  Console.WriteLine($"Executing on Thread {Thread.CurrentThread.ManagedThreadId}");

  if (throwBefore) {
    throw new ArgumentException("before");
  }

  return DelayAndMaybeThrow();

  async Task DelayAndMaybeThrow() {
    await Task.Delay(delay);

    Console.WriteLine($"Executing on Thread {Thread.CurrentThread.ManagedThreadId}");

    if (throwAfter) {
      throw new ArgumentException("after");
    }
  }
}
```
</details>
