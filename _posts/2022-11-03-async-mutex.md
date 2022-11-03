---
layout: post
title: Async Mutex
categories: [.NET]
tags: [.NET]
comments: true
---
The [`Mutex`](https://learn.microsoft.com/en-us/dotnet/standard/threading/mutexes){:target="_blank"} class in .NET helps manage exclusive access to a resource. When given a name, this can even be done across processes which can be extremely handy.

Though if you've ever used a `Mutex` you may have found that it cannot be used in conjunction with `async`/`await`. More specifically, from the documentation:

> Mutexes have thread affinity; that is, the mutex can be released only by the thread that owns it.

This can make the `Mutex` class hard to use at times and may require use of ugliness like `GetAwaiter().GetResult()`.

For in-process synchronization, [`SemaphoreSlim`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim){:target="_blank"} can be a good choice as it has a [`WaitAsync()`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim.waitasync){:target="_blank"} method. However semaphores aren't ideal for managing exclusive access (`new SemaphoreSlim(1)` works but is less clear) and do not support system-wide synchronization eg. `new Mutex(initiallyOwned: false, @"Global\MyMutex")`.

Below I'll explain how to implement an async mutex, but the full code can be found at the bottom or in the [Gist](https://gist.github.com/dfederm/35c729f6218834b764fa04c219181e4e){:target="_blank"}.

## How to use a Mutex

First, some background on how to properly use a `Mutex`. The simplest example is:

```cs
// Create the named system-wide mutex
using Mutex mutex = new(false, @"Global\MyMutex");

// Acquire the Mutex
mutex.WaitOne();

// Do work...

// Release the Mutex
mutex.ReleaseMutex();
```

As `Mutex` derives from [`WaitHandle`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.waithandle){:target="_blank"}, `WaitOne()` is the mechanism to acquire it.

However, if a `Mutex` is not properly released when a thread holding it exits, the `WaitOne()` will throw a [`AbandonedMutexException`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.abandonedmutexexception). The reason for this is explained as:

> An abandoned mutex often indicates a serious error in the code. When a thread exits without releasing the mutex, the data structures protected by the mutex might not be in a consistent state. The next thread to request ownership of the mutex can handle this exception and proceed, if the integrity of the data structures can be verified.

So the next thread to acquire the `Mutex` is responsible for verifying data integrity, if applicable. Note that a thread can exit without properly releasing the `Mutex` if the user kills the process, so `AbandonedMutexException` should always be caught when trying to acquire a `Mutex`.

With this our new example becomes:
```cs
// Create the named system-wide mutex
using Mutex mutex = new(false, @"Global\MyMutex");
try
{
    // Acquire the Mutex
    mutex.WaitOne();
}
catch (AbandonedMutexException)
{
    // Abandoned by another process, we acquired it.
}

// Do work...

// Release the Mutex
mutex.ReleaseMutex();
```

However, what if the work we want to do while holding the `Mutex` is async?

## `AsyncMutex`
First let's define what we want the shape of the class to look like. We want to be able to acquire and release the mutex asynchronously, so the following seems reasonable:

```cs
public sealed class AsyncMutex : IAsyncDisposable
{
    public AsyncMutex(string name);

    public Task AcquireAsync(CancellationToken cancellationToken);

    public Task ReleaseAsync();

    public ValueTask DisposeAsync();
}
```

And so the intended usage would look like:

```cs
// Create the named system-wide mutex
await using AsyncMutex mutex = new(@"Global\MyMutex");

// Acquire the Mutex
await mutex.AcquireAsync(cancellationToken);

// Do async work...

// Release the Mutex
await mutex.ReleaseAsync();
```

Now that we know what we want it to look like, we can start implementing.

## Acquiring
Because `Mutex` must be in a single thread, and because we want to return a `Task` so the mutex can be acquired async, we can start a new `Task` which uses the `Mutex` and return that.

```cs
public Task AcquireAsync()
{
    TaskCompletionSource taskCompletionSource = new();

    // Putting all mutex manipulation in its own task as it doesn't work in async contexts
    // Note: this task should not throw.
    Task.Factory.StartNew(
        state =>
        {
            try
            {
                using var mutex = new Mutex(false, _name);
                try
                {
                    // Acquire the Mutex
                    mutex.WaitOne();
                }
                catch (AbandonedMutexException)
                {
                    // Abandoned by another process, we acquired it.
                }

                taskCompletionSource.SetResult();

                // TODO: We need to release the mutex at some point
            }
            catch (Exception ex)
            {
                taskCompletionSource.TrySetException(ex);
            }
        }
        state: null,
        cancellationToken,
        TaskCreationOptions.LongRunning,
        TaskScheduler.Default);

    return taskCompletionSource.Task;
}
```

So now `AcquireAsync` returns a `Task` which doesn't complete until the `Mutex` is acquired.

## Releasing
At some point the code needs to release the `Mutex`. Because the mutex must be released in the same thread it was acquired in, it must be released in the `Task` which `AcquireAsync` started. However, we don't want to actually release the mutex until `ReleaseAsync` is called, so we need the `Task` to wait until that time.

To accomplish this, we need a `ManualResetEventSlim` which the `Task` can wait for a signal from, which `ReleaseAsync` will set.

```cs
private Task? _mutexTask;
private ManualResetEventSlim? _releaseEvent;

public Task AcquireAsync(CancellationToken cancellationToken)
{
    TaskCompletionSource taskCompletionSource = new();

    _releaseEvent = new ManualResetEventSlim();

    // Putting all mutex manipulation in its own task as it doesn't work in async contexts
    // Note: this task should not throw.
    _mutexTask = Task.Factory.StartNew(
        state =>
        {
            try
            {
                using var mutex = new Mutex(false, _name);
                try
                {
                    // Acquire the Mutex
                    mutex.WaitOne();
                }
                catch (AbandonedMutexException)
                {
                    // Abandoned by another process, we acquired it.
                }

                taskCompletionSource.SetResult();

                // Wait until the release call
                _releaseEvent.Wait();

                mutex.ReleaseMutex();
            }
            catch (Exception ex)
            {
                taskCompletionSource.TrySetException(ex);
            }
        },
        state: null,
        cancellationToken,
        TaskCreationOptions.LongRunning,
        TaskScheduler.Default);

    return taskCompletionSource.Task;
}

public async Task ReleaseAsync()
{
    _releaseEvent?.Set();

    if (_mutexTask != null)
    {
        await _mutexTask;
    }
}
```

Now the `Task` will acquire the `Mutex`, then wait for a signal from the `ReleaseAsync` method to release the mutex.

Additionally, the `ReleaseAsync` waits for the `Task` to finish to ensure its `Task` will not complete until the mutex is released.

## Cancellation
The caller may not want to wait forever for the mutex acquisition, so we need cancellation support. This is fairly straightforward since `Mutex` is a `WaitHandle`, and [`CancellationToken`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken){:target="_blank"} has a [`WaitHandle`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken.waithandle){:target="_blank"} property, so we can use `WaitHandle.WaitAny()`

```cs
public Task AcquireAsync(CancellationToken cancellationToken)
{
    cancellationToken.ThrowIfCancellationRequested();

    TaskCompletionSource taskCompletionSource = new();

    _releaseEvent = new ManualResetEventSlim();

    // Putting all mutex manipulation in its own task as it doesn't work in async contexts
    // Note: this task should not throw.
    _mutexTask = Task.Factory.StartNew(
        state =>
        {
            try
            {
                using var mutex = new Mutex(false, _name);
                try
                {
                    // Wait for either the mutex to be acquired, or cancellation
                    if (WaitHandle.WaitAny(new[] { mutex, cancellationToken.WaitHandle }) != 0)
                    {
                        taskCompletionSource.SetCanceled(cancellationToken);
                        return;
                    }
                }
                catch (AbandonedMutexException)
                {
                    // Abandoned by another process, we acquired it.
                }

                taskCompletionSource.SetResult();

                // Wait until the release call
                _releaseEvent.Wait();

                mutex.ReleaseMutex();
            }
            catch (OperationCanceledException)
            {
                taskCompletionSource.TrySetCanceled(cancellationToken);
            }
            catch (Exception ex)
            {
                taskCompletionSource.TrySetException(ex);
            }
        },
        state: null,
        cancellationToken,
        TaskCreationOptions.LongRunning,
        TaskScheduler.Default);

    return taskCompletionSource.Task;
}
```

## Disposal
To ensure the mutex gets released, we should implement disposal. This should release the mutex if held. It should also cancel any currently waiting acquiring of the mutex, which requires a linked cancellation token.

```cs
private CancellationTokenSource? _cancellationTokenSource;

public Task AcquireAsync(CancellationToken cancellationToken)
{
    cancellationToken.ThrowIfCancellationRequested();

    TaskCompletionSource taskCompletionSource = new();

    _releaseEvent = new ManualResetEventSlim();
    _cancellationTokenSource = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);

    // Putting all mutex manipulation in its own task as it doesn't work in async contexts
    // Note: this task should not throw.
    _mutexTask = Task.Factory.StartNew(
        state =>
        {
            try
            {
                CancellationToken cancellationToken = _cancellationTokenSource.Token;
                using var mutex = new Mutex(false, _name);
                try
                {
                    // Wait for either the mutex to be acquired, or cancellation
                    if (WaitHandle.WaitAny(new[] { mutex, cancellationToken.WaitHandle }) != 0)
                    {
                        taskCompletionSource.SetCanceled(cancellationToken);
                        return;
                    }
                }
                catch (AbandonedMutexException)
                {
                    // Abandoned by another process, we acquired it.
                }

                taskCompletionSource.SetResult();

                // Wait until the release call
                _releaseEvent.Wait();

                mutex.ReleaseMutex();
            }
            catch (OperationCanceledException)
            {
                taskCompletionSource.TrySetCanceled(cancellationToken);
            }
            catch (Exception ex)
            {
                taskCompletionSource.TrySetException(ex);
            }
        },
        state: null,
        cancellationToken,
        TaskCreationOptions.LongRunning,
        TaskScheduler.Default);

    return taskCompletionSource.Task;
}

public async ValueTask DisposeAsync()
{
    // Ensure the mutex task stops waiting for any acquire
    _cancellationTokenSource?.Cancel();

    // Ensure the mutex is released
    await ReleaseAsync();

    _releaseEvent?.Dispose();
    _cancellationTokenSource?.Dispose();
}
```

## Conclusion
`AsyncMutex` allows usage of `Mutex` with `async`/`await`.

Putting the whole thing together (or view the [Gist](https://gist.github.com/dfederm/35c729f6218834b764fa04c219181e4e){:target="_blank"}):
```cs
public sealed class AsyncMutex : IAsyncDisposable
{
    private readonly string _name;
    private Task? _mutexTask;
    private ManualResetEventSlim? _releaseEvent;
    private CancellationTokenSource? _cancellationTokenSource;

    public AsyncMutex(string name)
    {
        _name = name;
    }

    public Task AcquireAsync(CancellationToken cancellationToken)
    {
        cancellationToken.ThrowIfCancellationRequested();

        TaskCompletionSource taskCompletionSource = new();

        _releaseEvent = new ManualResetEventSlim();
        _cancellationTokenSource = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);

        // Putting all mutex manipulation in its own task as it doesn't work in async contexts
        // Note: this task should not throw.
        _mutexTask = Task.Factory.StartNew(
            state =>
            {
                try
                {
                    CancellationToken cancellationToken = _cancellationTokenSource.Token;
                    using var mutex = new Mutex(false, _name);
                    try
                    {
                        // Wait for either the mutex to be acquired, or cancellation
                        if (WaitHandle.WaitAny(new[] { mutex, cancellationToken.WaitHandle }) != 0)
                        {
                            taskCompletionSource.SetCanceled(cancellationToken);
                            return;
                        }
                    }
                    catch (AbandonedMutexException)
                    {
                        // Abandoned by another process, we acquired it.
                    }

                    taskCompletionSource.SetResult();

                    // Wait until the release call
                    _releaseEvent.Wait();

                    mutex.ReleaseMutex();
                }
                catch (OperationCanceledException)
                {
                    taskCompletionSource.TrySetCanceled(cancellationToken);
                }
                catch (Exception ex)
                {
                    taskCompletionSource.TrySetException(ex);
                }
            },
            state: null,
            cancellationToken,
            TaskCreationOptions.LongRunning,
            TaskScheduler.Default);

        return taskCompletionSource.Task;
    }

    public async Task ReleaseAsync()
    {
        _releaseEvent?.Set();

        if (_mutexTask != null)
        {
            await _mutexTask;
        }
    }

    public async ValueTask DisposeAsync()
    {
        // Ensure the mutex task stops waiting for any acquire
        _cancellationTokenSource?.Cancel();

        // Ensure the mutex is released
        await ReleaseAsync();

        _releaseEvent?.Dispose();
        _cancellationTokenSource?.Dispose();
    }
}
```