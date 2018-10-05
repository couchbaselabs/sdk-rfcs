# SDK-RFC 27: Analytics Querying

## Meta

 - RFC Name: Analytics Querying
 - RFC ID: 0027
 - Start Date: 2016-12-30
 - Owner: Michael Nitschinger
 - Current Status: Review

## Summary

This SDK RFC describes the user-facing API for the analytics service, as well as the internal API to use when communicating with the service.

## Motivation

With Couchbase Server 6.0 (codename "Alice") the analytics service is being promoted to general availability, and as a result the SDKs need to follow suit and provide a stable, supported API surface. As the start date suggests, the analytics API has been a long time coming and individual SDKs already provide experimental support to a varying degree. This RFC is an effort to steer all SDKs onto a common API, similar to what we already provide for N1QL, Views and FTS on querying.

## General Design

While not being a hard requirement, the analytics querying APIs by intention track the N1QL one very closely. Certain functionality is only available with analytics and vice versa, but overall the feeling of the API as well as naming conventions are very similar. The main theme is that users already using N1QL should feel right at home and not be exposed to too much different semantics.


### Basic Queries

A simple analytics query looks very similar to a N1QL query. As far as the SDK is concerned it is exposed as a raw `String`.


```sql
select 1=1
```

Higher level representations of this opaque string are out of scope for this RFC, but might be implemented at the discretion of the SDK (i.e. a syntax-aware builder API).

The query is sent via JSON to the `/analytics/service` HTTP (POST) endpoint.

The simplest body structure looks like this:


```json
{
    "statement": "select 1=1"
}
```


When sent to the server (here with curl) the response looks like this:


```
$ curl -X POST -H "Content-Type: application/json" -u user:pass --data "{\"statement\": \"select 1=1\"}" host:port/analytics/service
```

```json
{
	"requestID": "a3204b8c-c8db-40a5-a89e-1c1888de9752",
	"signature": "*",
	"results": [ { "$1": true }
 ]
	,
	"status": "success",
	"metrics": {
		"elapsedTime": "19.086771ms",
		"executionTime": "13.41269ms",
		"resultCount": 1,
		"resultSize": 15,
		"processedObjects": 0
	}
}
```

The request structure and the response format - discussed in a later section - is intentionally very similar to N1QL, but it also contains additional fields (i.e. the `processedObjects` field in the `metrics` section).


### Parameterized Queries

In addition to simple queries, analytics supports parameterized queries. Supported are:

 - Positional Parameters (referred to as `?` or `$n` where n >= 1)
 - Named Parameters (referred to as `$name`)

The identifiers are placed inside the statement and the actual values for the parameters are delivered as part of the JSON body structure. 

```
$ curl -X POST -H "Content-Type: application/json" -u user:pass --data '{"statement":"select count(*) from airports where name = $name", "$name":"foo"}' host:port/analytics/service
```

```
$ curl -X POST -H "Content-Type: application/json" -u user:pass --data '{"statement":"select count(*) from airports where name = ?", "args":["foo"]}' host:port/analytics/service
```

### Long-Running Queries

One difference at runtime is the expectation that N1QL queries complete fairly quickly, but analytics queries can take more time to complete.

This RFC does not propose a change in default timeout (75s), but acknowledges the fact that the timeout will be tuned more often by users. See the timeout propagation section for more info on this.

The analytics service provides different modes of execution:

 - `immediate`: the job will be scheduled and finished, returning a JSON envelope which contains the results. This is the default mode of operation.

 - `deferred`: the job will be scheduled and finished, but the response structure will contain a `handle` instead of results. This handle URI can then be used to retrieve the results.

 - `async`: the job will be scheduled and a `handle` is sent as part of the response immediately. This handle URI can then be used to check both the status of the computation and also to retrieve the results once finished.

Only `immediate` is required as part of this RFC, the `async` query execution is defined in RFC 45. As of this RFC, the `deferred` mode will not be exposed as well.

### Priorities

It is possible to specify a higher priority to a request. Requests with high priority have their own queue and executors.

This must be done through setting the `Analytics-Priority` HTTP header to `-1` or any negative integer. Note that since as of Couchbase Server 6.0 there is only one high priority work queue, the setting (named `priority`) will be exposed as a boolean rather than a number to avoid confusion (i.e. setting different priorities > 0 and expecting them to be scheduled accordingly).

The headers are set as follows, but note that there is no feedback in the response if the header has been applied correctly.


```
curl -X POST -H "Content-Type: application/json" -H "Analytics-Priority: -1" -vv -u Administrator:password --data "{\"statement\": \"select 1=1\"}" 127.0.0.1:8095/analytics/service

* Connected to 127.0.0.1 (127.0.0.1) port 8095 (#0)
* Server auth using Basic with user 'Administrator'
> POST /analytics/service HTTP/1.1
> Host: 127.0.0.1:8095
> Authorization: Basic QWRtaW5pc3RyYXRvcjpwYXNzd29yZA==
> User-Agent: curl/7.54.0
> Accept: */*
> Content-Type: application/json
> Analytics-Priority: -1
> Content-Length: 27
```

### Timeout Propagation

The SDK must ensure that, once it stops waiting for a result because of a timeout at the user level, the server does not waste resources in finish computing the result.

This is done by always setting the `timeout` value as part of the JSON request to the same value that is provided by the user. If no explicit timeout is configured, the default timeout must be set.

NOTE: timeout values are expressed on the wire with a time unit suffix, eg 75000ms (using the similar go-style formatting as N1QL does).

### Authentication

Analytics supports two modes of authentication:

 - HTTP Basic Auth
 - Client Certificate Authentication

Similar to the other HTTP-based services, the same rules apply: If client certificate authentication is enabled the HTTP header must be omitted, otherwise it is mandatory. If authentication fails, the server will return a `401 Unauthorized`.


### IPv6

Analytics supports IPv6 and the SDK must support it the same way it does for other services as well. Mentioned for completeness sake, since as of this writing there are no special cases known.


### Error Handling

An analytics response might contain errors instead of a successful response, so the format changes to an `errors` section instead of `results`:


```json
{
	"requestID": "34c37086-cef4-4ac3-915c-7a31fa428ed6",
	"signature": "*",
	"errors": [{ 
		"code": 1,
		"msg": "ASX1001: Syntax error: In line 1 >>select 1=;<< Encountered \"=\" at column 9. "
	}],
	"status": "fatal",
	"metrics": {
		"elapsedTime": "22.177942ms",
		"executionTime": "20.495729ms",
		"resultCount": 0,
		"resultSize": 0,
		"processedObjects": 0,
		"errorCount": 1
	}
}
```

The following error codes have been identified by the analytics team as retryable:

 - 21002: Request timed out and will be cancelled
 - 23000: Analytics Service is temporarily unavailable
 - 23003: Operation cannot be performed during rebalance
 - 23007: Job queue is full with [string] jobs

If there has been a retry policy in place already for N1QL queries, the same policy should be used for analytics queries (i.e. number of retries, delay,...). If there is no retry policy in place, the following should be used:

 - Maximum number of 10 retries
 - Exponential Delay, starting at 2ms, up to 500ms max

A couple of remarks on semantics:

 - If the server returns a status that is not `success`, the SDK must inspect the errors for the return codes above and handle the retry logic accordingly.
 - A full list of error codes can be found [https://github.com/couchbaselabs/asterix-opt/blob/master/cbas/doc/errors/error-codes.md](here).
 - Note that in addition of the `errors` section, analytics also defines a `warnings` section with the same structure that might or might not be present as well. As of this RFC there are no warnings defined but there will be in future server releases.
 - For the purpose of this RFC, there will be no error code bracketing defined. All error codes must be matched and handled exactly as defined in this RFC.


### Keepalive/Ping Endpoint

Analytics provides a HTTP endpoint that must be used for application-level keepalive/ping/diagnostics.

This API can be found under `/admin/ping` on the analytics port, which responds from every individual analytics service. This replaces deprecated APIs such as `/analytics/version` that have been used in the past.

Note that GET mut be used on `/admin/ping`, and while the endpoint returns a payload like `{"status": "ok"}` there is no requirement to inspect the payload. SDKs may log the status at their liberty to aid debugging.


### Out of Scope

This section covers a list of features which are not in scope for various reasons:

 - Prepared Statements: not supported by the server
 - Scan Consistency: not supported by the server (+ mutation tokens)
 - long Running Queries (async mode): see long-running queries section for rationale (and other RFC for experimental implementation)
 - HTTP/2 Support

## Reference

The following sections discuss the API and server communication in a reference style.

### Server Config Extensions

The cluster manager exposes the analytics service similar to all the others. There are plenty of different analytics-related ports exposed in the config, but only those two are important:

 - `"cbas": 8095`
 - `"cbasSSL": 18095`

The SDK must only send analytics queries to nodes where this service identifier is exposed. The default ports are `8095` and for TLS `18095`.

The SDK must use the *round robin* strategy to distribute analytics queries across the MDS-enabled analytics nodes.


### Request Format

Every query must be sent to the `/analytics/service` HTTP endpoint and use the `POST` verb. 

The following headers must be present at query time:

 - `Content-Type`: `application/json`
 - `User-Agent`: SDK identifier, similar to other services like N1QL.

If the `priority` flag is present, the following header needs to be set:

 - `Analytics-Priority`: `-1`

The payload of the request is a JSON body with the following fields:

<table>
  <tr>
   <td><strong>Field Name</strong>
   </td>
   <td><strong>Required</strong>
   </td>
   <td><strong>Type</strong>
   </td>
   <td><strong>Description</strong>
   </td>
  </tr>
  <tr>
   <td><code>statement</code>
   </td>
   <td>yes
   </td>
   <td>string
   </td>
   <td>Contains the actual query
   </td>
  </tr>
  <tr>
   <td><code>timeout</code>
   </td>
   <td>yes
   </td>
   <td>string
   </td>
   <td>not required by the server, but the SDK must always set it to the client side timeout. Uses the "go notation" exactly like N1QL
   </td>
  </tr>
  <tr>
   <td><code>client_context_id</code>
   </td>
   <td>yes
   </td>
   <td>string
   </td>
   <td>not required by the server, but SDK must always set it so it can be properly used for tracing (a UUID)
   </td>
  </tr>
  <tr>
   <td><code>pretty</code>
   </td>
   <td>no
   </td>
   <td>boolean
   </td>
   <td>if set to true, output will be nicely formatted
   </td>
  </tr>
  <tr>
   <td><code>args</code>
   </td>
   <td>no
   </td>
   <td>array of json values
   </td>
   <td>present if positional parameters are used
   </td>
  </tr>
  <tr>
   <td><code>$<identifier></code>
   </td>
   <td>no
   </td>
   <td>any json type
   </td>
   <td>present if named parameters are used
   </td>
  </tr>
</table>


If the field is marked as optional (non-required), it should not be present in the JSON body if not explicitly provided/set by the user.

Here is an example request payload (pretty formatted, but the SDK should send it as non-pretty as possible):


```json
{
  "statement": "select 1=1",
  "timeout":"75000ms",
  "client_context_id": "bfebf0ad-e022-43b5-95b3-ff345ef6adb6",
  "pretty": true
}  
```

### Response Format

The server responds with a JSON body which has the following fields (which might not be present at all time):


```json
{
	"requestID": "a1c4115c-792b-491e-8604-00ab9c12aea6",
    "clientContextID": "...",
	"signature": "*",
	"results": [ ],
      "errors": [ ],
      "warnings": [ ],
	"status": "...",
     "plans":{},
     "handle": "...",
	"metrics": {
		"elapsedTime": "22.998514ms",
		"executionTime": "16.47064ms",
		"resultCount": 1,
		"resultSize": 18,
		"processedObjects": 0,
           "mutationCount": 0,
           "sortCount": 0,
           "errorCount": 0,
           "warningCount": 0
	}
}
```

Also fields in the metrics might or might not be present, so make sure the code always has a sensible default value ready (optional, 0, null,...).

Especially note that while `plans` is present, it is to be ignored and not exposed to the user as part of this RFC (considered internal use at the time of writing).

### Request API

The following API aims to closely mirror what the SDK already exposes for N1QL, so if in doubt follow what the SDK currently provides.

If the language supports overloads, an analytics overload for the query should be provided. i.e here in java:

```java
bucket.query(AnalyticsQuery query)
// bucket.query(N1qlQuery query)
```

If not a new method should be introduced. i.e here in golang:

```java
bucket.ExecuteAnalyticsQuery(query)
// bucket.ExecuteN1qlQuery(query)
```

The query structure itself is composed of the following (optional in brackets):

```
struct AnalyticsQuery {
    statement: String,
    params: AnalyticsParams,
    [named_params: Map[String, String]],
    [positional_params: List[String]],
}
```

 - For parameterized queries, named and positional params must be mutually exclusive.

 - Because named params must have the `$`-sign set on the wire, the SDK *must* check the param and append a `$`-sign if not already present. This increases developer productivity.

The `AnalyticsParams` should reflect the properties available to configure. Follow the builder/construction pattern that is already used for N1QL queries as well.

```
struct AnalyticsParams {
    serverSideTimeout: Duration,
    withContextId: String,
    rawParam: (String, Object),
    pretty: boolean,
    priority: boolean
}
```

These params are all optional and can be used to override the defaults. Note that the naming of these fields should match the N1QL ones (even if for example `withContextId` translates to `client_context_id` on the wire).

Note that `rawParam` is used as an escape hatch to future-proof the API. This allows the user to pass in properties which are currently intentionally not exposed but accepted by the server (or will be in the future).


### Response API

The response API should follow the N1QL API closely, exposing a `AnalyticsQueryResult` which contains the results as well as any metadata associated:

```
struct AnalyticsQueryResult {
    rows: Iterable[AnalyticsQueryRow],
    errors: Iterable[JsonObject],
    warnings: Iterable[JsonObject],
    signature: Object,
    requestId: String,
    clientContextId: String,
    status: String,
    info: AnalyticsMetrics,
}
```

Since errors and warnings might contain free-form data it makes sense to expose them as a generic "json object".

The `AnalyticsQueryRow` contains the actual JSON data, so in languages where JSON is a first-class construct it can be exposed directly.


```
struct AnalyticsQueryRow {
    value: JsonObject
}
```

Finally, `AnalyticsMetrics` should provide typed access to the fields:


```
struct AnalyticsMetrics {
    elapsedTime: string,
    executionTime: string,
    resultCount: uint,
    resultSize: uint,
    processedObjects: uint,
    mutationCount: uint,
    sortCount: uint,
    errorCount": uint,
    warningCount: uint
}
```

The following fields are optional: `mutationCount`, `sortCount`, `errorCount`, `warningCount`.

## Language Specifics

Each language should supply the following structures into this RFC:

 - bucket-level query API
 - `AnalyticsQuery` struct/object
 - `AnalyticsParams` struct/object
 - `AnalyticsQueryResult` + rows
 - `AnalyticsMetrics`


### C

```c
static void row_callback(lcb_t instance, int type, const lcb_RESPN1QL *resp)
{
    if (resp->rc != LCB_SUCCESS) {
        fprintf(stderr, "ERROR: %s, HTTP STATUS: %d, BODY: %.*s\n",
	    lcb_strerror_short(resp->rc)
	    resp->htresp ? resp->htresp->htstatus : -1,
	    (int)resp->nrow, (const char *)resp->row);
    }
    printf("%s: %.*s\n",
	(resp->rflags & LCB_RESP_F_FINAL) ? "META" : "ROW",
       	(int)resp->nrow, (char *)resp->row);
}

const char *query = "{\"statement\": \"SELECT VALUE bw FROM breweries bw WHERE bw.name = $name\", \"$name\": \"Kona Brewing\"}";
lcb_CMDN1QL cmd = {0};
cmd.cmdflags = LCB_CMDN1QL_F_ANALYTICSQUERY;
cmd.callback = row_callback;
cmd.query = query;
cmd.nquery = strlen(query);
lcb_n1ql_query(instance, NULL, &cmd);
lcb_wait(instance);
```

### Java

Bucket-Level API

Async:

```java
Observable<AsyncAnalyticsQueryResult> query(AnalyticsQuery query);
Observable<AsyncAnalyticsQueryResult> query(AnalyticsQuery query, long timeout, TimeUnit timeUnit);
```

Sync:


```java
AnalyticsQueryResult query(AnalyticsQuery query);
AnalyticsQueryResult query(AnalyticsQuery query, long timeout, TimeUnit timeUnit);
```


`AnalyticsQuery` struct/object


```java
class AnalyticsQuery {


    SimpleAnalyticsQuery simple(final String statement);
    SimpleAnalyticsQuery simple(final String statement, final AnalyticsParams params);

    ParameterizedAnalyticsQuery parameterized(final String statement,
        final JsonArray positionalParams);
    ParameterizedAnalyticsQuery parameterized(final String statement,
        final JsonArray positionalParams, final AnalyticsParams params);
    static ParameterizedAnalyticsQuery parameterized(final String statement,
        final JsonObject namedParams);
    ParameterizedAnalyticsQuery parameterized(final String statement,
        final JsonObject namedParams, final AnalyticsParams params);

}
```

`AnalyticsParams` struct/object


```java
class AnalyticsParams {
    AnalyticsParams withContextId(String clientContextId);
    AnalyticsParams serverSideTimeout(long timeout, TimeUnit unit);
    AnalyticsParams rawParam(String name, Object value);
    AnalyticsParams pretty(boolean pretty);
    AnalyticsParams priority(boolean priority);    
}
```


`AnalyticsQueryResult` + rows


```java
interface AnalyticsQueryResult {
    List<AnalyticsQueryRow> allRows();
    Iterator<AnalyticsQueryRow> rows();
    Object signature();
    AnalyticsMetrics info();
    boolean parseSuccess();
    boolean finalSuccess();
    String status();

    // Note: right now warnings are folded into the errors
    // object because of limitations in the streaming parser
    // which we can lift in the future.
    List<JsonObject> errors();

    String requestId();
    String clientContextId();
}

interface AnalyticsQueryRow {
    byte[] byteValue();
    JsonObject value();
}
```


`AnalyticsMetrics`


```java
class AnalyticsMetrics {
    String elapsedTime();
    String executionTime();
    int sortCount();
    int resultCount();
    long resultSize();
    int mutationCount();
    int errorCount();
    int warningCount();
    long processedObjects();
}
```



### .NET

**Bucket Level API**


```csharp
 IAnalyticsResult<T> Query<T>(IAnalyticsRequest analyticsRequest);

 Task<IQueryResult<T>> QueryAsync<T>(IQueryRequest queryRequest, CancellationToken cancellationToken);

 Task<IAnalyticsResult<T>> QueryAsync<T>(IAnalyticsRequest analyticsRequest);

Task<IAnalyticsResult<T>> QueryAsync<T>(IAnalyticsRequest analyticsRequest, CancellationToken cancellationToken);
```

`AnalyticsQuery` struct/object includes `AnalyticsParams` struct/object

```csharp
 public interface IAnalyticsRequest
 {
      string OriginalStatement { get; }
      string CurrentContextId { get; }
      IDictionary<string, object> GetFormValues();
      string GetFormValuesAsJson();
      bool TimedOut();
      IAnalyticsRequest Statement(string statement);
      [Obsolete]
      IAnalyticsRequest Credentials(string username, string password, bool isAdmin);
      IAnalyticsRequest AddCredentials(string username, string password, bool isAdmin);
      IAnalyticsRequest ClientContextId(string contextId);
      IAnalyticsRequest Pretty(bool pretty);
      IAnalyticsRequest IncludeMetrics(bool includeMetrics);
      IAnalyticsRequest AddNamedParamter(string key, object value);
      IAnalyticsRequest AddPositionalParameter(object value);
      IAnalyticsRequest ExecutionMode(ExecutionMode mode);
      IAnalyticsRequest Timeout(TimeSpan timeout);
      IAnalyticsRequest Priority(bool priority);
      IAnalyticsRequest Priority(int priority);
  }
```

`AnalyticsQueryResult` + rows

```csharp
public interface IAnalyticsResult<T> : IResult, IEnumerable<T>
  { 
      HttpStatusCode HttpStatusCode { get; set; }
      List<T> Rows { get; }
      Guid RequestId { get; }
      string ClientContextId { get; }
      QueryStatus Status { get; }
      dynamic Signature { get; }
      List<Error> Errors { get; }
      Metrics Metrics { get; }
  }
```

`AnalyticsMetrics`

```csharp
public class Metrics
{
    public string ElaspedTime { get; set; }
    public string ExecutionTime { get; set; }
    public uint ResultCount { get; set; }
    public uint ResultSize { get; set; }
    public uint MutationCount { get; set; }
    public uint ErrorCount { get; set; }
    public uint WarningCount { get; set; }
    public uint SortCount { get; set; }
}
```

### Python

Bucket level query API:


```python

class couchbase.bucket.Bucket:
    analytics_query(query, host, *args, **kwargs)

        Parameters:
            query – The query to execute. This may either be a AnalyticsQuery object, or a string (which will be implicitly converted to one).
            host – The host to send the request to.
            args – Positional arguments for `AnalyticsQuery`.
            kwargs – Named arguments for `AnalyticsQuery`.

        Returns:
        An iterator which yields rows. Each row is a dictionary representing a single result


```


`AnalyticsQuery` struct:


```python
class couchbase.analytics.AnalyticsQuery:
    __init__(querystr, *args, **kwargs)
        Create an Analytics Query object. This may be passed as the
        params argument to `AnalyticsRequest`.

        Parameters:
            querystr – The query string to execute
            args – Positional placeholder arguments. These satisfy the 
                placeholder values for positional placeholders in the query
                string, demarcated by ?.
            kwargs – Named placeholder arguments. These satisfy named
                placeholders in the query string, such as $name,
                $email and so on. For the placeholder values, omit the
                leading dollar sign ($).


`AnalyticsQueryResult` (`AnalyticsRequest` for consistency with N1QL API):

class couchbase.analytics.AnalyticsRequest:
    __init__(params, host, parent)
        Object representing the execution of the request on the server.
        Parameters:
            params – An `AnalyticsQuery` object.
            host – the host to send the request to.
            parent – The parent Bucket object
```

`AnalyticsQueryRow` is just the result (typically a `dict` or a `list`)returned by the iterator interface of `AnalyticsRequest`:


```python
class couchbase.analytics.AnalyticsRequest:
    __iter__
```

`AnalyticsMetrics` is just a Python `dict` returned by this property:

```python
class couchbase.analytics.AnalyticsRequest:    metrics
```

### Golang

Bucket level query API:


```golang
func (b *Bucket) ExecuteAnalyticsQuery(q *AnalyticsQuery, params interface{}) (AnalyticsResults, error)
```


`AnalyticsQuery` struct/object + `AnalyticsParams` struct/object (functions in this case):


```golang
type AnalyticsQuery struct {
	options map[string]interface{}
}

func NewAnalyticsQuery(statement string) *AnalyticsQuery
func (aq *AnalyticsQuery) ServerSideTimeout(timeout time.Duration) *AnalyticsQuery
func (aq *AnalyticsQuery) Pretty(pretty bool) *AnalyticsQuery 
func (aq *AnalyticsQuery) ContextId(clientContextId string) *AnalyticsQuery
func (aq *AnalyticsQuery) RawParam(name string, value interface{}) *AnalyticsQuery 
func (aq *AnalyticsQuery) Priority(priority bool) *AnalyticsQuery
```


`AnalyticsQueryResult` + rows (named `AnalyticsResults `for consistency with N1QL):


```golang
type AnalyticsResults interface {
	One(valuePtr interface{}) error
	Next(valuePtr interface{}) bool
	NextBytes() []byte
	Close() error

	RequestId() string
	ClientContextId() string
	Status() string
	Warnings() []AnalyticsWarning
	Signature() interface{}
	Metrics() AnalyticsResultMetrics
}
```


`AnalyticsMetrics` (named `AnalyticsResultMetrics `for consistency with N1QL):


```golang
type AnalyticsResultMetrics struct {
	ElapsedTime      time.Duration
	ExecutionTime    time.Duration
	ResultCount      uint
	ResultSize       uint
	MutationCount    uint
	SortCount        uint
	ErrorCount       uint
	WarningCount     uint
	ProcessedObjects uint
}
```



### NodeJS


```javascript
Cluster.prototype.query = function(query, params, callback)...

AnalyticsQuery.fromString = function(str)...
AnalyticsStringQuery.prototype.pretty = function(pretty)...
AnalyticsStringQuery.prototype.priority = function(priority)...
AnalyticsStringQuery.prototype.rawParam = function(name, value)...
```


### PHP


```php
class AnalyticsQuery {
    public static function fromString(string $statement) {}
    public function positionalParams(array $params) {}
    public function namedParams(array $params) {}
    public function rawParam(string $key, mixed $value) {}
}
$query = \Couchbase\AnalyticsQuery::fromString('
    SELECT "Hello, " || $name || "!" AS greeting,
           "¡Hola, " || ? || "!" AS saludo
');
$query->namedParams(['name' => 'Beer']);
$query->positionalParams(['Cerveza']);
$res = $bucket->query($query);
var_dump($res->rows[0]);
//=> object(stdClass)#5 (2) {
//     ["greeting"]=>
//     string(12) "Hello, Beer!"
//     ["saludo"]=>
//     string(16) "¡Hola, Cerveza!"
//   }
```

## Questions

Q: Will errors be the same as N1QL REST?

A: No, as of July 2018, Till reports they've decided to partially follow, but diverge in some ways.

Q: Will there be any kind of error bracketing?

A: Interesting idea, needs more discussion said Matt and Till.

Q: How can one have the output of an analytics query go to a bucket/collection?

A: For now, possibly only in certain SDKs, we should consider adding API to implement this.  Suggestion is the streaming parser, some sort of sink output, and some sort of transformer/generator interface for the user to implement to add the necessary metadata.


## Open Items


## Changelog



 -   2018-09-24
   -   Moved async out to its own RFC (45)
   -   Added example for priority
   -   Added section for retry on retryable error codes
   -   Added section on http keepalive/ping endpoint
 -   2018-08-20 Added initial approach for experimental async mode
 -   2018-08-13 Big Overhaul of the RFC
 -   2018-08-29 Cleared up some open points, no changes just additions


# Signoff

<table>
  <tr>
   <td><strong>Language</strong>
   </td>
   <td><strong>Representative</strong>
   </td>
   <td><strong>Date</strong>
   </td>
  </tr>
  <tr>
   <td>C
   </td>
   <td>Sergey Avseyev
   </td>
   <td>2018-10-01
   </td>
  </tr>
  <tr>
   <td>Go
   </td>
   <td>Charles Dixon
   </td>
   <td>2018-10-03
   </td>
  </tr>
  <tr>
   <td>Java
   </td>
   <td>Michael N.
   </td>
   <td>2018-10-01
   </td>
  </tr>
  <tr>
   <td>.NET
   </td>
   <td>Jeff Morris
   </td>
   <td>2018-10-04
   </td>
  </tr>
  <tr>
   <td>Node.js
   </td>
   <td>Brett Lawson
   </td>
   <td>2018-10-04
   </td>
  </tr>
  <tr>
   <td>PHP
   </td>
   <td>Sergey Avseyev
   </td>
   <td>2018-10-01
   </td>
  </tr>
  <tr>
   <td>Python
   </td>
   <td>Ellis Breen
   </td>
   <td>2018-10-01
   </td>
  </tr>
</table>
