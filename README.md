# `net6.0` Serilog example

This is a sample project showing Serilog configured in the default .NET 6 web application template.

To show how everything fits together, the sample includes:

 * Support for .NET 6's `ILogger<T>` and `WebApplicationBuilder`
 * Namespace-specific logging levels to suppress noise from the framework
 * JSON configuration
 * Clean, themed console output
 * Local logging to a rolling file
 * Centralized structured logging with Seq
 * Streamlined HTTP request logging
 * Filtering out of noisy events using _Serilog.Expressions_
 * Exception logging
 * Fallback/fail-safe bootstrap logger
 * Proper flushing of logs at exit

The code is not commented, so that the structure is easier to compare with the default template. If you're keen 
to understand the trade-offs and reasoning behind the choices made here, there's some commentary on each section
in _Setting up from scratch_ below.

## Trying it out

You'll need the .NET 6.0 SDK or later to run the sample. Check the version you have installed with:

```shell
dotnet --version
```

After checking out this repository or downloading a zip file of the source code, you can run the project with:

```shell
dotnet run
```

Some URLs will be printed to the terminal: open them in a browser to see request logging in action.

 * `/` &mdash; should show "Hello, world!" and respond successfully
 * `/oops` &mdash; throws an exception, which will be logged

To see structured log output, start a temporary local Seq instance with:

```shell
docker run --rm -it -e ACCEPT_EULA=y -p 5341:80 datalust/seq
```

and open `http://localhost:5341` in your browser (for Windows users, there's also an MSI at https://datalust.co/download).

## Setting up from scratch

You can freely copy code from this project to your own applications. If you'd like to set up from scratch, and skip any steps that aren't relevant to you,
try following the steps below.

### 1. Create the project using the `web` template

```shell
mkdir dotnet6-serilog-example
cd dotnet6-serilog-example
dotnet new web
```

### 2. Install Serilog packages

```shell
dotnet add package serilog.aspnetcore
dotnet add package serilog.sinks.seq
dotnet add package serilog.expressions
```

### 3. Initialize Serilog at the start of `Program.cs`

Its important that logging is initialized as early as possible, so that errors that might prevent your app from starting are logged.

At the very top of `Program.cs`:

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

Log.Information("Starting up");
```

`CreateBootstrapLogger()` sets up Serilog so that the initial logger configuration (which writes only to `Console`), can be swapped 
out later in the initialization process, once the web hosting infrastructure is available.

### 4. Wrap the rest of `Program.cs` in `try`/`catch`/`finally`

Configuration of the web application in `Program.cs` can now be enclosed in a `try` block.

```csharp
try
{
    // <snip>
}
catch (Exception ex)
{
    Log.Fatal(ex, "Unhandled exception");
}
finally
{
    Log.Information("Shut down complete");
    Log.CloseAndFlush();
}
```

The `catch` block will log any exceptions thrown during start-up.

The `Log.CloseAndFlush()` in the `finally` block ensures that any queued log events will be properly recorded when the program exits.

### 5. Wire Serilog into the `WebApplicationBuilder`

```csharp
    builder.Host.UseSerilog((ctx, lc) => lc
        .WriteTo.Console()
        .ReadFrom.Configuration(ctx.Configuration));
```

### 6. Replace logging configuration in `appsettings.json`

In `appsettings.json`, remove `"Logging"` and add `"Serilog"`.

The complete JSON configuration from the example is:

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information"
      }
    },
    "Filter": [
      {
        "Name": "ByExcluding",
        "Args": {
          "expression": "@mt = 'An unhandled exception has occurred while executing the request.'"
        }
      }
    ],
    "WriteTo": [
      {
        "Name": "File",
        "Args": { "path":  "./logs/log-.txt", "rollingInterval": "Day" }
      },
      {
        "Name": "Seq",
        "Args": { "serverUrl":  "http://localhost:5341" }
      }
    ]
  },
  "AllowedHosts": "*"
}
```

Remove all `"Logging"` configuration from `appsettings.Development.json`. (During development, you should normally use the same logging
level as you use in production; if you can't find problems using the production logs in development, you'll have an even harder time
finding problems in the real production environment.)

### 7. Add Serilog's request logging middleware

By default, the ASP.NET Core framework logs multiple information-level events per request.

Serilog's request logging streamlines this, into a single message per request, including path, method, timings, status code, and exception.

```csharp
    app.UseSerilogRequestLogging();
```

## Writing log events

This setup enables both Serilog's static `Log` class, as you see used in the example above, and _Microsoft.Extensions.Logging_'s
`ILogger<T>`, which can be consumed through dependency injection into controllers and other components.

## Viewing structured logs

If you're running a local Seq instance, you can now view the structured properties attached to your application logs in the Seq UI:

![Application logs in Seq](https://github.com/datalust/dotnet6-serilog-example/blob/dev/asset/structured-data-in-seq.png)

## Getting help and advice

Ask your question on [Stack Overflow](https://stackoverflow.com) and tag it with `serilog`.
