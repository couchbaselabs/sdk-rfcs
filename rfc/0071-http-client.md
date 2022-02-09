# RFC-0071 HTTP Client

## Meta

* RFC Name: HTTP Client
* RFC ID: 0071-http-client
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
  The client automatically sets the "Content-Type" header to the appropriate value.


* A user can specify additional HTTP request headers.


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

### HttpTarget

Immutable value object.
Specifies where to dispatch the request.

#### Properties

* `serviceType` Enum. Identifies the Couchbase service (MANAGER, QUERY, etc.) that should service the request.

NOTE: Since HttpTarget has only one property, you might wonder why we don't just use the ServiceType enum instead.
A future enhancement might let a user target a specific node, and the node identifier would likely be part of the HttpTarget.
Also, the Views service requires the bucket name so the SDK can route the request to a node hosting active partitions for the bucket.
Even though we're not supporting the Views service, it's an example of how additional context may be required to target future services.

#### Factory methods / Constructors

1. `HttpTarget.analytics()`
2. `HttpTarget.backup()`
3. `HttpTarget.eventing()`
4. `HttpTarget.manager()`
5. `HttpTarget.query()`
6. `HttpTarget.search()`

### HttpPath

Immutable value object.
Represents the "path" component of a URI.

#### Factory methods / Constructors

1. `HttpPath.of(String template, String... args)`

This method has two parameters: a template string, and a varargs array (or List) of argument strings.

Each `{}` placeholder in the template string is replaced with the URL-encoded form of the corresponding additional argument.
For example:
```
HttpPath.of("/foo/{}/bar/{}", "hello world", "a/b")
```
creates the path `/foo/hello%20world/bar/a%2Fb`

If the number of placeholders in the template string does not match the number of additional arguments, this method throws IllegalArgumentException (or equivalent).

**NOTE:** At the implementor's discretion, a path may be represented as a string instead of an `HttpPath`.
In that case, the implementation should provide a static `formatPath(String template, String... args)` method that performs the placeholder substitution.

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
Represents the body of a POST or PUT request.

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

As a convenience, implementations may add `form` overloads whose signatures match the `NameValuePair` constructors and create the NameValuePair internally before delegating to `form(NameValuePair)`.

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

As a convenience, implementations may add a `contentAsString()` method that interprets the content as a UTF-8 String.

### CouchbaseHttpClient

Instances must be thread-safe.

#### Factory methods / Constructors

Users do not create instances directly.
`Cluster` has a new instance method `httpClient()` that returns a CouchbaseHttpClient.

#### Other methods

Instance methods of CouchbaseHttpClient correspond to the supported HTTP verbs.

```java
class CouchbaseHttpClient {
  HttpResponse get(
        HttpTarget target,
        HttpPath path,
        NameValuePairs queryString, // optional
        List<Header> headers // optional
  );

  HttpResponse post(
      HttpTarget target,
      HttpPath path,
      HttpBody body, // optional
      List<Header> headers // optional
  );

  HttpResponse put(
      HttpTarget target,
      HttpPath path,
      HttpBody body, // optional
      List<Header> headers // optional
  );

  HttpResponse delete(
      HttpTarget target,
      HttpPath path,
      List<Header> headers // optional
  );
}
```

A `Header` is just a name and a value, both strings.
These are additional headers the client will include in the HTTP request.
The documentation for this option should make it clear the user is not responsible for setting the "Content-Type" header.

In addition to the parameters explicitly mentioned above, every method also has the optional parameters common to all SDK 3 management requests.
The set of common parameters may vary by implementation, but typically includes "timeout", "retry strategy", and "parent span".

If the implementation uses the notion of idempotency to control whether a request may be retried, GET requests are considered idempotent, and all other requests are non-idempotent.

### Implementation notes

If the user provides a path that does not start with a slash, an implementation should prepend a slash to make it valid.

If the user provides a path that contains the query string delimiter (`?`), an implementation should use `&` instead of `?` as the delimiter when appending the `queryString` value to the path.
This allows the user to completely opt out of the query string parameter by encoding the query string as part of the path.

For example, the following should be valid:

```
val response = httpClient.get(
    target = HttpTarget.manager(),
    path = HttpPath.of("foo/bar?color={}", red),
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
* Jan 21, 2022 - Revision #2 (by David Nault)
    * Added `HttpTarget.backup()`
    * Added `HttpResponse.contentAsString()`
* Jan 24, 2022 - Revision #3 (by David Nault)
    * Replaced "CommonOptions" with a note that all HTTP client methods have the same optional parameters common to all SDK 3 management requests.
    * Added a note that GET requests should be considered idempotent, and all others are non-idempotent.
    * Added `HttpPath` for representing paths. An implementation may still use a String instead.
* Feb 9, 2022 - Revision #4 (by David Nault)
    * Removed "views" HTTP target because views are on the way out.
    * Removed NodeIdentifier because it was under-specified and specific to the Java SDK.
    * Removed the "nodeIdentifier" and "bucketName" properties from HttpTarget.
    * Added "headers" option to all HTTP client methods, for specifying custom HTTP request headers.
    * Clarified that the client should automatically set the "Content-Type" header based on the HttpBody type.


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
