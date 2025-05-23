# Meta

*   RFC Name: SDK3 Query
*   RFC ID: 0056-sdk3-query
*   Start Date: 2019-09-27
*   Owner: Michael Nitschinger
*   Current Status: ACCEPTED
*   Relates To:
    [0054-sdk3-management-apis](0054-sdk3-management-apis.md),
    [0058-sdk3-error-handling](0058-sdk3-error-handling.md),
    [0059-sdk3-foundation](0059-sdk3-foundation.md)

# Summary

This RFC is part of the bigger SDK 3.0 RFC and describes the N1QL Query APIs in detail.

# Technical Details

## API Entry Point

The Query API is located at Cluster level:

```
interface ICluster {
    ...
    QueryResult Query(string statement, [QueryOptions options]);
    ...
}
```

And at the Scope level:

```
interface IScope {
  ...
  QueryResult Query(string statement, [QueryOptions options]);
  ...
}
```

## ICluster#Query

The `ICluster::Query` method enables a user to perform a N1QL query and receive the results in a streaming manner. The SDK should perform this by executing a HTTP GET request against the query service.

**Signature:**

```
QueryResult Query(string statement, [QueryOptions options]);
```

**Parameters:**

* Required:
  * statement : string
    * Specifies the N1QL query statement in string form (i.e. "select foo from bar")
* Optional (part of QueryOptions):
  * scanConsistency(QueryScanConsistency) = undefined(NotBounded)
    * Specifies the level of consistency for the query.
    * Sent within the JSON payload as `scan_consistency` and is encoded as:
      * RequestPlus: `request_plus`
      * NotBounded: `not_bounded`
    * This option overrides any setting that was previously set on consistentWith
  * adhoc(boolean),
    * Specifies if the prepared statement logic should be executed internally.
    * This option is not sent over the wire but rather internally controls the prepared statement behavior as outlined in the "Enhanced Prepared Statements RFC"
  * clientContextId(String) = UUID(),
    * Specifies a context ID string which is mirrored back from the query engine on response
    * If not provided, the client must create and use a UUID instead. Sent as in the JSON payload as "client_context_id" as a JSON String `"foobar"`
  * consistentWith(MutationState) = undefined
    * Specifies custom scan consistency through "at_plus" with mutation state token vectors
    * On the wire represented as a custom scan_consistency together with a scan_vector. See the section on "ConsistentWith Handling" in this RFC for details
    * This option overrides any setting that was previously set on scanConsistency
  * maxParallelism(uint32) = undefined(0),
    * The maximum number of logical cores to use in parallel for this query
    * Sent in the JSON payload as "max_parallelism" as a JSON String i.e. `"1"`.
  * parameters(JsonObject | JsonArray) = undefined
    * Specifies positional or named parameters
    * For languages that do not support operator overloading, the alternative naming is positionalParameters(JsonArray) and namedParameters(JsonObject)
    * Sent in the JSON payload
      * For positional parameters as a json array under the "args" key
      * For named parameters directly in the JSON payload, but each named argument is prefixed with "$" if it doesn't already have the dollar prefix provided by the user
      * The payload can include both positional and named parameters
    * Setting JsonArray overrides any previously set _positional_ parameters
    * Setting JsonObject overrides any previously set _named_ parameters
  * pipelineBatch(uint32) = undefined
    * Specifies pipeline batching characteristics
    * Sent in the JSON payload as "pipeline_batch" as a JSON String
  * pipelineCap(uint32) = undefined
    * Specifies pipeline cap characteristics
    * Sent in the JSON payload as "pipeline_cap" as a JSON String
  * profile(QueryProfile) = undefined(Off)
    * Specifies the profiling level to enable
    * Sent within the JSON payload as `profile` and is encoded as:
      * Off: `"off"`
      * Phases: `"phases"`
      * Timings: `"timings"`
  * raw(String key, JsonValue value)
    * Specifies values with their key and value as presented in the map as part of the JSON payload
    * This is an escape hatch to support unknown commands and be forwards compatible
  * readonly(boolean) = undefined(false)
    * Allows to specify if the query is readonly
    * Sent within the JSON Payload as `readonly` as a JSON boolean
  * metrics(boolean) = false
    * Allows to enable the metrics at the end of the result in the metadata section
    * Sent within the JSON Payload as `metrics` as a JSON boolean
  * scanWait(Duration) = undefined
    * Specifies the maximum time wait for a scan.
    * Sent within the JSON payload as `scan_wait` encoded as a JSON String in go format! (so "1s" for example)
  * scanCap(uint32) = undefined
    * Specifies scan cap characteristics
    * Sent in the JSON payload as "scan_cap" as a JSON String
  * timeout(Duration) = $Cluster::queryOperationTimeout
    * Specifies how long to allow the operation to continue running before it is cancelled.
  * serializer(JsonSerializer) = $Cluster::Serializer
    * Specifies the serializer which should be used for deserialization of rows returned.
  * preserveExpiry(boolean) = undefined(false)
    * Specifies that the query engine should preserve expiration values set on any documents modified by this query.
    * Sent within the JSON Payload as `preserve_expiry` as a JSON boolean
    * Should be added at API stability level of uncommitted.
  * useReplica(bool) = undefined
    * Specifies that the query engine should use replica nodes for kv fetches if the active node is down.
    * Sent within the JSON payload as `use_replica` as the JSON string `"on"` or `"off"`, only if set by the user.
    * If the SDK does not have the ability to differentiate between not set and `false` then a different type should be used instead, suggested:
      * `enum QueryUseReplicaLevel { ON, OFF }` 
    * When `useReplica` is set by the user the SDK should check the `clusterCapabilities` to ensure that there is a `n1ql` field containing a `readFromReplica` entry.
      * If this is not present then fail with a FeatureNotAvailableException.
    * If the query service does read from a replica then it will populate an entry in the `warning` field in the response payload.


**Returns:**

A `QueryResult` that maps the result of the N1QL query to an object.

**Throws:**

* Documented
  * RequestTimeoutException (#1)
  * CouchbaseException (#0)
* Undocumented
  * CasMismatchException (#9)
  * RequestCanceledException (#2)
  * InvalidArgumentException (#3)
  * ServiceNotAvailableException (#4)
  * InternalServerException (#5)
  * AuthenticationException (#6)
  * PlanningFailedException (#201)
  * QueryIndexException (#202)
  * PreparedStatementException (#203)
  * ParsingFailedException (#8)
  * FeatureNotAvailableException (#15)

## IScope#Query

The API and implementation are identical to `ICluster::Query`, except that all queries will be sent with a `query_context` parameter of ``"default:`bucket-name`.`scope-name`"``.

## QueryProfile

```
enum QueryProfile {
  Off,
  Phases,
  Timings
}
```

In the JSON payload, the field name is `profile`.

| Identifier | Wire Format  | Description     |
|------------|--------------|-----------------|
| Off        | `"off"`      | No profiling information is returned from the cluster. This is the default. Not sent on the wire if set.   |
| Phases     | `"phases"`   | Profiling information about the various phases is included.   |
| Timings    | `"timings"`  | In addition to the phases, detailed timing information is available.  |


## QueryScanConsistency

```
enum QueryScanConsistency {
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

## QueryResult

The QueryResult is returned if the response arrived successfully and contains both the associated metadata and the actual rows.

```
struct QueryResult {
    Stream<T> Rows();
    Promise<QueryMetaData> MetaData();
}
```

Depending on the language best practices, rows() returns a list of entities, but how they are decoded is language-dependent.

```
struct QueryMetaData {
    String RequestId();
    String ClientContextId();
    QueryStatus Status();
    Optional<JsonValue> Signature();
    List<QueryWarning> Warnings();
    QueryMetrics Metrics();
    Optional<JsonObject> Profile();
}
```

NOTE: If present, the signature value returned by the server is usually a JSON Object or a JSON String, but the server may return any JSON type (Object, Array, String, Number, Boolean).

Everything but the signature and the profile should always be present. Even when the Metrics are disabled in the options, a default implementation should be provided. The reasoning behind this is that it will be enabled 99% of the time and more and having every user to check for non-existence is more error prone.

```
struct QueryMetrics {
    Duration elapsedTime()
    Duration executionTime()
    uint64 sortCount()
    uint64 resultCount()
    uint64 resultSize()
    uint64 mutationCount()
    uint64 errorCount()
    uint64 warningCount()
}
```

The status needs to be decoded from the wire representation, which is in all cases the lowercase version of the enum. So "success" on the wire is turning into SUCCESS.

```
enum QueryStatus {
    Running,
    Success,
    Errors,
    Completed,
    Stopped,
    Timeout,
    Closed,
    Fatal,
    Aborted,
    Unknown
}
```

A QueryWarning is always a tuple of code and message:

```
class QueryWarning {
    int32 code()
    String message()
}
```

# Changelog

* Sept 12, 2019 - Revision #1
  * Initial Draft
* Sept 27, 2019 - Revision #2 (by Michael Nitschinger)
  * Major refactoring and alignment with the analytics & view rfc
* Sept 27, 2019 - Revision #3 (by Michael Nitschinger)
  * Make QueryMetrics optional in QueryMetaData, because now they are disabled by default
* Sept 30, 2019 - Revision #4 (by Michael Nitschinger)
  * Clarify that max_parallelism, pipeline_cap, pipeline_batch and scan_cap need to be sent over the wire as Strings not Numbers!
* Oct 16, 2019 - Revision #5 (by Michael Nitschinger)
  * QueryStatus enum values are converted from uppercase to camel case
* April 30, 2020
  * Moved RFC to ACCEPTED state.
* January 5, 2022
  * Added preserveExpiry to QueryOptions.
* June 21, 2023 - Revision #6 (by Charles Dixon)
  * Added useReplica to QueryOptions.
* Dec 11, 2023 - Revision #7 (by David Nault)
  * Change type of QueryMetadata.signature() from JsonObject to JsonValue.
* Feb 6, 2025 - Revision #8 (by Dimitris Christodoulou)
  * Add IScope#Query API.
* April 30, 2025 - Revision #9 (by Dimitris Christodoulou)
  * Allow setting both positional and named parameters.

# Signoff

| Language   | Team Member    | Signoff Date  | Revision |
|------------|----------------|---------------|----------|
| Node.js    | Jared Casey    | May 1, 2025   | #9       |
| Go         | Charles Dixon  | July 6, 2023  | #6       |
| Connectors | David Nault    | May 1, 2025   | #9       |
| PHP        | Sergey Avseyev | May 7, 2025   | #9       |
| Python     | Jared Casey    | May 1, 2025   | #9       |
| Scala      | Graham Pople   | May 2, 2025   | #9       |
| .NET       | Jeffry Morris  | May 8, 2025   | #9       |
| Java       | David Nault    | May 1, 2025   | #9       |
| Kotlin     | David Nault    | May 1, 2025   | #9       |
| C          | Sergey Avseyev | May 7, 2025   | #9       |
| C++        | Sergey Avseyev | May 7, 2025   | #9       |
| Ruby       | Sergey Avseyev | May 7, 2025   | #9       |
