# Thread Safety

Because threads share the same heap memory it is important to consider the behavior of those threads when they are simultaneously using the same data. There are essentially two types of issues that arise when implementing multi-threaded algorithms: 1) simultaneous operations occuring where one of the operation overwrites or undoes some part of the other resulting in an invalid or unexpected state, 2) deadlocks, where two simultaneous operations both become blocked waiting for the other to complete.

When writing multi-threaded code it is important to remember that almost no operations that happen in a computer are innately "atomic." That is to say that even the simplest operation might be interrupted mid way through and another operation executed in the iterrum. Take the operation of incrementing a number stored in memory. This operation actually involves loading that number into a register, then performing the increment operation on the CPU, then storing the result back into memory. If, after the value in memory is loaded into the register, the current thread is iterrupted and another thread executes, that thread could write a new value to the location in memory. The original thread would then overwrite the new value with its version.

Some simple operations can be made atomic with special instructions to the CPU that do things like compare and swap to ensure that the current value is what is expected before it is replaced. More complex operations require locking to ensure that only one thread is accessing a particular resource at a time.

## Atomic Operations

The [Interlocked class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked?view=net-8.0) provides a number of static methods that can be used to make atomic updates to primitive values that may be more efficient than using locks or other critical sections. These operations include [`Increment`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked.increment?view=net-8.0), [`Add`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked.add?view=net-8.0), and [`CompareExchange`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked.compareexchange?view=net-8.0). These methods all take one `ref` argument which is the value to be modified.

## Locks, Critical Sections, and Mutual Exclusion

Locks, critical sections, and mutaual exclusion are all essentially different terms for the same thing: writing blocks of code that can only be executed by a single thread at a time, preventing concurrent access to the same piece of data that might result in an unexpected state.

## Synchronization Primitives and Async Code

The `await` keyword cannot be used with the `lock` keyword, however other sychronization primitives such as `Mutex` _can_ be used from `async` code. This works, even for re-entrant synchronization objects (those that allow the same code to acquire the lock multiple times), because .Net tracks which "thread" currently holds a lock not with storage that is local to the thread, but in a context that is propogated across the `await` boundary. However, it is important to avoid certain less common constructs that break this mechanism (i.e. the `ContinueWith` method).

## Concurrent Collections