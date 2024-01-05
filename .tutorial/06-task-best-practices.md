# Task Best Practices

When using Tasks and writing `async` functions it is important to follow a number of best practices.

There have been many articles and resources written on this topic, but by far the most concise that I have found is [this obscure markdown file](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md) from David Fowler.

Additional resources including [the Async/Await FAQ](https://devblogs.microsoft.com/pfxteam/asyncawait-faq/) as well as numerous other articles posted by the [pfxteam](https://devblogs.microsoft.com/pfxteam/) and [Stephen Toub](https://devblogs.microsoft.com/pfxteam/author/toub/).

## Highligts

 * Never write `async void` methods, always return a `Task`.
 * Avoid `Result` and `Wait` methods that block syncronously until a task completes.
 * Always `await` Tasks somewhere, avoid creating a task that you never `await`.

## Note: Unhandled Exception Behavior

Unlike Threads and Thread Pool work items, when an exception is raised in a task it does not cause the process to exit. This is one of the reasons it is important to `await` each task, because that _will_ raise any exceptions so that they can be appropriately handled by the calling code.

## ConfigureAwait

One contentious topic regarding async programming in .Net is the appropriate use of `ConfigureAwait`, with some recommending it be used prolifically, and most recommending it be used on each await statement in _shared public libraries_ only.

The rhyme and reason behind the use of `ConfigureAwait` is an advanced topic which I won't get into here. However, I will say that I think many people are too quick to recommend its use, and it is _not needed_ in a vast majority of cases. For now I will simply leave you with some [further reading](https://www.gabescode.com/dotnet/2022/02/04/dont-use-configureawait.html) on the topic, but I will come back to this in more detail later in this tutorial.