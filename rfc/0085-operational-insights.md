# Meta

* RFC Name: Operational Insights (previously Enterprise Analytics)
* RFC ID: 85
* Start Date: 2025-04-02
* Owners: Jared Casey
* Current Status: DRAFT
* Revision: #4
* Relates to:
    * [SDK-RFC#79 (Columnar API Foundation)][sdk-rfc-0079]
    * [SDK-RFC#80 (Columnar API Connection Management)][sdk-rfc-0080]
    * [SDK-RFC#81 (Columnar API Streaming)][sdk-rfc-0081]
    * [SDK-RFC#82 (Columnar API Error Handling and Retries)][sdk-rfc-0082]

# Motivation

This document aims to outline the structure of an SDK that supports Couchbase Operational Insights (previously named Enterprise Analytics). The Operational Insights SDK has a simplified connection strategy that removes the necessity for complex topology tracking to enable the SDK to be a thin wrapper around an HTTP client.

NOTE: In October 2026, Enterprise Analytics was renamed to Operational Insights, and the Enterprise Analytics SDK became the Operational Insights SDK. Earlier revisions of this RFC use the Enterprise Analytics name. Linked documents written before the change may still use it.

## Requirements

* The Operational Insights SDK must remain compatible with the API outlined for the Columnar SDK. See the [Columnar API Foundation RFC][sdk-rfc-0079] for details.
* The Operational Insights SDK must not use bootstrapping logic (or topology tracking) required in the Operational and the Columnar SDKs.

## Supporting Documentation

* [Analytics Services 2.0 SDK PRD](https://docs.google.com/document/d/136MqWlahzmDBJkMY5rtCAi8q8dve3VTLOTxwQcQejVs/edit?tab=t.0#heading=h.ei1nt9lpmivn)
* [Enterprise Analytics Migration](https://docs.google.com/document/d/1C2n58xoG8nKhd80HAH-pAERDAFW8QuIeUvrJbLCClSs/edit?tab=t.0)

# Terminology

* SDK3 - refers to the operational (or "classic") SDK
* Columnar SDK - refers to the v1 implementation of an SDK for Capella Columnar (the original implementation is not available for on-prem)
* Operational Insights SDK - refers to the v2 implementation of an SDK for Operational Insights (previously Enterprise Analytics). Nondescript references to "SDK" are synonymous with Operational Insights SDK
* Hostname - Synonymous with the `<host>` part of the HTTP endpoint syntax specified below

## HTTP Endpoint Syntax

Following [Section 3.1: Common Internet Scheme Syntax](https://datatracker.ietf.org/doc/html/rfc1738#section-3.1) of [RFC 1738 Uniform Resource Locators (URL)](https://datatracker.ietf.org/doc/html/rfc1738), the expected HTTP endpoint syntax is:

`<scheme>://<host>:<port>`

The `<user>`, `<password>` and `<url-path>` parts are omitted as the user and password are provided via a Credential (see the [Credential](#credential) section) and the url-path is currently out of scope.

# API

The transition from the Columnar SDK to the Operational Insights SDK should be as straight-forward as possible. However, the Operational Insights SDK is only expected to have the same general API "shape" (creating objects, executing a query, etc.) as the Columnar SDK. The [Columnar API Foundation RFC][sdk-rfc-0079] details the expected "shape" of the Operational Insights SDK's API. The two SDKs are not expected to have "compatible" APIs due to the requirements of the Operational Insights SDK (removal of topology tracking, exception handling, packaging, etc.).

## Management API

A notable departure from the Columnar SDK is the management APIs are currently considered out of scope. The Operational Insights SDK should not expose any portion of the management APIs to users. This is subject to change for future releases of the Operational Insights SDK as deemed necessary.

# Cluster Instance

Cluster creation remains mostly the same as outlined in the v1 [Columnar API Connection Management RFC][sdk-rfc-0080]. Some details are provided below for convenience.

Creating a Cluster instance performs no I/O and is obtained with a method that is language idiomatic and takes 3 parameters:

* `httpEndpoint (String)`
* `credential (Credential)`
* `clusterOptions (ClusterOptions) // optional!`

***IMPORTANT:*** The SDK must not include any logic regarding SDK3/Columnar KV bootstrapping and topology tracking.

# HTTP Endpoint

The Operational Insights SDK only supports HTTP/HTTPS and therefore users should provide only a single endpoint when creating a cluster instance. If multiple endpoints are provided the cluster creation should fail with a platform idiomatic invalid argument.

## Scheme

As Operational Insights will support both on-prem and cloud (eventually) deployments, the Operational Insights SDK should support both a secure (https) and insecure (http) scheme.

If either a scheme *is not* provided or an unsupported scheme *is* provided, the cluster creation method should fail with a platform idiomatic invalid argument.

## Port

If not specified, the default port should match the scheme:

* http: 80
* https: 443

The SDK must override the default port if a port is provided in the connection string.

## Parameters

In addition to the connection string parameters specified in the v1 [Columnar API Connection Management RFC][sdk-rfc-0080], the Operational Insights SDK should include the `MaxRetries` cluster option as a recognized connection string parameter (e.g. `max_retries`).

# Credential

The Operational Insights SDK uses a `Credential` abstraction similar in spirit to SDK3's `Authenticator`. An SDK must support three kinds of credentials:

* **Username and password.** Sets an "Authorization" header on every HTTP request using the "Basic" scheme. See [Basic access authentication](https://en.wikipedia.org/wiki/Basic_access_authentication). Username and password are encoded as UTF-8.
* **JSON Web Token (JWT).** Sets an "Authorization" header on every HTTP request using the "Bearer" scheme. Example:
    * `Authorization: Bearer <jwt>`
* **Client certificate (mTLS).** Instead of setting HTTP headers, the SDK authenticates the client during the TLS handshake. The SDK should support reading the certificate and private key from a PKCS#12 file.

This RFC does not specify the syntax for creating a Credential instance. Implementors should do whatever is idiomatic for their platform. For example, in the Java implementation the user creates a username & password credential like this:

```java
Credential usernameAndPassword = Credential.of("groucho", "swordfish");
```

… and a JWT credential like this:

```java
Credential jwt = Credential.ofJwt("xxxxx.yyyyy.zzzzz");
```

… and a client certificate credential like this:

```java
Credential clientCertificate = Credential.fromKeyStore(
  Paths.get("/path/to/client-cert.p12"),
  "My password"
);
```

## Updating the credential

After the cluster is created, the user should be able to supply a new credential by calling `cluster.credential(newCredential)`. The new credential should be used for all subsequent HTTP requests.

It must be safe for the user to call this method at any time from any thread. This method should throw `IllegalStateException` (or the idiomatic equivalent) if the new credential is not the same type as the old credential.

# ClusterOptions

Optional ClusterOptions may be provided when creating a cluster instance.

A notable addition to the Operational Insights SDK is the `MaxRetries` option. The option should also be included as a recognized connection string parameter. Note that the option should be marked as volatile.

```
type ClusterOptions {
    /// The default row deserializer, if applicable
    Deserializer(Deserializer) ClusterOptions

    /// Security configuration options
    Security(SecurityOptions) ClusterOptions

    /// Timeout configuration options
    Timeout(TimeoutOptions) ClusterOptions

    /// Maximum number of attempts to retry
    /// Default = 7
    /// **VOLATILE**
    MaxRetries(int) ClusterOptions
}
```

## TimeoutOptions

Optional TimeoutOptions may be provided as part of configuring a cluster's behavior.

```
type TimeoutOptions {
    /// Socket connection timeout, or more broadly the timeout
    /// for establishing an individual authenticated connection.
    /// Default = 10 seconds
    ConnectTimeout(Duration) TimeoutOptions

    /// Timeout applied to HTTP requests in the server async query
    /// requests API. For FetchResults, this is the default for
    /// FetchResultsOptions.Timeout.
    /// Default = 10 seconds
    HandleRequestTimeout(Duration) TimeoutOptions

    /// Columnar query timeout.
    /// Default = 10 minutes
    QueryTimeout(Duration) TimeoutOptions
}
```

## SecurityOptions

Optional SecurityOptions may be provided as part of configuring a cluster's behavior.

Notable departures from the Columnar SDK:

* The Operational Insights SDK must support both on-prem and cloud deployments which means the default for the Operational Insights SDK is to trust ***both*** the Capella Certificate Authority (CA) and other well-known CAs from the platform runtime.
* The Operational Insights SDK does not (currently) provide an option to set Cipher Suites and should only use TLS 1.3.

Except for `DisableServerCertificateVerification`, all other SecurityOptions must clear any existing trust settings prior to being applied.

```
type SecurityOptions {
    /// Clears any existing trust settings, and tells the SDK
    /// to trust only the Capella CA certificate(s) bundled with
    /// the SDK.
    TrustOnlyCapella() SecurityOptions

    /// Clears any existing trust settings, and tells the SDK to
    /// trust only the PEM-encoded certificate(s) in the file at
    /// the given FS path.
    TrustOnlyPemFile(FilesystemPath) SecurityOptions

    /// Clears any existing trust settings, and tells the SDK to
    /// trust only the PEM-encoded certificate(s) in the given
    /// string.
    TrustOnlyPemString(String) SecurityOptions

    /// Clears any existing trust settings, and tells the SDK to
    /// trust only the specified certificates.
    TrustOnlyCertificates(List<X509Certificate>) SecurityOptions

    /// If true, the SDK trusts ANY certificate regardless of
    /// validity.
    /// Impl Note: This is a separate property because a user
    /// should be able to set this to false and then back to
    /// true without losing the previous certificate trust
    /// settings.
    /// Default = false
    DisableServerCertificateVerification(Boolean) SecurityOptions
}
```

### TLS

The Operational Insights SDK must support both on-prem and cloud deployments, and therefore it must trust both the Capella Certificate Authority (CA) and other well-known CAs. Users are able, and should be encouraged, to configure the SDK in a manner that limits the scope of trusted certificates.

#### Disabling server certificate verification

The Operational Insights SDK should allow the cluster to be configured to disable server certificate verification.

The SDK should apply the same behavior outlined in the v1 [Columnar API Connection Management RFC][sdk-rfc-0080] and log an error if a user has configured the cluster to disable server certificate verification.

#### Non-prod certificate

The Operational Insights SDK may provide a simple way for internal users to trust the Couchbase non-prod CA certificate(s). If the SDK provides this functionality the SDK should follow the specification outlined in the v1 [Columnar API Connection Management RFC][sdk-rfc-0080].

## HTTP Client Tunables

In order to keep configuration of the Operational Insights SDK simple, it is recommended that the SDK avoid exposing configuration options from the underlying HTTP client. The assumption is that the preferred HTTP client used by each SDK should have reasonable defaults that will suffice for the majority of analytical workloads.

SDKs are free to adapt their HTTP client's defaults to better align with analytical workloads where applicable. However, the SDK should not expose configuration tunables to users initially, unless deemed necessary. Configuration tunables that are exposed to users can evolve as needed with future releases of the SDK.

# Sending Requests

The Operational Insights SDK is meant to be a lightweight wrapper around each language's preferred HTTP client.

The SDK must ensure the following either through configuration of the HTTP client or confirmed default behavior of the HTTP client.

1. For each request, the same A/AAAA record ***MUST NOT*** be used over and over (e.g. each request ***MUST*** use a random selection from the A/AAAA records for that hostname).
    1. See the [Retries](#retries) section for details on how the SDK should handle A/AAAA records on failures.
2. The hostname used by the HTTP client must match that of the original hostname given by the user (at client construction time), not the resolved DNS record. This is to enable proper TLS hostname verification to occur as well as to enable proper virtual-host routing on intermediary proxy servers.
    1. See the [Terminology](#terminology) section for hostname specifications.
3. Requests ***MUST*** include an Authorization header based on the user-provided `Credential`, unless the credential is a client certificate (see the [Credential](#credential) section).

## Query Options

Notable departures from the Columnar SDK:

* `QueryOptions` ***must not*** include a `Priority` option.
* `QueryOptions` ***must*** include a `ClientContextID` option.
* `QueryOptions` ***must*** include a `MaxRetries` option. If not set, defaults to the `MaxRetries` value set at the cluster level. Note that the option should be marked volatile.

The `QueryOptions` block is provided below for convenience. See the [Columnar API Foundation RFC][sdk-rfc-0079] for details.

```
type QueryOptions {
    ClientContextID(string) QueryOptions
    Parameters(object) QueryOptions
    Parameters(array) QueryOptions
    ReadOnly(bool) QueryOptions
    ScanConsistency(QueryScanConsistency) QueryOptions
    Raw(object) QueryOptions
    Timeout(duration) QueryOptions
    MaxRetries(int) QueryOptions // **VOLATILE**
    Deserializer(Deserializer) QueryOptions
}
```

## Response Streaming

The Operational Insights SDK must provide users with the ability to stream results. The SDK should follow the specification outlined in the [Columnar API Streaming RFC][sdk-rfc-0081].

# Error Handling and Retries

The Operational Insights SDK should follow the Columnar SDK in promoting consistency across SDKs while also allowing SDKs to align with their platform idioms.

## Exceptions

The Operational Insights SDK must provide a base exception with the subclasses of exceptions detailed below.

All exceptions should derive/implement the platform idiomatic representation (class, structure, etc.).

NOTE: SDKs may choose to adapt the exception naming convention to align with platform idioms.

### OperationalInsightsException

The base exception should derive from whatever system level generic exception/error is available. The purpose of the base exception is to distinguish an error handled and perhaps thrown by the SDK and a system or platform idiomatic error/exception.

The base exception must be the word "OperationalInsights" or "Insights" with the platform idiomatic name and casing for an exception/error appended (e.g. `OperationalInsightsException`, `OperationalInsightsError`, `InsightsException`, `InsightsError`, etc.). This RFC uses `OperationalInsightsException` to refer to the base exception.

> [!NOTE]
> SDKs released under the Enterprise Analytics name (before the October 2026 name change) used the word "Analytics" for the base exception (e.g. `AnalyticsException`, `AnalyticsError`). The Operational Insights SDKs replace it with the "OperationalInsights" or "Insights" form above. Both forms are allowed because the Operational Insights SDKs shipped with each: Python and Node.js use `OperationalInsightsError`, while Java uses `InsightsException` and Go uses `InsightsError`.

```py
class OperationalInsightsError(Exception):
    ...
```

### InvalidCredentialException

Thrown if the analytics service returns HTTP status code 401 or analytics error code 20000.

```py
class InvalidCredentialError(OperationalInsightsError):
    """
     Indicates the analytics service returned an HTTP status code 401 or analytics
     error code 20000
    """
```

### TimeoutException

Thrown if the SDK detects a client-side timeout, or the server returns analytics error code 21002 (server-side timeout).

NOTE: This is the user's only indication that a request is potentially retriable, since the SDK automatically retries retriable operations until the timeout duration elapses.

```py
class TimeoutError(OperationalInsightsError):
    """
     Indicates the SDK has detected a client-side timeout or the analytics service
     returned error code 21002.
    """
```

### QueryException

Thrown if the analytics service returns a response with an error code other than 20000 or 21002.

The QueryException should provide two properties:

* `code` (integer): error code from analytics service
* `serverMessage` (string): human readable message from analytics service

NOTE: The API documentation for this exception should indicate most users are not expected to take programmatic action based on the error code, but the code is programmatically accessible to support advanced use cases.

```py
class QueryError(OperationalInsightsError):
    """
     Indicates the analytics service returned a response with an error code other than
     20000 or 21002.
    """
    @property
    def code(self) -> int:
        """
         Error code from analytics service.
        """
    @property
    def server_message(self) -> str:
        """
         Response message from analytics service.
        """
```

### QueryNotFoundException

Thrown if the analytics service returns HTTP status code 404. Currently only applies to the server async query requests API.

NOTE: `QueryNotFound` is not raised in all scenarios where the SDK might receive a HTTP status code 404. See the API methods below for details on when `QueryNotFound` should be raised.

```py
class QueryNotFoundError(OperationalInsightsError):
    """
     Indicates the analytics service returned a HTTP Status 404 Not Found.
    """
```

### Custom and Platform Exceptions

The Operational Insights SDK should follow the same guidelines outlined in the [Columnar API Error Handling and Retries RFC][sdk-rfc-0082].

## HTTP Response Body

When handling errors from the analytics service, the expectation is that the response body will be JSON compliant and contain an `errors` array. If the response body is not JSON compliant or does not contain an `errors` array, an `OperationalInsightsException` should be raised.

Example of a minimal analytics service response indicating an error:

```json
{
  "errors": [
    {
      "code": 123,
      "msg": "Oh no, not again.",
      "retriable": false
    }
  ]
}
```

## Exceptions from HTTP library

All exceptions thrown by the HTTP library must not be propagated to the user. Exceptions should be handled internally and translated to the appropriate platform or `OperationalInsightsException`. In the case where the HTTP library's exception should be translated to an `OperationalInsightsException` the HTTP library's exception should be a part of the error context.

## Error Context

The Operational Insights SDK should follow the same guidelines outlined in the [Columnar API Error Handling and Retries RFC][sdk-rfc-0082].

## Retries

See [Appendix A](#appendix-a-retry-logic-pseudo-code) for a conceptual guide on the retry logic.

The Operational Insights SDK should retry a request when the request is eligible to retry and the number of request attempts has not exceeded the `MaxRetries` limit. The SDK should calculate an exponential back-off to use as a delay between each retry attempt (see the [Backoff Calculation](#backoff-calculation) section for more details).

### MaxRetries Limit

The `MaxRetries` limit defaults to 7 and is user configurable at both the cluster and request level.

The `MaxRetries` limit applies only to retries on top of the initial attempt. For instance, a request that retries on a connection failure and then retries from a query error means the next retry attempt is 2 (0: initial attempt with a connection failure, 1: next attempt with query error; 2: next attempt).

### Retry Eligibility

A request is eligible for retry when the request falls into one of the categories below:

* DNS failures and Connection failures
    * Connection failures specifically refers to failures when dialing the TCP connection, not any read/write errors thereafter. This would include a connection timeout.
* Analytics service error response where all errors indicated to be retriable
    * The SDK should follow the retry guidelines outlined in the [Columnar API Error Handling and Retries RFC][sdk-rfc-0082] to determine if a response from the analytics service is eligible for retries.
* 503 HTTP Status Code

NOTE: Credential errors are returned by the analytics service as a normal non-retriable error and thus should not be retried by the SDK (this behavior is notably different from Operational SDKs).

### Retry Limit Exceeded

See [Appendix A](#appendix-a-retry-logic-pseudo-code) for a conceptual guide on the retry logic.

If a request fails due to exceeding the `MaxRetries`, the SDK should raise the error from the most recent attempt. If the error to be raised is an `OperationalInsightsException` the error context should include the error from the most recent attempt.

If possible, the number of attempts should be included as part of the error context.

### Timeout Exceeded

See [Appendix A](#appendix-a-retry-logic-pseudo-code) for a conceptual guide on the retry logic.

If a request fails due to exceeding the `QueryTimeout`, a `TimeoutException` should be raised. The error context should include the error from the most recent attempt. If possible, the number of attempts should also be included as part of the error context.

### HTTP Status Codes

As the Operational Insights SDK is a lightweight wrapper over an HTTP client there are certain HTTP status codes that should be retried:

#### 5xx: Server error

[503 Service Unavailable](https://http.dev/503)

### Backoff Calculation

In between retries, the SDK should use a capped exponential backoff with full jitter, starting at 100 milliseconds, with an exponent of 2 and capped at 1 minute. If the calculated backoff would exceed the timeout deadline, the SDK should immediately report a timeout instead of waiting for the backoff duration to elapse.

# Server Async Requests SDK API

## Overview

* Provide an API that intuitively "guides" the end user in the correct direction when working through the various stages of the async operation.
    * Note that an API for server asynchronous operations is a new concept for SDKs.
* The API provides objects that represent the various "stages" in the path of query execution on the server (or at least the "stages" relevant to the client). Each object provides a mechanism to move to the next "stage" once the next "stage" becomes available.
* The API is an extension of the previously defined API. It is not a stand-alone API or new client.

### Supporting Documentation

* [Async Server Requests API PRD](https://docs.google.com/document/d/1Ce8ME8w_13D_S2-tdrDqdoChNZPB1rCKA2k6VHoujNA/edit?tab=t.0)
* [Appendix B: Asynchronous Server Request curl Commands](#appendix-b-asynchronous-server-request-curl-commands)

## Cluster and Scope

The server async request API adds an additional method on the cluster and scope objects.

```
type [Cluster | Scope] {
    // ... previous defined logic
    StartQuery(statement string, [options StartQueryOptions]) QueryHandle
}
```

### StartQuery

#### Signature

`StartQuery(statement string, [options StartQueryOptions]) QueryHandle`

#### Implementation

**Request**

Sends an HTTP request to the server. The operation must include `mode` in the request body. The value of the mode must be set to `async`. The operation must return a new `QueryHandle` object.

**StartQueryOptions**

The `StartQueryOptions` block should include the same options provided in `QueryOptions` except for the `Deserializer` which must not be included. `StartQueryOptions` must not inherit from `QueryOptions`.

See the [Columnar API Foundation RFC][sdk-rfc-0079] for details around other options in the `StartQueryOptions`.

```
type StartQueryOptions {
    ClientContextID(string) StartQueryOptions
    Parameters(object) StartQueryOptions
    Parameters(array) StartQueryOptions
    ReadOnly(bool) StartQueryOptions
    ScanConsistency(QueryScanConsistency) StartQueryOptions
    Raw(object) StartQueryOptions
    Timeout(duration) StartQueryOptions
    MaxRetries(int) StartQueryOptions // **VOLATILE**
}
```

`StartQueryOptions.Timeout` is the server-side execution timeout for the query. It does not change client-side timeouts for any server async API operation. Client-side timeouts are set by `TimeoutOptions.HandleRequestTimeout` and, for FetchResults, `FetchResultsOptions.Timeout`.

**Response**

The SDK should use the `requestID` and the `handle` properties in the response when creating the `QueryHandle` object to return.

Other information in the response (e.g. metrics) are not currently used.

**Errors and Retries**

Errors and Retries must work the same as errors and retries for `ExecuteQuery()`. See the [Error Handling and Retries](#error-handling-and-retries) section for details.

If the server provides a response that is not indicated as an error and does not include the `requestID` or the `handle`, an `OperationalInsightsException` should be raised. The error message should indicate which required properties are missing.

##### Headers

`content-type: application/json`

##### Method

`POST`

##### Path

`/api/v1/request`

##### Response

```json
{
  "requestID": "8fb94489-dcd7-466d-8639-cb02144c3f79",
  "status": "queued",
  "handle": "/api/v1/request/status/8fb94489-dcd7-466d-8639-cb02144c3f79/12-0",
  "plans": {},
  "metrics": {
    "elapsedTime": "30.061476ms",
    "compileTime": "11.792157ms",
    "queueWaitTime": "0ns"
  }
}
```

Following the example response body, the `QueryHandle` object should have the following for its `RequestId` and `Handle` properties:

* `QueryHandle.RequestId` = `8fb94489-dcd7-466d-8639-cb02144c3f79`
* `QueryHandle.Handle` = `/api/v1/request/status/8fb94489-dcd7-466d-8639-cb02144c3f79/12-0`

## QueryHandle

```
type QueryHandle {
    // Properties
    // NOTE:  These are private
    handle string
    requestId string

    // Methods
    FetchStatus([options FetchStatusOptions]) QueryStatus
    Cancel([options CancelOptions])
}
```

### FetchStatus

#### Signature

`FetchStatus([options FetchStatusOptions]) QueryStatus`

#### Implementation

**Request**

Sends an HTTP request to the server to get the status of the query.

**FetchStatusOptions**

Optional options block that can be used as placeholder for potential options in the future.

**Response**

The operation should return a new `QueryStatus` instance unless an exception should be raised. See the Errors and Retries section below for details on when the SDK should raise an exception.

**Errors and Retries**

If the server provides a response that indicates an error has occurred, the expectation is that the response body will contain an `errors` array. The `errors` array should be processed and handled as outlined in the [Error Handling and Retries](#error-handling-and-retries) section.

If the response body is not JSON compliant or does not contain an `errors` array, an `OperationalInsightsException` should be used.

If a HTTP Status Code `404 Not Found` is received, the SDK should raise a `QueryNotFoundException`.

**Other**

Other information in the response (metrics, partitions, etc.) are only represented in the `QueryStatus.ToString()` and are otherwise not used. See [QueryStatus](#querystatus) for more information.

##### Method

`GET`

##### Path

`/api/v1/request/status/{requestID}/{handleID}`

##### Response

**Running:**

**NOTE**: Must be using a 2.2.x version of the server otherwise only the `status` property is returned.

```json
{
  "status": "running",
  "metrics": {
    "elapsedTime": "10.582s",
    "executionTime": "10.477s",
    "compileTime": "105ms",
    "queueWaitTime": "0ns"
  },
  "createdAt": "2026-04-15T23:22:19.246Z"
}
```

**Success:**

```json
{
  "status": "success",
  "handle": "/api/v1/request/result/8fb94489-dcd7-466d-8639-cb02144c3f79/12-0",
  "resultCount": 100,
  "partitions": [
    {
      "handle": "/api/v1/request/result/8fb94489-dcd7-466d-8639-cb02144c3f79/12-0/0",
      "resultCount": 100
    }
  ],
  "resultSetOrdered": false,
  "metrics": {
    "elapsedTime": "607.586µs",
    "executionTime": "50.195959551s",
    "compileTime": "11.792157ms",
    "queueWaitTime": "0ns",
    "processedObjects": 0
  },
  "createdAt": "2026-03-16T17:15:40.850Z"
}
```

**Fatal:**

```json
{
  "status": "fatal",
  "errors": [
    {
      "code": 23034,
      "msg": "Insufficient memory is provided for the join operators, please increase the join memory budget"
    }
  ],
  "metrics": {
    "elapsedTime": "3.1s",
    "executionTime": "2s",
    "queueWaitTime": "1s"
  },
  "createdAt": "2025-09-23T01:01:01Z"
}
```

**Timeout:**

```json
{
  "status": "timeout",
  "errors": [
    {
      "code": 21002,
      "msg": "Request timed out",
      "retriable": true
    }
  ],
  "metrics": {
    "elapsedTime": "3.1s",
    "executionTime": "2s",
    "queueWaitTime": "1s"
  },
  "createdAt": "2025-09-23T01:01:01Z"
}
```

The status types the server currently returns are (see the server's [ResultStatus](https://github.com/apache/asterixdb/blob/30164fe4b2b72ba0caac6eeed80b4a370a1bdb48/asterixdb/asterix-app/src/main/java/org/apache/asterix/api/http/server/AbstractQueryApiServlet.java#L57) enum):

* `queued`
* `running`
* `success`
* `fatal`
* `failed`
* `timeout`

### Cancel

#### Signature

`Cancel([options CancelOptions])`

#### Implementation

Sends an HTTP request to the server. The operation should set the `request_id` to the `QueryHandle.RequestId` in the request body. It is also possible to set the `client_context_id` to the query's `client_context_id`, but `request_id` is preferred for the asynchronous server requests API.

**CancelOptions**

Optional options block that can be used as placeholder for potential options in the future.

##### Headers

`content-type: application/x-www-form-urlencoded`

##### Method

`DELETE`

##### Path

`/api/v1/active_requests/`

##### Response

A server response with a JSON body is not expected. The SDK should handle the following HTTP status codes:

* `2xx` & `404`: Success, SDK does nothing.
* `503`: SDK should retry request (should be able to use previously defined retry logic).
* `401`: SDK should raise an `InvalidCredentialException`.

If the HTTP request times out, the SDK should raise a `TimeoutException`.

For any other error scenario, the SDK should raise an `OperationalInsightsException`.

## QueryStatus

```
type QueryStatus {
    ResultsReady() bool
    ResultHandle() QueryResultHandle
    // ToString() should be the canonical mechanism an SDK's language uses to
    // provide a string representation of an object.
    ToString() string
}
```

### ResultsReady

#### Signature

`ResultsReady() bool`

#### Implementation

Returns `true` if and only if the response from the `/api/v1/request/status/` HTTP request provides a handle for the SDK to retrieve the query results. Otherwise returns `false`. Must not perform network calls.

### ResultHandle

#### Signature

`ResultHandle() QueryResultHandle`

#### Implementation

The operation should return a new `QueryResultHandle` instance. Must not perform network calls.

If results are not ready, the SDK should raise a platform idiomatic exception if clearly appropriate (e.g. `IllegalState`, etc.). Otherwise an `OperationalInsightsException` should be raised.

### ToString

#### Signature

`ToString() string`

#### Implementation

The operation should return a string representation of the query status. Must not perform network calls.

The string should include at least the following properties from the server response if they were provided.

* `status`
* `metrics`

## QueryResultHandle

```
type QueryResultHandle {
    // Properties
    // NOTE:  These are private
    requestId string

    // Methods
    FetchResults([options FetchResultsOptions]) QueryResult

    // For languages where a callback based approach to streaming rows
    // makes more sense.
    // An SDK must not implement both FetchResults() and StreamResults()
    // StreamResults(callback, [options FetchResultsOptions])

    DiscardResults([options DiscardResultsOptions])
}
```

### FetchResults

#### Signature

`FetchResults([options FetchResultsOptions]) QueryResult`

#### Implementation

**Request**

Sends an HTTP request to the server.

**FetchResultsOptions**

```
type FetchResultsOptions {
    Deserializer(Deserializer) FetchResultsOptions

    /// Maximum time to wait for the server to begin responding.
    /// See Timeouts below.
    /// Default = TimeoutOptions.HandleRequestTimeout (10 seconds)
    Timeout(Duration) FetchResultsOptions
}
```

**Response**

The SDK should return a `QueryResult` which is the mechanism the application will use to stream query rows. The behavior of the `QueryResult` (and the response from the server) should be the same as in the synchronous query API, except for timeouts (see below).

**Timeouts**

`FetchResultsOptions.Timeout` applies from sending the request until the SDK receives the start of the response. If the response has not started within `Timeout`, the SDK should raise a `TimeoutException`. Retries during this phase count toward the same `Timeout`. After the response starts, each SDK should rely on its HTTP client's idle or read timeout so a stalled server does not block the application indefinitely.

Unlike the synchronous query API, the stream as a whole is not bounded by a client-side timeout. The query has already completed on the server, and the application controls how quickly results are consumed. This is an intentional exception to the Timeout section of the [Columnar API Streaming RFC][sdk-rfc-0081]. Applications must close or cancel the `QueryResult` when they stop consuming it, and SDKs should release the underlying connection when this happens. An application that keeps an open `QueryResult` without consuming or closing it holds its connection indefinitely. A client-side limit on the whole stream may be added later if users need one (see Out of Scope).

**Errors and Retries**

If the server provides a response that indicates an error has occurred, the expectation is that the response body will contain an `errors` array. The `errors` array should be processed and handled as outlined in the [Error Handling and Retries](#error-handling-and-retries) section.

If the response body is not JSON compliant or does not contain an `errors` array, an `OperationalInsightsException` should be used.

If a HTTP Status Code `404 Not Found` is received, the SDK should raise a `QueryNotFoundException`.

**Other**

Metadata and metrics should be handled the same as in the synchronous query API. The `QueryResultHandle` should already have access to the `requestID` so it is not expected that the server response contains the `requestID`.

##### Method

`GET`

##### Path

`/api/v1/request/result/{requestID}/{handleID}`

##### Response

```
{
  "results": [ ... ],
  "metrics": {
    "elapsedTime": "72.323µs",
    "executionTime": "50.12914524s",
    "compileTime": "12.096728ms",
    "queueWaitTime": "0ns",
    "resultCount": 100,
    "resultSize": 300,
    "processedObjects": 0
  },
  "createdAt": "2026-03-16T20:34:45.861Z"
}
```

### DiscardResults

#### Signature

`DiscardResults([options DiscardResultsOptions])`

#### Implementation

Sends an HTTP request to the server.

**DiscardResultsOptions**

Optional options block that can be used as placeholder for potential options in the future.

##### Method

`DELETE`

##### Path

`/api/v1/request/result/{requestID}/{handleID}`

##### Response

A server response with a JSON body is not expected. The SDK should handle the following HTTP status codes:

* `2xx` & `404`: Success, SDK does nothing.
* `503`: SDK should retry request (should be able to use previously defined retry logic).
* `401`: SDK should raise an `InvalidCredentialException`.

If the HTTP request times out, the SDK should raise a `TimeoutException`.

For any other error scenario, the SDK should raise an `OperationalInsightsException`.

# SDK Specifics

## Go

* Go exposes an `ErrServiceUnavailable` error value to surface the 503 HTTP response case.
    * Non-2xx status codes are not error cases from the Go http library.
* Go does not expose a Timeout on QueryOptions, instead utilizing `context.Context` for timeouts/cancellation - as is idiomatic to the language.

# Out of Scope

The following topics are considered to be out of scope until future iterations of the Operational Insights SDK determines the need for the feature based on market and/or user demand.

* HTTP/2
* Cipher Suites
* QueryException subclasses (Management API Exceptions, etc.)
* Management APIs

## Server Async Query Requests

* Tunable `resultTTL`
* A client-side timeout on the stream as a whole for `FetchResults`
* Serialization and result partitions
    * NOTE: We should think through serialized scope-level query handles that are then deserialized on the cluster (and vice-versa)

# Changelog

* 2025-07-09 - Revision #1 (Jared Casey)
* 2026-05-01 - Revision #2 (Jared Casey)
    * Added Server Asynchronous Requests API
* 2026-10-30 - Revision #3 (Jared Casey)
    * Converted from Google Doc to Markdown.
    * Renamed from Enterprise Analytics to Operational Insights (October 2026 name change).
    * Base exception now uses the "OperationalInsights" or "Insights" prefix instead of "Analytics".
* 2026-11-06 - Revision #4 (Jared Casey)
    * Added `Timeout` to `FetchResultsOptions`. It bounds the time to first byte when starting to stream rows in the server async query requests API.
    * Clarified that `StartQueryOptions.Timeout` is the server-side execution timeout only.
    * Added a client-side timeout for the whole `FetchResults` stream to Out of Scope.

# Signoff

| Language | Team Member | Signoff Date | Revision |
| :---- | :---- | :---- | :---- |
| Node.js | Jared Casey | | |
| Go | Charles Dixon | | |
| PHP | Sergey Avseyev | | |
| Python | Jared Casey | | |
| Scala | Graham Pople | | |
| .NET | Jeffry Morris | | |
| Java | David Nault | 2025-07-09 | 1 |
| C++ | Sergey Avseyev | | |
| Ruby | Sergey Avseyev | | |
| Kotlin | David Nault | 2025-07-09 | 1 |

# Appendix A: Retry Logic Pseudo Code

Note that the pseudo code below is not specifying how the SDK should implement the retry logic. Instead it is meant to provide a framework to understand how the SDK should approach retries.

```ts
class OperationalInsightsError {
  // This may not be a field that is publicly exposed, but
  // must be included in the error context somehow.
  cause: Error;
}

class TimeoutError extends OperationalInsightsError {
  // This may actually be represented through error context
  // or through stringification in some SDKs.  It is included
  // in this pseudocode to make it easier to reason about below.
  lastError: Error;
}

function isRetriable(err: Error): boolean {
  // - DNS failures and Connection failures are retriable
  // - Query responds, and response has retriable=true on all errors is retriable.
  // - 503 Status Code is retriable
  // - Otherwise, not retriable

  // NOTES:
  // Credential errors are returned by query as a normal error with
  //   retriable=false, and thus should not be retried by the SDKs
  //   (contrary to the behaviour in Operational).
  // Connection failures specifically refers to failures when dialing
  //   the TCP connection, not any read/write errors thereafter.
}

function query(...): any {
  // maxRetries should be 6 for 7 total attempts, note the use of <=
  //   in the for loop which is important to avoid overflowing in the
  //   'retry forever' use case (don't use <maxRetries+1).
  let maxRetries = 6;

  let lastErr = null;
  for (let retryIdx = 0; retryIdx <= maxRetries; ++retryIdx) {
    try {
      return this._query(...);
    } catch (e) {
      if (e instanceof TimeoutError) {
        e.lastError = lastErr;
        throw e;
      }

      if (!isRetriable(e)) {
        throw e;
      }

      lastErr = e;
    }

    let nextAttemptTime = calculateBackoffRetry(...)
    if (nextAttemptTime >= requestDeadline) {
      throw TimeoutError({
        lastError: lastErr,
      });
    }

    sleepUntil(nextAttemptTime);
  }

  throw lastErr;
}
```

# Appendix B: Asynchronous Server Request curl Commands

## Setup (save some typing)

### Save your endpoint

```sh
export ENDPOINT=https://192.168.107.129:18095
```

### Save the query

```sh
cat << EOF > payload.json
{
  "statement": "SELECT VALUE SLEEP(\"x\", 500) FROM RANGE(1, 100) AS id;",
  "mode": "async"
}
EOF
```

## Start a query

```sh
curl -v -k -u Administrator:password -H "content-type: application/json" -d @payload.json $ENDPOINT/api/v1/request
```

If you want to save the handle (reduce even more typing):

```sh
export HANDLE=$(
  curl -s \
    -X POST \
    -u Administrator:password \
    -H "content-type: application/json" \
    -d @payload.json \
    $ENDPOINT/api/v1/request \
  | jq -r 'select(.status == "queued") | .handle | ltrimstr("/api/v1/request/status/")'
)
```

Can also save the `requestID` (useful for cancelling the query):

```sh
export REQUEST_ID="${HANDLE%%/*}"
```

If you want to add a `client_context_id` (can also use `export CCID=$(uuidgen)`):

```sh
export CCID=$(openssl rand -hex 16)

export RAW_RESPONSE=$(jq --arg id "$CCID" '.client_context_id = $id' payload.json | \
  curl -s -k -u Administrator:password \
  -H "content-type: application/json" \
  $ENDPOINT/api/v1/request \
  -d @-)

export STATUS=$(echo "$RAW_RESPONSE" | jq -r '.status')
export HANDLE=$(echo "$RAW_RESPONSE" | jq -r 'select(.status == "queued") | .handle | ltrimstr("/api/v1/request/status/")')
```

## Get the query status

If you didn't save the handle, refer back to the start query response body.

```sh
curl -v -k -u Administrator:password $ENDPOINT/api/v1/request/status/$HANDLE
```

## Get the query results

If you didn't save the handle, refer back to the start query response body.

```sh
curl -v -k -u Administrator:password $ENDPOINT/api/v1/request/result/$HANDLE
```

## Discard the query results

If you didn't save the handle, refer back to the start query response body.

```sh
curl -v -k -X DELETE -u Administrator:password $ENDPOINT/api/v1/request/result/$HANDLE
```

## Get active requests

```sh
curl -v -k -u Administrator:password $ENDPOINT/api/v1/active_requests
```

## Cancel the query

Using the `requestID` (if you didn't save the `requestID`, refer back to the start query response body):

```sh
curl -X DELETE -v -k -u Administrator:password -H "content-type: application/x-www-form-urlencoded" $ENDPOINT/api/v1/active_requests -d "request_id=$REQUEST_ID"
```

Using the `client_context_id` (if you didn't save the `CCID`, refer to what was sent in the start query request):

```sh
curl -X DELETE -v -k -u Administrator:password -H "content-type: application/x-www-form-urlencoded" $ENDPOINT/api/v1/active_requests -d "client_context_id=$CCID"
```

[sdk-rfc-0079]: https://docs.google.com/document/d/1BzNX8B5XqcDxn8NAi8lVrxP257IdG5kaOsA3mwDMLyQ
[sdk-rfc-0080]: https://docs.google.com/document/d/11LRvVUGt3fEZt2K3CmxwtQQ0QHYW6hZO733cXPtxHVE
[sdk-rfc-0081]: /rfc/0081-columnar-api-streaming.md
[sdk-rfc-0082]: https://docs.google.com/document/d/12WJ4i-B9CcNjIq5NnaFXbxZQfaaNm_tzemrmlRl5YcE
