# Meta

*   RFC Name: SDK3 Analytics
*   RFC ID: 0057-sdk3-analytics
*   Start Date: 2019-09-27
*   Owner: Michael Nitschinger
*   Current Status: ACCEPTED
*   Relates To:
    [0054-sdk3-management-apis](0054-sdk3-management-apis.md),
    [0058-sdk3-error-handling](0058-sdk3-error-handling.md),
    [0059-sdk3-foundation](0059-sdk3-foundation.md)

# Summary

This RFC is part of the bigger SDK 3.0 RFC and describes the Analytics Query APIs in detail.

# Technical Details

## API Entry Point

```
interface ICluster {
  ...
  IAnalyticsResult AnalyticsQuery(string statement, [AnalyticsOptions options]);
  ...
}
```

## ICluster#AnalyticsQuery

The ICluster::AnalyticsQuery method enables a user to perform an analytics query and receive the results in a streaming manner. The SDK should perform this by executing a HTTP GET request against the analytics service.

**Signature:**

```
IAnalyticsResult AnalyticsQuery(string statement, [AnalyticsOptions options]);
```

**Parameters:**

* Required:
  * statement : string
    * Specifies the Analytics statement in string form (i.e. "select foo from bar")
* Optional (part of QueryOptions):
  * scanConsistency(AnalyticsScanConsistency) = undefined(NotBounded)
    * Specifies the level of consistency for the query.
    * Sent within the JSON payload as `scan_consistency` and is encoded as:
      * RequestPlus: `request_plus`
      * NotBounded: `not_bounded`
  * clientContextId(String) = UUID(),
    * Specifies a context ID string which is mirrored back from the query engine on response
    * If not provided, the client must create and use a UUID instead. Sent as in the JSON payload as "client_context_id" as a JSON String `"foobar"`
  * raw(String key, JsonValue value) = undefined
    * Specifies values with their key and value as presented as part of the JSON payload
    * This is an escape hatch to support unknown commands and be forwards compatible
  * readonly(boolean) = undefined(false)
    * Allows to specify if the query is readonly
    * Sent within the JSON Payload as `readonly` as a JSON boolean
  * parameters(JsonObject | JsonArray) = undefined
    * Specifies positional or named parameters
    * For languages that do not support operator overloading, the alternative naming is positionalParameters(JsonArray) and namedParameters(JsonObject)
    * Sent in the JSON payload
      * For positional parameters as a JSON array under the "args" key
      * For named parameters directly in the JSON payload, but each named argument is prefixed with "$" if it doesn't already have the dollar prefix provided by the user
      * The payload can include both positional and named parameters
    * Setting JsonArray overrides any previously set _positional_ parameters
    * Setting JsonObject overrides any previously set _named_ parameters
  * priority(boolean) = undefined(0)
    * Allows to give certain requests higher priority than others.
    * Sent on the wire in the HTTP header as "Analytics-Priority" a JSON Number. See the section on Priority in this RFC for more details how it should be encoded.
  * timeout(Duration) = $Cluster::analyticsOperationTimeout
    * SSpecifies how long to allow the operation to continue running before it is cancelled.
  * serializer(JsonSerializer) = $Cluster::Serializer
    * SSpecifies the serializer which should be used for deserialization of rows returned.

**Returns:**

An `IAnalyticsResult` that maps the result of the analytics query to an object.

**Throws:**

* Documented
  * RequestTimeoutException (#1)
  * CouchbaseException (#0)
* Undocumented
  * RequestCanceledException (#2)
  * InvalidArgumentException (#3)
  * ServiceNotAvailableException (#4)
  * InternalServerException (#5)
  * AuthenticationException (#6)
  * CompilationFailedException (#301)
  * JobQueueFullException (#302)
  * DatasetNotFound (#303)
  * ParsingFailedException (#8)

## Named and Positional Parameter Handling

**Note:** named and positional parameters must not be set at the same time. If both are set the SDK must fail immediately with an IllegalArgumentException or similar.

Named parameters are applied with a prefix "$" to the JSON payload, positional ones as an array in the "args" element.

## AnalyticsScanConsistency

```
enum AnalyticsScanConsistency {
  NotBounded,
  RequestPlus
}
```

In the JSON payload, the field name is `scan_consistency`.

| Identifier  | Wire Format      | Description     |
|-------------|------------------|-----------------|
| NotBounded  | `"not_bounded"`  | No scan consistency is used. This is the default and not sent over the wire when set.  |
| RequestPlus | `"request_plus"` | The indexer will grab the highest seqnos at query time and wait until indexed before returning.   |

## Raw Params

Raw params are the escape hatch for options which are not directly exposed and also future-proofs the API. The API must only accept valid JSON values and/or encoded it into their JSON representations.

## Priority

The server allows the client to specify priority, but it is a numeric value on the wire. So a higher priority means a lower value (i.e. -1 over 0). To make it simpler for a user, this has been mapped to a boolean on the `QueryOptions` API.

So if priority is set to true, it must be written as numeric -1 on the wire.

Also it is important that the priority is NOT part of the json blob, but rather set as a http header!

* Header name: `"Analytics-Priority"`
* Header value: not present if 0 (false), set to -1 if true

## AnalyticsResult

The IAnalyticsResult is returned if the response arrived successfully and contains both the associated metadata and the actual rows.

```
struct AnalyticsResult {
  Stream<T> Rows();
  Promise<AnalyticsMetaData> MetaData();
}
```

Depending on the language best practices, rows() returns a list of entities, but how they are decoded is language-dependent.

```
struct AnalyticsMetaData {
    String RequestId();
    String ClientContextId();
    AnalyticsStatus Status();
    Optional<JsonObject> Signature();
    List<AnalyticsWarning> Warnings();
    AnalyticsMetrics Metrics();
}
```

Only the signature is optional, since it might not be present all the time. 


```
struct AnalyticsMetrics {
  Duration elapsedTime()
  Duration executionTime()
  uint64 resultCount()
  uint64 resultSize()
  uint64 errorCount()
  uint64 processedObjects()
  uint64 warningCount()
}
```

The status needs to be decoded from the wire representation, which is in all cases the lowercase version of the enum. So "success" on the wire is turning into SUCCESS.

```
enum AnalyticsStatus {
    RUNNING,
    SUCCESS,
    TIMEOUT,
    FAILED,
    FATAL,
    UNKNOWN
}
```

A AnalyticsWarning  is always a tuple of code and message:

```
class AnalyticsWarning {
    int32 code()
    String message()
}
```

## Cluster compatibility
With the upcoming release of Enterprise Analytics, and the risk of users accidentally using operational SDKs against it, a compatibility check is being added.

[MB-67103](https://jira.issues.couchbase.com/browse/MB-67103) adds to the cluster and bucket configs a `prodName` string field identifying the cluster type.

Before performing each analytics operation, check the cluster config, and iff the prodName field is present, check if it starts with "Couchbase Server".  If it doesn't, fast-fail the request with a generic `CouchbaseException`.  A suitable error message could be:

```
var errStr = "This '${prodName}' cluster cannot be used with this SDK, which is intended for use with operational clusters."
if (prodName == "Enterprise Analytics") {
  errStr += ". For this cluster, an Enterprise Analytics SDK should be used."
}
throw new CouchbaseException(errStr)
```

If the cluster config is not yet available, wait for it using the request timeout.  On timeout follow standard timeout rules.

For discussion of why "Couchbase Server" is matched, and why we check it starts with rather than the exact string, see the MB.

An SDK-specific 'backdoor' needs to be added to disable the error.  This should not be exposed through the public API, but instead handled via a platform/SDK-specific path - System properties, envvars, undocumented connection string params, or similar.  This is to ease migration of Couchbase tools and drivers that will continue to use the operational SDKs for a time.  This mechanism is not user-facing and should not be documented externally.

# Changelog

* Sept 27, 2019 - Revision #1 (by Michael Nitschinger)
  * Initial Draft
* April 30, 2020
  * Moved RFC to ACCEPTED state.
* May 20, 2024 - Revision #2 (by Dimitris Christodoulou)
  * Correct `AnalyticsStatus` enum values.
* April 30, 2025 - Revision #3 (by Dimitris Christodoulou)
  * Allow setting both positional and named parameters.
* June 19, 2025 - Revision #4 (by Graham Pople)
  * Add check for operational cluster.

# Signoff

| Language   | Team Member         | Signoff Date    | Revision |
|------------|---------------------|-----------------|----------|
| Node.js    | Jared Casey         | May 1, 2025     | #3       |
| Go         | Charles Dixon       | April 22, 2020  | #1       |
| Connectors | David Nault         | May 1, 2025     | #3       |
| PHP        | Sergey Avseyev      | May 7, 2025     | #3       |
| Python     | Jared Casey         | May 1, 2025     | #3       |
| Scala      | Graham Pople        | May 2, 2025     | #3       |
| .NET       | Jeffry Morris       | May 8, 2025     | #3       |
| Java       | David Nault         | May 1, 2025     | #3       |
| Kotlin     | David Nault         | May 1, 2025     | #3       |
| C          | Sergey Avseyev      | May 7, 2025     | #3       |
| C++        | Sergey Avseyev      | May 7, 2025     | #3       |
| Ruby       | Sergey Avseyev      | May 7, 2025     | #3       |
