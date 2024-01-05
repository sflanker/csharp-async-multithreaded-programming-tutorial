# Thread Pool

Even in scenarios where a _managed thread_ does not map 1:1 to an _operating system thread_, threads are still a finite resource with overhead to spin up. For this reason, programs that tend to initialize numerous, short lived threads _should_ make use of a thread pool. An example of this would be an HTTP server which uses a thread to handle is request it receives.

The thread pool is a low level mechanism and there exist a number of higher level abstractions that utilize it including Tasks, timers, and asynchronous callback operations. In most cases application developers can forego any direct use of the thread pool and use these higher level abstractions.

> Note: Because the thread pool is a global resource within each .Net process it is important not to block thread pool threads except on specific operations that actually return the thread to the thread pool and may continue on a different thread.

> Warning: As with regular managed threads, any unhandled exceptions raised by a thread pool thread will terminate the entire process. So it is important to include appropriate exception handling in your thread pool work items.

## Example

```csharp
public static void Main() {
  String line;
  while (!String.IsNullOrEmpty(line = Console.ReadLine())) {
    ThreadPool.QueueUserWorkItem<String>(
      str => {
        var f = GetFactors(str);
        // Console.WriteLine is thread safe
        Console.WriteLine($"{str} => {String.Join(", ", f)}");
      },
      line,
      preferLocal: false
    );
  }
}

// This is not actually a good function to run on the thread pool, since for
// some inputs it can take a very long time. But it illustrates computation
// happening in parallel. For example input 5659968623 and then immediately
// input 5659968624 and see which completes first.
private static Int64[] GetFactors(String str) {
  var factors = new Int64[0];
  if (Int64.TryParse(str, out var num)) {
    for (var f = 2; f < num; f++) {
      while (num % f == 0) {
        factors = factors.Append(f).ToArray();
        num /= f;
      }
    }
    if (num > 1) {
      factors = factors.Append(num).ToArray();
    }
  }
  return factors;
}
```