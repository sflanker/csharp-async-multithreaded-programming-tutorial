# Threads

Threads are one of, if not the, most fundamental concept when it comes to performing multiple operations in parallel in a computer program. Operating systems have built in support for creating multiple threads within a processes. However many programming language runtimes, including the .Net Common Language Runtime used by C#, implement their own thread concept because operating system threads are a finite resource, the behavior of native threads may differ between operating systems, and the creation of new threads often comes with some overhead.

Each thread within a process has its own instruction pointer and stack, however all threads share heap memory. In the CLR each thread is managed via an instance of the [Thread class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.thread?view=net-8.0). A new thread can be created by instantiating a new Thread instance with a delegate representing the entry point of the thread, and then calling the [Thread.Start](https://learn.microsoft.com/en-us/dotnet/api/system.threading.thread.start?view=net-8.0) method. The delegate that is used as the entry point may either be an [ThreadStart](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadstart?view=net-8.0) (`void ThreadStart()`) or a [ParameterizedThreadStart](https://learn.microsoft.com/en-us/dotnet/api/system.threading.parameterizedthreadstart?view=net-8.0) (`void ParameterizedThreadStart(Object? state)`), and the thread should be started with the corresponding overload of the `Start` method.

## Example

```csharp
public static void Main() {
  var backgroundThread = new Thread(CalculatePI);
  backgroundThread.Start();
  var (left, top) = Console.GetCursorPosition();
  while (!done) {
    Console.SetCursorPosition(left, top);
    Console.Write(pi.ToString());
    Thread.Sleep(100);
  }
  backgroundThread.Join();
}

private static Boolean done = false;
private static Decimal pi = 4m;
private static void CalculatePI() {
  for (var i = 0; i < 50000000; i++) {
    pi += (i % 2 == 0 ? -1 : 1) * 4m / (2 * i + 3);
  }

  done = true;
}
```

## Caution: Unhandled Exceptions

Unhandled exceptions in any thread created and started via the Thread class will cause the program to exit. When manually creating background threads it is important to properly handle any exceptions that should not terminate the process.

## Passing State to a Thread

There are several different mechanisms by which state can be passed to the new thread.

### Via ParameterizedThreadStart

Using a `ParameterizedThreadStart` delegate enables you to pass a single `Object` argument to the entry point of your thread. The downside of this approach is that there is no type safety and you have to pack your state into a single instance.

<details>
  <summary><strong>Example</strong></summary>

```csharp
public static void Main() {
  var backgroundThread = new Thread(CalculatePI);
  backgroundThread.Start(50000000);
  var (left, top) = Console.GetCursorPosition();
  while (!done) {
    Console.SetCursorPosition(left, top);
    Console.Write(pi.ToString());
    Thread.Sleep(100);
  }
  backgroundThread.Join();
}

private static Boolean done = false;
private static Decimal pi = 4m;
private static void CalculatePI(Object iterationObj) {
  var iterations = (Int32)iterationObj;
  for (var i = 0; i < iterations; i++) {
    pi += (i % 2 == 0 ? -1 : 1) * 4m / (2 * i + 3);
  }

  done = true;
}
```
</details>

### Using an Anonymous Function (Closure)

This is a great way to pass state to a thread. It is easy to implement and has very little overhead. However be cognizant of _what_ state you include. One possible mistake would be capturing an `IDisposable` that is disposed in the parent scope.

<details>
  <summary><strong>Example</strong></summary>

```csharp
public static async Task Main() {
  var backgroundThread = new Thread(() => CalculatePI(50000000));
  backgroundThread.Start();
  var (left, top) = Console.GetCursorPosition();
  while (!done) {
    Console.SetCursorPosition(left, top);
    Console.Write(pi.ToString());
    Thread.Sleep(100);
  }
  backgroundThread.Join();
}

private static Boolean done = false;
private static Decimal pi = 4m;
private static void CalculatePI(Int32 iterations) {
  for (var i = 0; i < iterations; i++) {
    pi += (i % 2 == 0 ? -1 : 1) * 4m / (2 * i + 3);
  }

  done = true;
}
```
</details>

### Using an Instance Method on a Stateful Object

This approach requires some additional boilerplate code, however it can help to encapsulate logic when the state being passed is complex.

<details>
  <summary><strong>Example</strong></summary>

```csharp
class Program {
  public static void Main() {
    var calculator = new PICalculator(50000000);
    var backgroundThread = new Thread(calculator.CalculatePI);
    backgroundThread.Start();
    var (left, top) = Console.GetCursorPosition();
    while (!calculator.done) {
      Console.SetCursorPosition(left, top);
      Console.Write(calculator.pi.ToString());
      Thread.Sleep(100);
    }
    backgroundThread.Join();
  }
}

class PICalculator {
  public readonly Int32 iterations;
  public Boolean done;
  public Decimal pi = 4m;

  public PICalculator(Int32 iterations) {
    this.iterations = iterations;
  }

  public void CalculatePI() {
    for (var i = 0; i < this.iterations; i++) {
      this.pi += (i % 2 == 0 ? -1 : 1) * 4m / (2 * i + 3);
    }

    this.done = true;
  }
}
```

> Note: in this example it is important that `PICalculator` be a _class_ and not a _struct_. Any time you capture a struct either via an anonymous function (closure), or via an instance method delegate, that struct is copied _by value_ to the heap as part of the delegate or closure. So the original copy that the parent method has _on its stack_ will not be mutated.
</details>