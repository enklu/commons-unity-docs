# Messaging

## Overview

The messaging library is designed to be intuitive and simple. It is not a message _bus_ intended for anything and everything to pass through. This messaging system is also, very purposefully, _single threaded_. For use in multithreaded applications, be sure to move messages to a single thread before publishing them (or choose another library).

## Abstract

```csharp
var message = new MessageRouter();
```

The messaging system has one main class: `MessageRouter` with two primary methods: `publish` and `subscribe`. Any object may be published, so a base class for messages is not required (or encouraged).

You can have any number of `MessageRouter`s in your application. They do not share state of any kind.

## Subscribe

> Subscribe to a message.

```csharp
router.Subscribe(10, message => {
	...
});
```

> Subscribe to a message once.

```csharp
router.SubscribeOnce(messageType, message => {
	...
	
	// automatically unsubscribed
});
```

> Subscribe to all events.

```csharp
router.SubscribeAll((message, messageType) => {
	// this method will receive all messages

	...
});
```

To subscribe to a message type, simply call one of the flavors of `Subscribe` with the integer corresponding to the message type.

## Unsubscribe

> There is no way to safely remove this lambda.

```csharp
MyThing.OnAdded += message => Log.Info(this, "Received!");
```

> Subscribe functions return function to unsubscribe with.

```csharp
var unsub = router.Subscribe(...);
```

> Use Subscribe overload to pass unsubscribe function to subscriber.

```csharp
router.Subscribe(messageType, (message, unsub) => {
	unsub();
});
```

Unsubscription occurs through functions returned from subscription. This is so that we can safely allow subscription of any type of method: named class methods, delegates, or lambdas.

Each flavor of `Subscribe` has two different forms: one that returns an unsubscription method, and one that passes it to the callback. Subscription and unsubscription is _stack-safe_, meaning unsubscribe is safe to call at any time in a single threaded application.

## Consume

```csharp
router.Subscribe(messageType, message => {
	...

	// next subscribers won't get message 
	router.Consume(message);
});
```

Sometimes it is useful to stop a message from propogating to later handlers. Since there is no required base class for messages, this is done by calling the router itself.

## Publish

```csharp
router.Publish(messageType, message);
```

> Publish many messages.

```csharp
router.Publish(messageType, messageA, messageB, messageC);
```

Publishing a message is straight forward.