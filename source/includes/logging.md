# Logging

The logging package provides a simple interface for logging, as well as a few nice log targets.

## Log

```csharp
Log.Debug(this, "This is a {0} {1}.", "debug", "log");
Log.Info(this, "This is an info log.");
Log.Warning(this, "This is a warning log.");
Log.Error(this, "This is an error log.");
Log.Fatal(this, "This is a fatal log.");
```

The `Log` class provides a static interface for logging. It has one method per `LogLevel`, and requires the caller to be specified first, then a message and replacements.

## Log Levels

> No log under `LogLevel.Error` will be output.

```csharp
Log.Filter = LogLevel.Error;
```

The `Log` class also features a level filter which can be used to globally set the log level:

## Log Targets

```csharp
Log.AddLogTarget(new DiagnosticsLogTarget(formatter));
```

The `ILogTarget` interface is where the meat of the logging system resides. This library provides a few implementations, which are described in more detail below.

These targets may be added and removed from the `Log` class at any time. Once an `ILogTarget` implementation has been added, logs will be forwarded to that target.

## DefaultLogFormatter

```csharp
Log.AddLogTarget(new UnityLogTarget(new DefaultLogFormatter
{
	Level = true,
	Timestamp = false,
	TypeName = true
}));
```

Most targets require an `ILogFormatter` to be passed along as well, which will format the logs for the specific target. `DefaultLogFormatter` is provided which will output logs with timestamp, callee, and a few other things.

## DiagnosticsLogTarget

```csharp
Log.AddLogTarget(new DiagnosticsLogTarget(formatter));
```

This target will forward logs to `System.Diagnostics.Debug`.

## FileLogTarget

```csharp
var target = new FileLogTarget(new DefaultFormatter(), "file.txt");
Log.AddLogTarget(target);
```

> This log target is also an `IDisposable`, so it should be cleaned up when removed.

```csharp
target.Dispose();
```

Forwards logs to a file.

## UnityLogTarget

```csharp
Log.AddLogTarget(new UnityLogTarget(new DefaultFormatter()));
```

Forwards logs to the Unity console.