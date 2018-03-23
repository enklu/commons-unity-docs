# Commons-Unity-Async

The Async package provides primitives for working with asynchronous code. Many projects within the `Common` namespace use the `IAsyncToken<T>` interface, so it is best to get well acquainted with it. This object may be considered either a very simplified Promise, or just a wrapper around a callback.

## The Limitation of Callbacks

```csharp
_controller.Foo(MyCallback);
```

In many codebases, callbacks are used to manage asynchronous code. Callbacks are great! They are handy for a whole host of applications, and we use them frequently in special cases. However, callbacks are lacking in some key areas.

```csharp
public Resource MyResource;

public void Open()
{
	_controller.Foo(Callback);
}

public void Close()
{
	MyResource.Destroy();
}

private void Callback()
{
	MyResource.Bar();
}
```

Most notably, callbacks provide a great method for subscribing, but provide no out-of-the-box method for unsubscribing. Consider this UI class.

This is a simple example, but demonstrates the problem well. Here we have some asynchronous method `Foo` which is given a callback. That callback may be called at any future time. Now the problem: what if `Close` is called before `Callback`? We would have an error, because `Callback` is making an assumption on the order of execution. We could add a Boolean to tell us we're closed, but that's easy to forget. What we'd really like to be able to do is prevent the callback from ever being called.

## The Limitation of Events

```csharp
public void Open()
{
	_controller.OnEvent += Callback;
	_controller.Foo();
}

public void Close()
{
	_controller.OnEvent -= Callback;
}
```

C# has a builting asynchronous primitive called `event`, which allows for subscription and unsubscription. Thus, we can get around the earlier problem we had with callbacks. Unsubscribing from events means we don't need to have certain classes of conditional logic in our event handlers, because we know they have been called within a certain window.

```csharp
public MyClass(Dependency dependency)
{
	dependency.OnReady += Initialize;
}
```

However, it has a whole different problem.

In this example, we have an object that needs something to happen before it can initialize. However, what if `Dependency` has already called `OnReady`? This object will be waiting forever. The problem with events is that you may subscribe to late.

Thus, the need arises for a primitive that will fix these two problems.

## IAsyncToken<T>

> Execute an asynchronous operation.

```csharp
var token = _controller.Operation();
```

> OnSuccess is called if the operation was a success.

```csharp
token.OnSuccess(value => ...);
```

> OnFailure is called if the operation was a failure.

```csharp
token.OnFailure(exception => ...);
```

> OnFinally is called after all success and failure handlers.

```csharp
token.OnFinally(_ => ...);
```

Our answer is `IAsyncToken<T>`. This object has acts much like an asynchronous counterpart to `try/catch/finally` blocks. At a high level, this object represents an asynchronous action being performed. This object accepts exactly one resolution.

In this example, internally to `Operation`, the token is being resolved with either a success or a failure. It cannot be resolved with both. Once a token has been resolved, its resolution cannot be changed.

Additionally, `Operation` *may* be asynchronous, but it may not be. Consider the interface for a lazy cache-- the first call may be asynchronous, the second may be synchronous. For `IAsyncToken<T>`, it doesn't matter. Unlike an event, the callback will be called immediately if already resolved.

## Chaining

```csharp
_controller
	.Foo()
	.OnSuccess(value => ...)
	.OnFailure(exception => ...)
	.OnFinally(token => ...);
```

For syntactic sugar familiar to Promise-users, you can also combine all of these calls in a chain:

## Multiple Callbacks

```csharp
_controller
	.Foo()
	.OnSuccess(value => ...)
	.OnSuccess(value => ...)
	.OnSuccess(value => ...);
```

Much like an event, multiple callbacks may be added. These callbacks will be called in the order they were added.

## AsyncToken<T>

```csharp
public IAsyncToken<MyClass> Foo()
{
	var token = new AsyncToken<MyClass>();
  
	// do something
	InternalMethod(value =>
	{
    	if (value.IsValid())
    	{
      		token.Succeed(value);
    	}
    	else
    	{
      		token.Fail(new Exception("Could not retrieve instance."));
    	}
  	});
  
  	return token;
}
```

Much like the Promise/Deferred pattern, the methods for resolving a token and triggering the handlers are not present on the `IAsyncToken<T>` interface. Instead, these methods are present only on the `AsyncToken<T>` implementation. This allows us to very simply protect a token from being tampered with externally.

In this example, the `Foo` method needs to do some sort of asynchronous work. `InternalMethod` happens to take a callback, but it could listen for a load event, wait on a message from a thread, use a coroutine, or any other number of asynchronous structures. `Foo` then returns the token, but it is cast to the interface, `IAsyncToken<T>`. This means that the `Succeed` method is not visible to the object calling `Foo`, thus that object cannot resolve the token without breaking the contract and casting the object.

## Resolving

> Resolve a token via `Succeed` or `Fail`.

```csharp
token.Succeed(myValue);			// calls OnSuccess
token.Fail(new Exception());	// token already resolved, discarded

...

token
	.OnSuccess(value => ...)		// called
	.OnFailure(exception => ...);	// never called
```

> In addition, a token may be resolved via constructor.

```csharp
var successful = new AsyncToken<Foo>(new Foo());
var failure = new AsyncToken<Foo>(new Exception("Nope."));
```

Resolving tokens is a one time deal. A single token can be resolved once and only once. Any resolution after the first resolution is discarded.

## Abort

The `Abort` method essentially destroys a token. Once `Abort` is called, no handler will be called ever again.

```csharp
private IAsyncToken<MyValue> _fooToken;

public void Foo()
{
	_fooToken = _controller
		.Foo()
		.OnSuccess(Callback);
}

public void Close()
{
	_fooToken.Abort();
{
```

Revisiting the problem we had with callbacks, let's do this again with tokens. After `Abort` is called, the token is dead. It cannot be resolved and it will call no future callbacks. In this case, if `Close` is called before the token calls `Callback`, then `Callback` will not be called at all. This gets around the limitations of both callbacks and events.

## Exception Handling

```csharp
_token.OnSuccess(myValue => throw Exception());
_token.Succeed(3);	// throws exception at this line
```

If a callback assigned to a token throws an exception, what should happen? There are a number of ways to answer this. The easiest way is to let exceptions propogate naturally, i.e. do nothing at all.

This can lead to inconsistencies and hard to track bugs, unfortunately.

```csharp
_token.OnSuccess(myValue => throw Exception());
_token.OnSuccess(myValue => Log.Info(this, "Received!"));
```

Here, our second callback would not be called at all! There is the potential that these callbacks may be in separate classes. One may even be in a separate dll.

```csharp
_token.OnSuccess(myValue => throw Exception());
_token.OnSuccess(myValue => Log.Info(this, "Received!")); // this is guaranteed to be called
_token.Succeed(value);	// the exception is thrown here, after both callbacks
```

Instead, if a callback is added to a token, it is _guaranteed_ to be called. Internally, the token simply catches any exceptions that occur during execution, and throws them again after all callbacks have been called.

## Void

```csharp
var token = new AsyncToken<Void>(Void.Instance);
```

There are some cases in which it's handy not to have to have to return anything in a token. In these cases, it's recommended to use `Void`.

## Token()

```csharp
IAsyncToken<Void> Bar()
{
	return _fooer
		.Foo()
		.OnSuccess(...)
		.Token();
}
```

Sometimes it's handy to create a new token from an existing token. In this example, we want to protect the token from `_fooer.Foo()` because `Bar` adds its own callbacks. If the token were returned directly, the caller could call `Abort` which would remove our internal callbacks as well. In this case, we call `Token` which returns a new token that is chained to the existing token.

## Further Reading

* For loads of other edge cases, check out the [unit test suite](https://github.com/create-ar/commons-unity-async/blob/master/CreateAR.Commons.Unity.Async.Test/AsyncToken_Tests.cs).