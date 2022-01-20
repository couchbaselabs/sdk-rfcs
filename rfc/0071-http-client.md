# RFC-0071 HTTP Client

## Meta

* RFC Name: SDK3 HTTP Client API
* RFC ID: 0071-sdk3-http-client
* Start Date: 2022-01-19
* Owner: David Nault
* Current Status: Draft

## Summary

Specification for a public SDK 3 API for making custom HTTP requests to Couchbase Server.

## Motivation

Couchbase Server has a well-documented public HTTP interface.
The SDK 3 Management APIs provide abstractions over much of the HTTP interface, but some advanced options are not available in the Management APIs.
To cover those gaps, we'd like to make it easier for developers to make custom HTTP requests.

An HTTP client built into the SDK relieves developers of these burdens:

* **Routing the request to an appropriate Couchbase Server node.**
  In a deployment that uses multidimensional scaling, some services are only available on certain nodes.
  The SDK already has a map of where the services are running, and uses this information for request routing.
* **Establishing a secure connection.**
  The SDK already knows which TLS certificates to trust.
* **Presenting user credentials.**
  The SDK already knows the username and password, or in the case of mutual TLS, the client certificate.

In other words, the SDKs are good at dispatching secure, authenticated HTTP requests to Couchbase Server.
The goal is to expose this capability in a way that's easy and safe for developers to use.

## Limitations

This is not a general-purpose HTTP client; it can only talk to Couchbase Server.

The HTTP client is not intended for making requests that return streaming responses (like executing a N1QL query, for example).
The entire response is expected to fit comfortably in memory.

## Requirements

A compliant implementation has the following capabilities:


* The client inherits credentials and transport security settings from the `Cluster`.


* A user specifies which Couchbase service should handle the request, and the client dispatches the request to an appropriate host and port.


* The supported HTTP verbs are: GET, POST, PUT, and DELETE.


* All requests have a path.


* GET requests may have query parameters.
  The user should be able to specify query parameters separately from the path (so the client can automatically URL-encode them).


* POST and PUT requests may have a body with content type `application/json` or `application/x-www-form-urlencoded`.


* A user can specify query strings and form bodies with duplicate names. For example, `color=green&color=blue`.


* The client automatically URL-encodes names and values in query parameters and form bodies, unless the user goes out of their way to bypass the encoding.
  An "escape hatch" allows users to provide a pre-encoded query string or form body.


* The client provides a way to replace placeholders (`{}`) in a path with URL-encoded arguments.


* The user can access the HTTP status code and content of the server response.

### Accessing the HTTP client

The HTTP client is available via the `Cluster`.
`Cluster` has a new `httpClient()` method that returns a `CouchbaseHttpClient`.

```java
class Cluster {
    ...

    /**
     * Returns an HTTP client configured to talk to this cluster.
     */
    CouchbaseHttpClient httpClient()
}
```

## Reference Design

This section offers recommendations for implementing the HTTP client.
Implementers should adhere to the reference design where it makes sense, and diverge where the idioms of a specific language or SDK provide a more natural solution.

Except where noted, ***all properties should be considered implementation details and hidden from the user*** (made `private` or equivalent).

### NodeIdentifier

Immutable value object.
Identifies a specific node within a Couchbase cluster.

An implementation that does not support targetting a specific node may omit this component.

#### Properties

* `hostname` String. Name of the machine hosting the node.
* `managerPort` int. Port where the manager service is listening.

### HttpTarget

Immutable value object.
Specifies where to dispatch the request.

#### Properties

* `serviceType` Enum. Identifies the Couchbase service (MANAGER, QUERY, etc.) that should service the request.
* `nodeIdentifier` NodeIdentifier (Optional). If absent, SDK selects an arbitrary node running the service.
* `bucketName`: String (Optional). Required when dispatching to the VIEWS service, to select a node with active KV partitions.

NOTE: An implementation may omit the `nodeIdentifer` property if they do not support targeting a specific node.
Likewise, the `bucketName` may be omitted if the implementation does not need it.

#### Factory methods / Constructors

1. `HttpTarget.analytics()`
2. `HttpTarget.eventing()`
3. `HttpTarget.manager()`
4. `HttpTarget.query()`
5. `HttpTarget.search()`
6. `HttpTarget.views(String bucketName)`

The `withNode(NodeIdentifier)` instance method creates a copy that targets the given node.

### NameValuePairs

Immutable value object.
Represents a query string or form data.

#### Properties

* `urlEncoded` String. The URL-encoded form of the query string or form data (not including the leading `?`)

#### Factory methods / Constructors

1. `NameValuePairs.of(Map<String, ?>)`
   Takes a map from name to value.
   The value may be any type that is convertible to a String (or just String, depending on the language).
   URL-encodes the names and values before joining them to initialize `urlEncoded`.
   The order of the encoded parameters is determined by the map's iteration order.

```java
NameValuePairs nvp = NameValuePairs.of(
    Map.of(
        "friendly greeting", "hello world",
        "number", 3
    )
)

// friendly%20greeting=hello%20world&number=3
```

2. `NameValuePairs.of(List<Pair<String, ?>>)`
   Takes a list of pairs (name -> value).
   Languages with varargs may provide an additional overload that accepts a variable number of pairs.
   The value may be any type that is convertible to a String (or just String, depending on the language).
   URL-encodes the names and values before joining them to initialize `urlEncoded`.
   The order of the encoded parameters is the same as their order in the given list.

```java
NameValuePairs nvp = NameValuePairs.of(
    List.of(
        Pair.of("friendly greeting", "hello world"),
        Pair.of("friendly greeting", "Olá"),
        Pair.of("number", 3),
    )
)

// friendly%20greeting=hello%20world&friendly%20greeting=Ol%C3%A1&number=3
```

3. `NameValuePairs.ofPreEncoded(String)`
   This is the "escape hatch" where a developer can supply their own pre-encoded string.
```java
NameValuePairs nvp = NameValuePairs.ofPreEncoded(
    "Garbage in, garbage out!"
)

// Garbage in, garbage out!
```

### HttpBody

Immutable value object.
Represents the body of a POST request.

#### Properties

* `contentType` String.
  Either "application/json" or "application/x-www-form-urlencoded".

* `content` Byte array. The payload to post.

#### Factory methods / Constructors

1. `HttpBody.json(ByteArray)`
   Creates an HTTP body with content type "application/json" and the given content.

2. `HttpBody.json(String)`
   Creates an HTTP body with content type "application/json".
   Content is the given string encoded as UTF-8.

3. `HttpBody.form(NameValuePairs)`
   Creates an HTTP body with content type "application/x-www-form-urlencoded".
   Content is the `urlEncoded` property encoded as UTF-8.

4. As a convenience, implementations may add `form` overloads whose signatures match the `NameValuePair` constructors and create the NameValuePair internally before delegating to `form(NameValuePair)`.

### HttpResponse

Immutable value object.
Represents the result of an HTTP request.

#### Properties

These properties are public / visible to the user.

* `statusCode` int.
  The HTTP status code returned by Couchbase Server.

* `contentType` String (Optional)
  The value of the Content-Type header, if present in the response from Couchbase Server.

* `content` Byte array.
  The content of the HTTP response.
  Possibly zero-length, but never null.

If the implementation does not have access to the response headers, it may omit the `contentType` property.

### CouchbaseHttpClient

Instances must be thread-safe.

#### Factory methods / Constructors

Users do not create instances directly.
`Cluster` has a new instance method `httpClient()` that returns a CouchbaseHttpClient.

#### Other methods

Instance methods of CouchbaseHttpClient correspond to the supported HTTP verbs.

Every method takes an optional `CommonOption` parameter block consisting of the usual `timeout`, `retryStrategy`, etc. common to all SDK 3 requests.

```java
class CouchbaseHttpClient {
    HttpResponse get(
        HttpTarget target,
        String path,
        NameValuePairs queryString, // optional
        CommonOptions common // optional
    );

  HttpResponse post(
      HttpTarget target,
      String path,
      HttpBody body, // optional
      CommonOptions common // optional
  );

  HttpResponse put(
      HttpTarget target,
      String path,
      HttpBody body, // optional
      CommonOptions common // optional
  );

  HttpResponse delete(
      HttpTarget target,
      String path,
      CommonOptions common // optional
  );
}
```

Implementations in languages that lack support for default arguments might provide multiple overloads for each HTTP verb.
For example:

```
HttpResponse get(target, path)
HttpResponse get(target, path, queryString)
HttpResponse get(target, path, common)
HttpResponse get(target, path, queryString, common)
```

Alternatively, an implementor might opt to bundle all optional parameters into a method-specific options block, such as `HttpGetOptions`, `HttpPostOptions`, etc.

### Utility methods

#### formatPath
Implementations should provide a `formatPath` convenience method for building path strings.
This can be a static method of the `CouchbaseHttpClient` class, or whatever makes the most sense for the implentation language.

This method has two parameters: a template string, and a varargs array (or List) of argument strings.

`fomatPath` replaces each `{}` placeholder in the template string with the URL-encoded form of the corresponding additional argument.
For example:
```
formatPath("/foo/{}/bar/{}", "hello world", "a/b")
```
returns the string `/foo/hello%20world/bar/a%2Fb`

If the number of placeholders in the template string does not match the number of additional arguments, this method throws IllegalArgumentException (or equivalent).

### Implementation notes

If the user provides a path that does not start with a slash, an implementation should prepend a slash to make it valid.

If the user provides a path that contains the query string delimiter (`?`), an implementation should use `&` instead of `?` as the delimiter when appending the `queryString` value to the path.
This allows the user to completely opt out of the query string parameter by encoding the query string as part of the path.

For example, the following should be valid:

```
val response = httpClient.get(
    target = HttpTarget.manager(),
    path = formatPath("foo/bar?color={}", red),
    queryString = NameValuePairs.of("magicWord" to "xyzzy"),
)
```

and should result in a path + query string of:

    /foo/bar?color=red&magicWord=xyzzy

#### Asynchronous requests

An implementation may provide multiple flavors of CouchbaseHttpClient to support blocking vs. asynchronous APIs, following the same patterns used elsewhere in the SDK 3 APIs.


## Changelog
* Jan 20, 2022 - Revision #1 (by David Nault)
    * Initial Draft


## Signoff

| Language     | Team Member        | Signoff Date | Revision |
|--------------|--------------------|--------------|----------|
| C            | Sergey Avseyev     |              |          |
| Go           | Charles Dixon      |              |          |
| Java         | Michael Nitchinger |              |          |
| .NET         | Jeffry Morris      |              |          |
| Node.js      | Brett Lawson       |              |          |
| PHP          | Sergey Avseyev     |              |          |
| Python       | Jared Casey        |              |          |
| Ruby         | Sergey Avseyev     |              |          |
| Scala        | Graham Pople       |              |          |
