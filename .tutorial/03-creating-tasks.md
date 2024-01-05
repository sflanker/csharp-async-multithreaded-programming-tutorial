# Creating Tasks in C#

Modern asynchronous programming in C# revolves primarily around the [`Task`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task) class. There are several different ways to create a `Task`. The simplest way is probably to simply call an existing function that returns a `Task` for testing purposes the static [`Task.Delay`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.delay) method is very useful. The `main.cs` file in this tutorial contains just about the simplest asynchronous program you can write (Hello World William Shatner Edition):

```csharp
using System;
using System.Threading.Tasks;

Console.WriteLine("Hello...");

await Task.Delay(1000);

Console.WriteLine("World");
```

You can also create a task by writing a function using the `async` keyword:

```csharp
using System;
using System.Threading.Tasks;

class Program {
  public static async Task Main() {
    Console.Write("Hello");
    
    await DoSomethingAsync(4);
    
    Console.WriteLine("World");
  }

  public static async Task DoSomethingAsync(Int32 count) {
    for (var i = 0; i < count; i++) {
      await Task.Delay(500);
      Console.Write(".");
    }
        
    await Task.Delay(500);
  
    Console.WriteLine();
  }
}
```

Note that you don't have to do anything explicit for a `Task` to begin executing in most cases (although to some extent this depends on how it is created). For example, consider the following code:

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program {
  static Boolean completed = false;
  
  public static async Task Main() {
    Console.Write("Hello");
    
    var task = DoSomethingAsync(4);

    // Even though we never explicitly start or await the Task, it is automatically running
    while (!completed) {
      Thread.Sleep(100);
    }
    
    Console.WriteLine("World");
  }

  static async Task DoSomethingAsync(Int32 count) {
    for (var i = 0; i < count; i++) {
      await Task.Delay(500);
      Console.Write(".");
    }
        
    await Task.Delay(500);
  
    Console.WriteLine();
    completed = true;
  }
}
```

It is also possible to create a `Task` entirely from code that does not use `await`. There are several different ways to do this:

| Method | Behavior |
|--------|----------|
| `Task.Run(Action) `| Creates _and starts_ a new Task that executes the specified delegate. |
| `new Task(Action)` | Creates a Task object that executes the specified delegate, _but does not start it._ |
| `Task.Factory.StartNew(Action)` | Same as `Task.Run` which is generally favored. |

All of these methods have generic alternatives to create a Task that returns a value, and have overloads that take various options to control how the Task is scheduled or executed. Typically you do not have to worry about these.

The most common use case for code the be executed asynchronously using a `Task` is when it needs to perform IO such as reading from a file or making a network request. Generally a `Task` would only be created for otherwise synchronous code if that code is CPU intensive. Tasks for CPU intensive code can be leveraged to perform multiple operations in parallel (see: Multithreading), continue to process user inputs or network requests while a CPU intensive operation is running, or to implement a timeout for the CPU intensive operation.

> Note: while it is possible to use a Task to execute CPU bound work, if a Task is expected to be long running special consideration needs to be taken, because the default behavior is to execute the Task on the thread pool and that thread will be utilized until the first `await`, which could lead to thread pool starvation. 

Given this potentially CPU intensive function, can you turn it into a Task that can be executed in the background while the program is still responsive (demonstrated by displaying some output, you can use `DoSomethingAsync` from the previous examples):

```csharp
// Note: this only takes an appreciable amount of time for iterations >= 10_000_000
static Decimal DoSomethingCpuIntensive(Int32 iterations) {
  var pi = 4m;
  for (var i = 0; i < iterations; i++) {
    pi += (i % 2 == 0 ? -1 : 1) * 4m / (2 * i + 3);
  }
  
  return pi;
}
```

<details>
  <summary>
    Solution
  </summary>
  
```csharp
using System;
using System.Threading.Tasks;

class Program {
  static Boolean completed = false;
  
  public static async Task Main() {
    Console.WriteLine("Starting");

    // CPU Intensive task starts executing automatically
    var task = DoSomethingCpuIntensive(10000000);

    Console.Write("CPU Bound Task Created");

    // Do something IO related while computations are being executed
    var spinner = DoSomethingAsync(10);

    // Wait for the CPU bound task to complete
    var result = await task;

    Console.WriteLine($"Done: {result}");
    
    await spinner;
  }

  static Task<Decimal> DoSomethingCpuIntensive(Int32 iterations) {
    return Task.Run(() => {
      var pi = 4m;
      for (var i = 0; i < iterations; i++) {
        pi += (i % 2 == 0 ? -1 : 1) * 4m / (2 * i + 3);
      }
      
      return pi;
    });
  }

  static async Task DoSomethingAsync(Int32 count) {
    for (var i = 0; i < count; i++) {
      await Task.Delay(500);
      Console.Write(".");
    }
        
    await Task.Delay(500);
  
    Console.WriteLine();
    completed = true;
  }
}
```
</details>