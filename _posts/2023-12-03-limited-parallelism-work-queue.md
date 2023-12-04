---
layout: post
title: Limited Parallelism Work Queue
categories: [.NET]
tags: [.NET]
comments: true
---

In the realm of asynchronous operations within C# applications, maintaining optimal performance often requires a delicate balance between execution and resource allocation. The need to prevent CPU oversubscription while managing numerous concurrent tasks is a common challenge faced by developers.

This blog post delves into a crucial strategy to navigate this challenge: the implementation of a limited parallelism work queue. Rather than allowing unchecked parallelism that might overwhelm system resources, employing a limited parallelism work queue offers a systematic approach to manage asynchronous tasks effectively.

The heart of implementing a work queue is the producer/consumer model. This can be well-represented by [Channels](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels){:target="_blank"}. If you're not familiar with channels, Stephen Toub has a [great introduction](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/){:target="_blank"}. Essentially a channel stores data from one or more producers to be consumed by one or more consumers.

In our case, the producers will be the components which enqueue work and the consumers will be the workers we spin up to process the work.

If desired, you can skip the explanation and go straight to the [Gist](https://gist.github.com/dfederm/445e971abf5340ab4b5b9ec8ef41a460){:target="_blank"}.

Let's start with creating the channel and the worker tasks. For now, we don't know exactly what kind of data we need to store, so we're just using `object`.

```cs
public sealed class WorkQueue
{
    private readonly Channel<object> _channel;
    private readonly Task[] _workerTasks;

    public WorkQueue(int parallelism)
    {
        _channel = Channel.CreateUnbounded<object>();

        // Create a bunch of worker tasks to process the work.
        _workerTasks = new Task[parallelism];
        for (int i = 0; i < _workerTasks.Length; i++)
        {
            _workerTasks[i] = Task.Run(
                async () =>
                {
                    await foreach (object context in _channel.Reader.ReadAllAsync())
                    {
                        // TODO: Process work
                    }
                });
        }
    }
}
```

This simply creates the `Channel` and creates multiple worker tasks whick simply continuously try reading from the channel. `Channel.Reader.ReadAllAsync` will yield until there is data to read, so it's not blocking any threads.

Now we need the producer side of things. For this initial implementation with `Task` and not `Task<T>`, so we know the return type for the method needs to be a `Task`. The caller needs to provide a factory for actually performaing the work, so the parameter can be a `Func<Task>`. This leads us to the following signature:

```cs
    public async Task EnqueueWorkAsync(Func<Task> taskFunc);
```

As we want to manage the parallelism of the work, we cannot call the `Func<Task>` to get the `Task`, as that would start execution of the task. The way to return a `Task` when we don't have one is to use `TaskCompletionSource`. This allows us to return a `Task` which we can later complete with a result, cancellation, or exception, based on what happens with the provided work.

We also know we need to write _something_ to the channel, but we still don't know what yet, so let's continue to use `object`.

```cs
    public async Task EnqueueWorkAsync(Func<Task> taskFunc)
    {
        TaskCompletionSource taskCompletionSource = new();
        object context = new();
        await _channel.Writer.WriteAsync(context);
        await taskCompletionSource.Task;
    }
```

Now that we have the channel reader and writer usages, we can figure out what we actually need to store in the channel. The caller provided a `Func<Task>` to perform the work, and we need to capture the `TaskCompletionSource` so we can complete the `Task` we returned to the caller. So let's define the context as a simple record struct with those two members:

```cs
private readonly record struct WorkContext(Func<Task> TaskFunc, TaskCompletionSource TaskCompletionSource);
```

The `Channel<object>` should be updated to use `WorkContext` instead, as the reader and writer call sites should also be adjusted. We now have the following:

```cs
public sealed class WorkQueue
{
    private readonly Channel<WorkContext> _channel;
    private readonly Task[] _workerTasks;

    private readonly record struct WorkContext(Func<Task> TaskFunc, TaskCompletionSource TaskCompletionSource);

    public WorkQueue(int parallelism)
    {
        _channel = Channel.CreateUnbounded<WorkContext>();

        // Create a bunch of worker tasks to process the work.
        _workerTasks = new Task[parallelism];
        for (int i = 0; i < _workerTasks.Length; i++)
        {
            _workerTasks[i] = Task.Run(
                async () =>
                {
                    await foreach (WorkContext context in _channel.Reader.ReadAllAsync())
                    {
                        // TODO: Process work
                    }
                });
        }
    }

    public async Task EnqueueWorkAsync(Func<Task> taskFunc)
    {
        TaskCompletionSource taskCompletionSource = new();
        WorkContext context = new(taskFunc, taskCompletionSource);
        await _channel.Writer.WriteAsync(context);
        await taskCompletionSource.Task;
    }
```

Now we need to actually process the work. This involes executing the provided `Func<Task>` and handling the result appropriately. We will simply invoke the `Func` and await the resulting `Task`. Whether that `Task` completed successfully, threw an exception, or was cancelled, we should pass through to the `Task` we returned to the caller who queued up the work.

```cs
    private static async Task ProcessWorkAsync(WorkContext context)
    {
        try
        {
            await context.TaskFunc();
            context.TaskCompletionSource.TrySetResult();
        }
        catch (OperationCanceledException ex)
        {
            context.TaskCompletionSource.TrySetCanceled(ex.CancellationToken);
        }
        catch (Exception ex)
        {
            context.TaskCompletionSource.TrySetException(ex);
        }
    }
```

Finally, we need to handle shutting down the work queue. This is done by completeing the channel and waiting for the worker tasks to drain. Calling `Channel.Writer.Complete` will disallow additional items from being written, and as a side-effect cause `Channel.Reader.ReadAllAsync` enumerable to stop awaiting more results and complete. This in turn allows our worker tasks to complete.

For convenience, we will make `WorkQueue : IAsyncDisposable` so the `WorkQueue` can simply be disposed to shut it down.

```cs
    public async ValueTask DisposeAsync()
    {
        _channel.Writer.Complete();
        await _channel.Reader.Completion;
        await Task.WhenAll(_workerTasks);
    }
```

On thing we've left out is cancellation, both for executing work to be cancelled when the work queue is shut down, and for allowing the caller enqueing a work item to cancel that work item.

To address this, a `CancellationToken` should be provided by the caller enqueueing a work item. Additionally, the `WorkQueue` itself will need to manage a `CancellationTokenSource` which it cancels on `DisposeAsync`. Finally, when a work item is enqueued, the two cancellation tokens need to be linked and provided to the work item so it can properly cancel when either the caller who enqueued the work item cancels, or if the work queue is being shut down entirely. Putting all that together:

```cs
public sealed class WorkQueue : IAsyncDisposable
{
    private readonly CancellationTokenSource _cancellationTokenSource;
    private readonly Channel<WorkContext> _channel;
    private readonly Task[] _workerTasks;

    private readonly record struct WorkContext(Func<CancellationToken, Task> TaskFunc, TaskCompletionSource TaskCompletionSource, CancellationToken CancellationToken);

    public WorkQueue()
        : this (Environment.ProcessorCount)
    {
    }

    public WorkQueue(int parallelism)
    {
        _cancellationTokenSource = new CancellationTokenSource();
        _channel = Channel.CreateUnbounded<WorkContext>();

        // Create a bunch of worker tasks to process the work.
        _workerTasks = new Task[parallelism];
        for (int i = 0; i < _workerTasks.Length; i++)
        {
            _workerTasks[i] = Task.Run(
                async () =>
                {
                    // Not passing using the cancellation token here as we need to drain the entire channel to ensure we don't leave dangling Tasks.
                    await foreach (WorkContext context in _channel.Reader.ReadAllAsync())
                    {
                        await ProcessWorkAsync(context);
                    }
                });
        }
    }

    public async Task EnqueueWorkAsync(Func<CancellationToken, Task> taskFunc, CancellationToken cancellationToken = default)
    {
        cancellationToken.ThrowIfCancellationRequested();
        TaskCompletionSource taskCompletionSource = new();
        CancellationToken linkedToken = cancellationToken.CanBeCanceled
            ? CancellationTokenSource.CreateLinkedTokenSource(cancellationToken, _cancellationTokenSource.Token).Token
            : _cancellationTokenSource.Token;
        WorkContext context = new(taskFunc, taskCompletionSource, linkedToken);
        await _channel.Writer.WriteAsync(context, linkedToken);
        await taskCompletionSource.Task;
    }

    public async ValueTask DisposeAsync()
    {
        await _cancellationTokenSource.CancelAsync();
        _channel.Writer.Complete();
        await _channel.Reader.Completion;
        await Task.WhenAll(_workerTasks);
        _cancellationTokenSource.Dispose();
    }

    private static async Task ProcessWorkAsync(WorkContext context)
    {
        if (context.CancellationToken.IsCancellationRequested)
        {
            context.TaskCompletionSource.TrySetCanceled(context.CancellationToken);
            return;
        }

        try
        {
            await context.TaskFunc(context.CancellationToken);
            context.TaskCompletionSource.TrySetResult();
        }
        catch (OperationCanceledException ex)
        {
            context.TaskCompletionSource.TrySetCanceled(ex.CancellationToken);
        }
        catch (Exception ex)
        {
            context.TaskCompletionSource.TrySetException(ex);
        }
    }
}
```

If the result of the work is required, this approach may be awkward since the provided `Func<CancellationToken, Task>` would need to have side-effects. For example, something like the following:

```cs
string input = // ...
int result = -1;
await queue.EnqueueWorkAsync(async ct => result = await ProcessAsync(input, ct), cancellationToken));

// Do something with the result here
// ...
```

An alternate approach would be to have the `WorkQueue` take the processing function and then the `EnqueueWorkAsync` method could return the result directly. This requires the work queue to process inputs of the same type and in the same way, but can make the calling pattern more elegant:

```cs
string input = // ...
int result = await queue.EnqueueWorkAsync(input, cancellationToken);

// Do something with the result here
// ...
```

The change to the implementation is straightforward. `WorkQueue` becomes the generic `WorkQueue<TInput, TResult>` and the `Func<CancellationToken, Task>` becomes a `Func<TInput, CancellationToken, Task<TResult>>` and can move from `EnqueueWorkAsync` to the constructor.

```cs
public sealed class WorkQueue<TInput, TResult> : IAsyncDisposable
{
    private readonly Func<TInput, CancellationToken, Task<TResult>> _processFunc;
    private readonly CancellationTokenSource _cancellationTokenSource;
    private readonly Channel<WorkContext> _channel;
    private readonly Task[] _workerTasks;

    private readonly record struct WorkContext(TInput Input, TaskCompletionSource<TResult> TaskCompletionSource, CancellationToken CancellationToken);

    public WorkQueue(Func<TInput, CancellationToken, Task<TResult>> processFunc)
        : this(processFunc, Environment.ProcessorCount)
    {
    }

    public WorkQueue(Func<TInput, CancellationToken, Task<TResult>> processFunc, int parallelism)
    {
        _processFunc = processFunc;
        _cancellationTokenSource = new CancellationTokenSource();
        _channel = Channel.CreateUnbounded<WorkContext>();

        // Create a bunch of worker tasks to process the work.
        _workerTasks = new Task[parallelism];
        for (int i = 0; i < _workerTasks.Length; i++)
        {
            _workerTasks[i] = Task.Run(
                async () =>
                {
                    // Not passing using the cancellation token here as we need to drain the entire channel to ensure we don't leave dangling Tasks.
                    await foreach (WorkContext context in _channel.Reader.ReadAllAsync())
                    {
                        await ProcessWorkAsync(context, _cancellationTokenSource.Token);
                    }
                });
        }
    }

    public async Task<TResult> EnqueueWorkAsync(TInput input, CancellationToken cancellationToken = default)
    {
        cancellationToken.ThrowIfCancellationRequested();
        TaskCompletionSource<TResult> taskCompletionSource = new();
        CancellationToken linkedToken = cancellationToken.CanBeCanceled
            ? CancellationTokenSource.CreateLinkedTokenSource(cancellationToken, _cancellationTokenSource.Token).Token
            : _cancellationTokenSource.Token;
        WorkContext context = new(input, taskCompletionSource, linkedToken);
        await _channel.Writer.WriteAsync(context, linkedToken);
        return await taskCompletionSource.Task;
    }

    public async ValueTask DisposeAsync()
    {
        await _cancellationTokenSource.CancelAsync();
        _channel.Writer.Complete();
        await _channel.Reader.Completion;
        await Task.WhenAll(_workerTasks);
        _cancellationTokenSource.Dispose();
    }

    private async Task ProcessWorkAsync(WorkContext context, CancellationToken cancellationToken)
    {
        if (cancellationToken.IsCancellationRequested)
        {
            context.TaskCompletionSource.TrySetCanceled(cancellationToken);
            return;
        }

        try
        {
            TResult result = await _processFunc(context.Input, cancellationToken);
            context.TaskCompletionSource.TrySetResult(result);
        }
        catch (OperationCanceledException ex)
        {
            context.TaskCompletionSource.TrySetCanceled(ex.CancellationToken);
        }
        catch (Exception ex)
        {
            context.TaskCompletionSource.TrySetException(ex);
        }
    }
}
```

This blog post shows two alternate approaches for implementing a limited parallelism work queue to manage many tasks while avoiding overscheduling. These two implementations are best suited to different usage patterns and can be further customized or optimized for your specific use-cases.