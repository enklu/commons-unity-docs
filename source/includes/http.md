# Http

## Overview

`IHttpService` is the entry point into making Http requests in Unity. This interface is fairly low level, allowing you to configure every part of an Http request-- but it also provides some powerful abstractions to help make writing requests and responses quick and easy.

To make a request, `IHttpService` provides methods for several Http verbs (we do not support some of the more esoteric verbs (HEAD, CONNECT, TRACE, etc).

## Setup

```csharp

var service = new HttpService(new JsonSerializer(), bootstrapper, new UrlFormatterCollection());

```

One standard implementation is provided: `HttpService`. This implementation requires an `ISerializer`, an `IBootstrapper`, and a `UrlFormatterCollection`. The serializer is so that `HttpService` can interact with XML, JSON, or any other serialization strategy. The bootstrapper is needed to abstract the service away from `MonoBehaviour`, but still use the Unity Http API. Lastly, the `UrlFormatterCollection` is to provide custom protocols for specific endpoints.

In general, code should be written against the interface, not the implementation, allowing the user to stub out the service for testing.

### Service Registration

> The base url for service requests

```csharp
var serviceUrl = "http://myrestservice.com:9999/v1/";
```

> Create the formatter

```csharp
var serviceFormatter = new UrlFormatter();
if (!serviceFormatter.FromUrl(serviceUrl))
{
	throw new Exception("Failed to create url formatter for service.");
}
```

> Register the service using a custom service name

```csharp
_http.Services.Register("myservice", serviceFormatter);
```

> Shorthand protocol notation for making requests

```csharp
_http.Get("myservice://users/create")...
```

Registering a service using the [`UrlFormatter`](#urlformatter) provides convenience for specifying short-hand urls for endpoints. While registering a service for all requests is not necessary, it is a requirement if custom headers are used on a request.

Registering a service name not only provides covenience for url resolution, but also allows separation of service configuration and specific service headers.

## Headers and Authentication

```csharp
_http.Services.AddHeader("myservice", "Authorization", "Bearer " + token);
```

> Future requests made to the specific service will pass authentication header.

```csharp
_http.Get("myservice://users/create")...
```

Currently, headers may only be set for a specific service, not globally or per request.

In our case, authentication is done via headers. We use the JWT standard, so once a user has obtained a token for a service, the service header is added for the token.


## GET/DELETE

### Making a Request

```csharp
_http
	.Get<MyResponse>(url)
	.OnSuccess(response => ...)
	.OnFailure(exception => ...);
```

Both `GET` and `DELETE` requests have their own method on `IHttpService` that accepts a single URL parameter. All methods return an `IAsyncToken`.

### Inspecting the Response

```csharp
_http
	.Get<MyResponse>(url)
	.OnSuccess((HttpResponse<MyResponse> response) =>
	{
		Log(response.StatusCode);
		Log(response.Headers["X-Custom-Header"]);
		Log(Encoding.UTF8.GetString(response.Raw));
	});
```

The methods are parameterized with the type of the response. An `HttpResponse<MyResponse>` is returned and includes network information about the request, like `StatusCode` and `Headers`. The request is considered successful if and only if the response status code is in the 200 range.

When a request does fail, an exception is generated and passed to `OnFailure`. This exception is populated either with the transport failure, or the response from the server.

### Reading the Payload

```csharp
_http
	.Get<MyResponse>(url)
	.OnSuccess((HttpResponse<MyResponse> response) =>
	{
		// will be of type MyResponse
		var myResponse = response.Payload;

		Foo(myResponse.Bar);
	})
	.OnFailure(exception => ...);
```

The payload is deserialized and returned as the `Payload` property.

## POST/PUT

```csharp
_http
	.Post<MyRespose>(url, new MyRequest(...))
	.OnSuccess(...)
	.OnFailure(...);
```

POST and PUT requests are different only in that they allow passing in a body. With `HttpService`, the body is serialized using the passed in `ISerializer`.

## UrlFormatter

### Basic Usage

```csharp
var formatter = new UrlFormatter();
formatter.BaseUrl = "my.baseurl.com";
formatter.Port = 10206;
formatter.Version = "v2";

var url = formatter.Url("/user/" + myUserId);
```

Generally, applications will not want to hard code URLs to make Http calls. Protocols, environments, ports, etc-- all of these are subject to change. Furthermore, users don't want to write the same `.Trim` methods over and over, making sure `/`s are in the right place. To provide for these needs, the `IHttpService.Services` accepts a `UrlFormatter` when [registering a service](#service-registration) which automatically cleans and formats urls which contain the service protocol.

### Replacement Tokens

> Add a replacement.

```csharp
formatter.Replacements.Add(Tuple.Create("myUserId", userId));
```

> Done as part of a registered service on `IHttpService`

```csharp
var formatter = _http.Services.GetUrlFormatter("myservice");
formatter.Replacements.Add(Tuple.Create("myUserId", userId));
```

> Now the token may be used inline.

```csharp
var url = formatter.Url("/user/{myUserId}");
```

> Making a request via registered service on `IHttpService`

```csharp
_http.Get("myservice://user/{myUserId}");
```

Tokens may also be added for automatic replacement.

## Further Reading

* [JWT](http://jwt.io)
* [Http Status Codes](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)
* [Http Request Methods](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html)

## Future needs

* Ability to set headers per request.
* Request failure should return `IHttpException`, which wraps exception with status code.