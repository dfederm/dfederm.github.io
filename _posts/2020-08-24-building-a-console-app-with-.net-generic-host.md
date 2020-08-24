---
layout: post
title: Building a Console App with .NET Generic Host
categories: [.NET]
tags: [.NET, console, CLI, generic host]
comments: true
---

The [.NET Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host){:target="_blank"} is a feature which sets up some convenient patterns for an application including those for [dependency injection (DI)](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection){:target="_blank"}, [logging](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging){:target="_blank"}, and [configuration](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration){:target="_blank"}. It was originally named Web Host and intended for Web scenarios like ASP.NET Core applications but has since been generalized (hence the rename to *Generic* Host) to support other scenarios, such as Windows services, Linux daemon service, or even a console app.

A working example can be found at [dfederm/GenericHostConsoleApp](https://github.com/dfederm/GenericHostConsoleApp){:target="_blank"}.

## Basics

You will first need to create a new console application and add a `PackageReference` to `[Microsoft.Extensions.Hosting`](https://www.nuget.org/packages/Microsoft.Extensions.Hosting/){:target="_blank"}.

```
dotnet new console
dotnet add package Microsoft.Extensions.Hosting
```

Now for the `Main` method. Typically the `Main` method for console apps just immediately jump into the application's core logic, but when using the .NET Generic Host, instead the host is set up. This should look familiar if you've developed web applications using the Web Host or Generic Host before.

```cs
internal sealed class Program
{
    private static async Task Main(string[] args)
    {
        await Host.CreateDefaultBuilder(args)
            .RunConsoleAsync();
    }
}
```

Running that alone will start up the host, but without any logic it will do nothing and never exit. The generic host simply sets up some reasonable defaults around configuration and logging, as well as provides a few services in the DI container which handle the the application lifetime. Calling `RunConsoleAsync` will run start the host and wait for a `Ctrl+C` or `SIGTERM` to exit, which means without explicitly telling the app to exit, it will not exit.

To actually implement your console app's logic, and get the application to exit after it's done, you'll want to implement and register an `IHostedService`, as well as interact with the `IHostApplicationLifetime` from the DI container.

```cs
internal sealed class Program
{
    private static async Task Main(string[] args)
    {
        await Host.CreateDefaultBuilder(args)
            .ConfigureServices((hostContext, services) =>
            {
                services.AddHostedService<ConsoleHostedService>();
            })
            .RunConsoleAsync();
    }
}

internal sealed class ConsoleHostedService : IHostedService
{
    private readonly ILogger _logger;
    private readonly IHostApplicationLifetime _appLifetime;

    public ConsoleHostedService(
        ILogger<ConsoleHostedService> logger,
        IHostApplicationLifetime appLifetime)
    {
        _logger = logger;
        _appLifetime = appLifetime;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogDebug($"Starting with arguments: {string.Join(" ", Environment.GetCommandLineArgs())}");

        _appLifetime.ApplicationStarted.Register(() =>
        {
            Task.Run(async () =>
            {
                try
                {
                    _logger.LogInformation("Hello World!");

                    // Simulate real work is being done
                    await Task.Delay(1000);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Unhandled exception!");
                }
                finally
                {
                    // Stop the application once the work is done
                    _appLifetime.StopApplication();
                }
            });
        });

        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        return Task.CompletedTask;
    }
}
```

Other useful events to subscribe to on the `IHostApplicationLifetime` are `ApplicationStopping` and `ApplicationStopped`.

Note that you can registrer multiple `IHostedService` implementations and each of them will have their `StartAsync` and `StopAsync` methods call. However, personally I find that to be a bit confusing for a console application.

## Dependency Injection

As you may have noticed, your `IHostedService` implementation has full access to the DI container, so you can register services in `Main` and then use them in your `IHostedService`.

In `Main`:
```cs
await Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        services
            .AddHostedService<ConsoleHostedService>();
            .AddSingleton<IWeatherService, WeatherService>();
    })
    .RunConsoleAsync();
```

In `ConsoleHostedService`

```cs
private readonly ILogger _logger;
private readonly IHostApplicationLifetime _appLifetime;
private readonly IWeatherService _weatherService;

public ConsoleHostedService(
    ILogger<ConsoleHostedService> logger,
    IHostApplicationLifetime appLifetime,
    IWeatherService weatherService)
{
    _logger = logger;
    _appLifetime = appLifetime;
    _weatherService = weatherService;
}

public Task StartAsync(CancellationToken cancellationToken)
{
    _appLifetime.ApplicationStarted.Register(() =>
    {
        Task.Run(async () =>
        {
            try
            {
                IReadOnlyList<int> temperatures = await _weatherService.GetFiveDayTemperaturesAsync();
                for (int i = 0; i < temperatures.Count; i++)
                {
                    _logger.LogInformation($"{DateTime.Today.AddDays(i).DayOfWeek}: {temperatures[i]}");
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Unhandled exception!");
            }
            finally
            {
                // Stop the application once the work is done
                _appLifetime.StopApplication();
            }
        });
    });

    return Task.CompletedTask;
}
```

## Exit Code
To return a non-zero exit code, your `IHostedService` implementation can set `Environment.ExitCode` when the host is stopping.

```cs
    internal sealed class ConsoleHostedService : IHostedService
    {
        private int? _exitCode;

        // ...

        public Task StartAsync(CancellationToken cancellationToken)
        {
            // ...

            _appLifetime.ApplicationStarted.Register(() =>
            {
                Task.Run(async () =>
                {
                    try
                    {
                        // ...

                        _exitCode = 0;
                    }
                    catch (Exception ex)
                    {
                        _logger.LogError(ex, "Unhandled exception!");
                        _exitCode = 1;
                    }
                    finally
                    {
                        // Stop the application once the work is done
                        _appLifetime.StopApplication();
                    }
                });
            });

            return Task.CompletedTask;
        }

        public Task StopAsync(CancellationToken cancellationToken)
        {
            _logger.LogDebug($"Exiting with return code: {_exitCode}");

            // Exit code may be null if the user cancelled via Ctrl+C/SIGTERM
            Environment.ExitCode = _exitCode.GetValueOrDefault(-1);
            return Task.CompletedTask;
        }
    }
```

## Logging and Configuration
Logging and configuration work mostly just as they do in Web applications. The one caveat is that `appsettings.json` is loaded from the content root, which by default is the current working directory, so the content root needs to be set to the same directly the executable is in using `UseContentRoot`.

In `Main`:
```cs
await Host.CreateDefaultBuilder(args)
    .UseContentRoot(Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location))
    .ConfigureLogging(logging =>
    {
        // Add any 3rd party loggers like NLog or Serilog
    })
    .ConfigureServices((hostContext, services) =>
    {
        services
            .AddHostedService<ConsoleHostedService>()
            .AddSingleton<IWeatherService, WeatherService>();

        services.AddOptions<WeatherSettings>().Bind(hostContext.Configuration.GetSection("Weather"));
    })
    .RunConsoleAsync();
```

In `WeatherService.cs`:
```cs
internal sealed class WeatherService : IWeatherService
{
    private readonly IOptions<WeatherSettings> _weatherSettings;

    public WeatherService(IOptions<WeatherSettings> weatherSettings)
    {
        _weatherSettings = weatherSettings;
    }

    public Task<IReadOnlyList<int>> GetFiveDayTemperaturesAsync()
    {
        int[] temperatures = new[] { 76, 76, 77, 79, 78 };
        if (_weatherSettings.Value.Unit.Equals("C", StringComparison.OrdinalIgnoreCase))
        {
            for (int i = 0; i < temperatures.Length; i++)
            {
                temperatures[i] = (int)Math.Round((temperatures[i] - 32) / 1.8);
            }
        }

        return Task.FromResult<IReadOnlyList<int>>(temperatures);
    }
}
```

In `WeatherSettings.cs`:
```cs
internal sealed class WeatherSettings
{
    public string Unit { get; set; }
}
```

In the project file:
```xml
  <ItemGroup>
    <Content Include="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>
```

In `appsettings.json`:
```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Information",
            // Avoid logging lifetime events
            "Microsoft.Hosting.Lifetime": "Warning"
        }
    },
    "Weather": {
        "Unit": "C"
    }
}
```