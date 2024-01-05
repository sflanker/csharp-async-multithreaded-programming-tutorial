# Task Helper Functions

## ☢️ Danger Zone ☢️

There are a number of properties and methods on `Task` or `Task<T>` instances that should generally be avoided.

### `Task<T>.Result`

Accessing the `Result` property of a `Task<T>` while **synchronously** block the current thread until the Task completes or raises an exception. This is almost never what you want to do except in the case where you are working with a `Task` from synchronous code that you cannot make `async`, and even then you have to take special steps to be sure you don't create a deadlock.

Instead of using `Task<T>.Result`, use the `await` keyword which has the same effect but does so _asynchronously_.

### `Task.RunSynchronously`

This method is actually a low-level implementation detail of the `Task` system in .Net, it can only be used on tasks in the `Created` state, so it is not a good general purpose way to run `Task`s from a non-async context.

### `Task.Wait*` (except `WaitAsync`)

Similar to the `Result` property mentioned above, these methods block synchronously, which is generally problematic and not waht you want to do. The one exception is the `WaitAsync` method which can be used to `await` a task but with your own CancellationToken, or an explicit timeout.

### `ContinueWith`

Technically `ContinueWith` can be used to queue some synchronous or asynchronous code to execute after a task has completed. However, usage of this function is generally discouraged, and it is better to implement any code that should run after a Task completes in an async function that awaits the task. The reasons for this are fairly subtle and relate to `SynchronizationContext`s. Usage of `ContinueWith` should be limitted to simple, side-effect-free transformations of a Task's result.

## `WhenAll` and `WhenAny`

These static methods on the `Task` class are very helpful for running multiple tasks in parallel when you want to asynchronously wait for either one or all of the Tasks to complete. The `WhenAll` method helpfully returns the list of results when awaited, and `WhenAny` returns the task that completed first (since it is completed, you can immediately await that task and get the actual result).

## `Task.Unwrap`

The `Task.Unwrap` extension method can be used to unwrap an inner `Task<T>` given `Task<Task<T>>` without any synchronous blocking operations. This is useful when awaiting the result from the task returned from a call to `Task.WhenAny`.

## `Task.Yield`

The `Task.Yield` function can be used to ensure that any code after the `Task.Yield` is actually run asynchronously. Otherwise when a async function is invoked, all of the code up to the first `await` _may_ be run syhcronously in the calling thread.

## `Task.Delay`

A `Task.Delay` can be used to implement timeout logic for another operation, or to add a pause to code performing other async operations (for example to pause before retrying a network operation). However, it is important to pass a cancellation token to calls to `Task.Delay` because the timers used to implement `Task.Delay` are a finite resource, and if creating long timeouts and not cancelling them can lead to resource starvation.