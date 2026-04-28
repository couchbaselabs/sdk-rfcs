# Meta

- RFC Name: Eventing Management APIs
- RFC ID: 62
- Start Date: January 28, 2020
- Owner: David Nault
- Current Status: DRAFT

# Motivation

Developers want to use the SDK to manage eventing functions.

# Limitations

- The SDK API described in this document covers only the most common use cases. Some advanced features of the Eventing Service's HTTP API are not exposed by the SDK API.
- This SDK API is not supported when using Couchbase Server versions prior to 7.0.
- Capella is out of scope.
- At the time of writing, there is no practical way for the SDK to determine whether the server supports the eventing function scope feature added in Couchbase Server 7.1. When used with a pre-7.1 server, the SDK always upserts functions in the \*.\* function scope, even if a scoped manager is used.

# References

## HTTP API

The SDK uses the Eventing Service HTTP API documented here:  
https://docs.couchbase.com/server/current/eventing/eventing-api.html

The server advertises the Eventing Service HTTP ports as "eventingAdminPort" for non-TLS (default 8096), and "eventingSSL" for TLS (default 18096). The SDK MUST get the actual ports from the \`nodesExt\` field of the cluster config, taking into account alternate addresses.

## JSON Schema

The server repository contains a formal specification of the JSON used in the HTTP API: https://github.com/couchbase/eventing/tree/master/parser/handler_schema.json

# Concepts

## Eventing Function

An eventing function is a script the Couchbase Server Eventing Service runs whenever a document changes on the server. The definition of a function also includes several properties that control the function's execution environment.

## Keyspace

A keyspace is the fully-qualified name of a Key/Value (KV) collection. It consists of a bucket name, a scope name, and a collection name.

The definition of an eventing function includes two keyspaces:

* Source Keyspace \- the collection being watched for changes.  
* Metadata Keyspace \- the collection where function execution metadata is stored.

## Function Scope

Before Couchbase Server 7.1, all eventing functions were in the same namespace, and function names were unique within a cluster.

Couchbase Server 7.1 added the concept of eventing function scopes. These scopes are used for function management Role-Based Access Control (RBAC) and for organizing functions into logical groups. Function names are unique within a scope.

An eventing function scope consists of a bucket name and a scope name. A special scope with bucket name "\*" and scope name "\*" is used for functions created prior to Couchbase Server 7.1, or when no scope is specified. This \*.\* scope is sometimes referred to as the "admin" scope, since prior to Couchbase 7.1 only administrators could manage eventing functions.

With the exception of the \*.\* admin scope, every eventing function scope corresponds to a KV scope. When a KV scope is dropped, the server automatically undeploys and drops all eventing functions in the scope.

> [!NOTE]
> A function's scope \*does not\* limit the documents the function can access; document access is controlled by the permissions of the user who created the function. A function's source and metadata keyspaces may be in KV scopes that differ from the function's scope, although this is strongly discouraged.

## Eventing Function Manager

An eventing function manager is the SDK component that lets a user do CRUD operations on eventing functions. It also lets a user deploy, undeploy, pause, and resume functions, and get the status of functions.

There is a separate eventing function manager for each KV scope, as well as a single cluster-level manager for the \*.\* (admin) scope. Each manager is bound to a single scope, and operates only on functions within that scope.

The cluster-level manager for functions in the \*.\* scope is accessed by calling `cluster.eventingFunctions()`. Similarly, a scope-level manager is accessed by calling `scope.eventingFunctions()`.

The cluster- and scope-level managers currently have the same methods with the same signatures. However, they are not the same class. The cluster-level class is called `EventingFunctionManager`, while the scope-level class is called `ScopeEventingFunctionManager`. These classes should not have an inheritance relationship, because in the future we may add methods to one class but not the other. This follows the precedent set by other scoped managers, and lets us evolve the cluster- and scope-level APIs independently.

## Client-Side Scope Filtering

Some manager methods return lists of functions or function statuses. These methods MUST NOT return items that are not in the manager’s scope.

The Eventing Service HTTP API does not support getting all functions (or function statuses) in a specific scope. Consequently, the SDK MUST filter these responses and exclude any items whose scope does not match the manager’s scope.

For `EventingFunctionManager`, this means including only items that have a “function\_scope” field with bucket “\*” and scope “\*”, and also functions that have no “function\_scope” field. (The absence of that field means the server is pre-7.1, and all functions are implicitly in the \*.\* scope.)

For `ScopeEventingFunctionManager`, this means including only items that have a “function\_scope” field with bucket and scope matching the manager’s scope.

# Implementation

## API Overview

The `EventingFunctionManager` can be accessed off the `Cluster` instance, similar to other management APIs in SDK 3:

```
class Cluster {  
    ...  
    EventingFunctionManager EventingFunctions();  
    ...  
}
```

A `ScopeEventingFunctionManager` can be accessed off a `Scope` instance:

```
class Scope {  
    ...  
    ScopeEventingFunctionManager EventingFunctions();  
    ...  
}
```

These managers allow the user to read functions, modify them, and change their deployment state. The API is modeled with common verbs across the other managers while still trying to preserve the semantics of the exposed eventing REST API.

```
class [Scope]EventingFunctionManager {

    void UpsertFunction(EventingFunction function, [UpsertFunctionOptions options])

    void DropFunction(String name, [DropFunctionOptions options])

    void DeployFunction(String name, [DeployFunctionOptions options])

    Iterable<EventingFunction> GetAllFunctions([GetAllFunctionsOptions options]);

    EventingFunction GetFunction(String name, [GetFunctionOptions options]);

    void PauseFunction(String name, [PauseFunctionOptions options])

    void ResumeFunction(String name, [ResumeFunctionOptions options])

    void UndeployFunction(String name, [UndeployFunctionOptions options])

    EventingFunctionsStatus functionsStatus(FunctionsStatusOptions options)

}
```

The `EventingFunction` encapsulates state and properties of a server side eventing function (see section on the `EventingFunction` for details).

## Options

The options blocks for all methods include the standard optional parameters common to most SDK 3 requests. Namely: `timeout` (or `timeoutMillis`), `retryStrategy`, and `parentSpan`. 

### Timeout

The timeout is a client-side timeout. It is the duration the SDK should allow for the server to send a complete response. If the SDK does not receive a complete response before this duration elapses, the SDK throws a timeout exception.

The SDK throws an `UnambiguousTimeoutException` if the operation uses the HTTP method GET, otherwise `AmbiguousTimeoutException`.

If not specified, the timeout defaults to the SDK’s `managementTimeout` client setting.

## EventingFunctionManager

The following sections explain the methods of the manager in detail.

### `DropFunction`

Deletes a function.

**Signature**

```
void DropFunction(String name, [DropFunctionOptions options])
```

**Parameters**

* Required:  
  * *Name: String  \- the name of the function*

**Returns**

* Nothing. Completes once the operation is acknowledged by the server.

**Throws**

* EventingFunctionNotFound / EventingFunctionNotDeployed (see MB-47840)  
* EventingFunctionDeployed  
* Any exceptions raised by the underlying platform

**Uri**

* `DELETE http://localhost:8096/api/v1/functions/<name>`
* If manager scope is not \*.\* then append `?bucket=<bucket>&scope=<scope>` (Make sure to URL-encode the parameter values).

### `DeployFunction`

Deploys a function (from state undeployed to deployed).

**Signature**

```
void DeployFunction(String name, [DeployFunctionOptions options])
```

**Parameters**

* Required:  
  * *Name: String  \- the name of the function*

**Returns**

* Nothing, completes once the operation is acknowledged by the server

**Throws**

* EventingFunctionNotFound  
* EventingFunctionNotBootstrapped  
* Any exceptions raised by the underlying platform

**Uri**

* `POST http://localhost:8096/api/v1/functions/<name>/deploy`
* If manager scope is not \*.\* then append `?bucket=<bucket>&scope=<scope>` (Make sure to URL-encode the parameter values).

### `GetAllFunctions`

Lists all functions (both deployed and undeployed) in the manager’s scope.

**Signature**

```
Iterable<EventingFunction> GetAllFunctions([GetAllFunctionsOptions options])
```

**Parameters**

* Required: none

**Returns**

* An Iterable of \<EventingFunction\>; one entry per matching function in the HTTP response.  

> [!IMPORTANT]
> The SDK MUST filter the result and exclude items that do not match the manager’s scope. See [Client-Side Scope Filtering](#client-side-scope-filtering) for more information.

**Throws**

* Any exceptions raised by the underlying platform

**Uri**

* `GET http://localhost:8096/api/v1/functions`

### `GetFunction`

Fetches a specific function.

**Signature**

```
EventingFunction GetFunction(String name, [GetFunctionOptions options])
```

**Parameters**

* Required:  
  * *Name: String  \- the name of the function*

**Returns**

* A EventingFunction if found

**Throws**

* EventingFunctionNotFound  
* Any exceptions raised by the underlying platform

**Uri**

* `GET http://localhost:8096/api/v1/functions/<name>`
* If manager scope is not \*.\* then append `?bucket=<bucket>&scope=<scope>` (Make sure to URL-encode the parameter values).

### `PauseFunction`

Pauses a function.

**Signature**

```
void PauseFunction(String name, [PauseFunctionOptions options])
```

**Parameters**

* Required:  
  * *Name: String  \- the name of the function*

**Returns**

* Nothing, completes once the operation is acknowledged by the server

**Throws**

* EventingFunctionNotFound  
* EventingFunctionNotBootstrapped  
* Any exceptions raised by the underlying platform

**Uri**

* `POST http://localhost:8096/api/v1/functions/<name>/pause`
* If manager scope is not \*.\* then append `?bucket=<bucket>&scope=<scope>` (Make sure to URL-encode the parameter values).

### `ResumeFunction`

Resumes a function.

**Signature**

```
void ResumeFunction(String name, [ResumeFunctionOptions options])
```

**Parameters**

* Required:  
  * *Name: String  \- the name of the function*

**Returns**

* Nothing, completes once the operation is acknowledged by the server

**Throws**

* EventingFunctionNotFound (see MB-47840)  
* EventingFunctionNotDeployed  
* Any exceptions raised by the underlying platform

**Uri**

* `POST http://localhost:8096/api/v1/functions/<name>/resume`
* If manager scope is not \*.\* then append `?bucket=<bucket>&scope=<scope>` (Make sure to URL-encode the parameter values).

### `UndeployFunction`

Undeploys a function (from state deployed to undeployed).

**Signature**

```
void UndeployFunction(String name, [UndeployFunctionOptions options])
```

**Parameters**

* Required:  
  * *Name: String  \- the name of the function*

**Returns**

* Nothing, completes once the operation is acknowledged by the server

**Throws**

* EventingFunctionNotFound (see MB-47840)  
* EventingFunctionNotDeployed  
* Any exceptions raised by the underlying platform

**Uri**

* `POST http://localhost:8096/api/v1/functions/<name>/undeploy`
* If manager scope is not \*.\* then append `?bucket=<bucket>&scope=<scope>` (Make sure to URL-encode the parameter values).

### `UpsertFunction`

Inserts or updates a function.

> [!IMPORTANT]
> `ScopeEventingFunctionManager` MUST add a `"function_scope"` property as a top-level field of the JSON representation of the function before posting it to the server. The field looks like this:

```json
{
  "function_scope": {  
    "bucket":"<bucketName>",
    "scope":"<scopeName>"
  },
  ...
}
```

where `<bucketName>` and `<scopeName>` come from the manager's scope.

**Signature**

```
void UpsertFunction(EventingFunction eventingFunction, [UpsertFunctionOptions options])
```

**Parameters**

* Required:  
  * *Function: the eventing function that should be updated*  
    * *See the important note in the above method description*  
* Optional:  
  * Timeout or timeoutMillis (int/duration) \- the time allowed for the operation to be terminated. This is controlled by the client. If not provided, the default is 75s like other management APIs.

**Returns**

* Nothing, completes once the operation is acknowledged by the server

**Throws**

* EventingFunctionCompilationFailure  
* CollectionNotFound  
* BucketNotFound  
* EventingFunctionIdenticalKeyspace  
* Any exceptions raised by the underlying platform

**Uri**

* `POST http://localhost:8096/api/v1/functions/<name>`
* If manager scope is not \*.\* then append `?bucket=<bucket>&scope=<scope>` (Make sure to URL-encode the parameter values).

### `FunctionsStatus`

Receives the status of all the eventing functions in the manager’s scope.

**Signature**

```
EventingFunctionsStatus FunctionsStatus([FunctionsStatusOptions options])
```

**Parameters**

* Required: none

**Returns**

* The EventingFunctionsStatus structure as described below in its own section  

> [!IMPORTANT]
> The SDK MUST filter the result and exclude items that do not match the manager’s scope. See [Client-Side Scope Filtering](#client-side-scope-filtering) for more information.

**Throws**

* Any exceptions raised by the underlying platform

**Uri**

* `GET http://localhost:8096/api/v1/status`

## EventingFunction

This section defines the individual properties and semantics of the EventingFunction object.

> [!NOTE]
> See the [JSON Schema](#json-schema) section for a link to the handler JSON spec. Some properties and structures are renamed in the SDK API to provide a better user experience.

The JSON representation of the EventingFunction contains a deploymentConfig object field which is not present in the SDK representation. The metadataKeyspace, sourceKeyspace, bucketBindings, urlBindings, constantBindings belong to this object which can be seen in the table below.

The JSON representation has a “function\_scope” field, but this must not be part of the SDK representation. In the SDK, function scope is \_contextual\_; in other words, the scope is determined by the scope of the manager accessing the function.

When parsing the JSON representation of an eventing function, the SDK must treat the absence of a function scope field the same as if the function were explicitly in the \*.\* scope. This ensures compatibility with pre-7.1 servers.

```
class EventingFunction {  
    name: String  
    code: String  
    version: String  
    enforceSchema: Boolean  
    handlerUuid: Integer  
    functionInstanceId: String

    metadataKeyspace: EventingFunctionKeyspace  
    sourceKeyspace: EventingFunctionKeyspace

    bucketBindings: List<EventingFunctionBucketBinding>  
    urlBindings: List<EventingFunctionUrlBinding>  
    constantBindings: List<EventingFunctionConstantBinding>

    settings: EventingFunctionSettings  
}
```

| Property | JSON Attribute | JSON Type | Required | Readonly |
| :---- | :---- | :---- | :---- | :---- |
| name | appname | String | true | false |
| code | appcode | String | true | false |
| settings | settings | Object | true (the object needs to contain deployment\_status and processing\_status at least) | false |
| version | version | String | false (server will fill in) | true |
| enforceSchema | enforce\_schema | Boolean | false | false |
| handlerUuid | handleruuid | Number | false | true |
| functionInstanceId | function\_instance\_id | String | false | true |
| metadataKeyspace | depcfg.metadata\_bucket, depcfg.metadata\_scope, depcfg.metadata\_collection, | String, String, String | true, false, false | false |
| sourceKeyspace | depcfg.source\_bucket, depcfg.source\_scope, depcfg.source\_collection | String, String, String | true, false, false | false |
| constantBindings | depcfg.constants | Array | false | false |
| bucketBindings | depcfg.buckets | Array | false | false |
| urlBindings | depcfg.curl | Array | false | false |

Reference JSON (Server 7.0) payload:

```json
{  
  "appname": "my_function",  
  "appcode": "...",  
  "version": "evt-7.0.0-5302-ee",  
  "enforce_schema": false,  
  "handleruuid": 2576578203,  
  "function_instance_id": "*si6d",  
  "depcfg": {...},  
  "settings": {...},  
}
```

### Eventing Function Bindings

The spec for the deployment config can be found online at https://github.com/couchbase/eventing/blob/cheshire-cat/parser/depcfg_schema.json.

Reference JSON (Server 7.0) payload:

```json
"depcfg": {  
  "source_bucket": "travel-sample",  
  "source_scope": "_default",  
  "source_collection": "_default",

  "metadata_bucket": "foo",  
  "metadata_scope": "_default",  
  "metadata_collection": "_default",

  "buckets": [{...}, {...}], // bucket bindings  
  "curl": [{...}, {...}], // url bindings  
  "constants": [{...}, {...}], // constant bindings  
}
```

Every function can have 0 to N bindings. Right now there are three different bindings: buckets, urls and constants. They are reflected as three different “instances” of the EventingFunctionBinding interface and are flattened at the protocol level.

#### EventingFunctionBucketBinding

```
class EventingFunctionBucketBinding {  
    alias: String  
    name: EventingFunctionKeyspace  
    access: EventingFunctionBucketAccess  
}
```

```
enum EventingFunctionBucketAccess {  
    ReadOnly,  
    ReadWrite  
}
```

| Property | JSON Attribute | JSON Type | Required |
| :---- | :---- | :---- | :---- |
| keyspace | bucket_name, scope_name, collection\_name | String, String, String | true, false, false |
| alias | alias | String | true |
| access | access | String | true |

`EventingFunctionBucketAccess` maps as follows:

* ReadOnly \=\> "r"  
* ReadWrite \=\> "rw"

Reference JSON (Server 7.0) payload:

```
{  
  "alias": "mybucketbinding",  
  "bucket_name": "travel-sample",  
  "scope_name": "inventory",  
  "collection_name": "airline",  
  "access": "r"  
}
```

#### EventingFunctionUrlBinding

```
class EventingFunctionUrlBinding {  
    hostname: String  
    alias: String  
    auth: EventingFunctionUrlAuth  
    allowCookies: boolean  
    validateSslCertificate: boolean  
}
```

The `EventingFunctionUrlAuth` can be implemented a couple different ways depending on the programming language. If the language supports inheritance as a first class concept, the following format is preferred.

```
interface EventingFunctionUrlAuth {}

class EventingFunctionUrlNoAuth \< EventingFunctionUrlAuth {}

class EventingFunctionUrlAuthBasic \< EventingFunctionUrlAuth {  
    username: String  
    password: String  
}

class EventingFunctionUrlAuthDigest \< EventingFunctionUrlAuth {  
    username: String  
    password: String  
}

class EventingFunctionUrlAuthBearer \< EventingFunctionUrlAuth {  
    key: String  
}
```

In languages with “proper” enums a format like this can be used:

```
enum EventingFunctionUrlAuth {  
  None,  
  Basic(username: String, password: String),  
  Digest(username: String, password: String),  
  Bearer(key: String)  
}
```

And finally in languages that do not support either, a format similar to the JSON representation can be used:

```
interface EventingFunctionUrlAuth {  
 Method(): String  
 Username(): Option<String>  
 Password(): Option<String>  
 Key(): Option<String>  
}
```

| Property | JSON Attribute | JSON Type | Required |
| :---- | :---- | :---- | :---- |
| hostname | hostname | String | true |
| alias | value | String | true |
| auth | auth\_type | String | true |
| allowCookies | allow\_cookies | Boolean | true |
| validateSslCertificate | validate\_ssl\_certificate | Boolean | true |
| username | username | String | false |
| password | password | String | false |
| key | bearer\_key | String | false |

The auth property maps as follows:

* NoAuth \=\> "no-auth"  
* AuthBasic \=\> "basic"  
* AuthDigest \=\> "digest"  
* AuthBearer \=\> "bearer"

Reference JSON (Server 7.0) payload:

```json
{  
  "hostname": "http://couchbase.com",  
  "value": "myurlalias",  
  "auth_type": "no-auth",  
  "allow_cookies": true,  
  "validate_ssl_certificate": true,

  "username": "",  
  "password": "*****",  
  "bearer_key": "*****"  
}
```

As can be seen, the server always returns username, password and bearer\_key independent of which auth\_type is actually used (yes, the \*\*\* stars are actually returned in the payload). The \*\*\* fields should be ignored and treated as not present, since they do not convey any value to the user. Consequently the decision which type needs to be used must be based on the auth\_type value field and not derived from the present fields.

#### EventingFunctionConstantBinding

```
class EventingFunctionConstantBinding {  
    alias: String  
    literal: String  
}
```

| Property | JSON Attribute | JSON Type | Required |
| :---- | :---- | :---- | :---- |
| alias | value | String | true |
| literal | literal | String | true |

Reference JSON (Server 7.0) payload:

```json
{  
  "value": "myconstantalias",  
  "literal": "myvalue"  
}
```

### EventingFunctionSettings

The spec for the deployment config can be found online at https://github.com/couchbase/eventing/blob/cheshire-cat/parser/settings_schema.json.

Please note that the following list of settings is way more exhaustive than what is provided in the UI. This is intentional.

```
class EventingFunctionSettings {  
    cppWorkerThreadCount: Integer  
    dcpStreamBoundary: EventingFunctionDcpBoundary  
    description: String  
    deploymentStatus: EventingFunctionDeploymentStatus  
    processingStatus: EventingFunctionProcessingStatus  
    languageCompatibility: EventingFunctionLanguageCompatibility  
    logLevel: EventingFunctionLogLevel  
    executionTimeout: Duration  
    lcbInstCapacity: Integer  
    lcbRetryCount: Integer  
    lcbTimeout: Duration  
    queryConsistency: QueryScanConsistency  
    numTimerPartitions: Integer  
    sockBatchSize: Integer  
    tickDuration: Duration  
    timerContextSize: Integer  
    userPrefix: String  
    bucketCacheSize: Integer  
    bucketCacheAge: Integer  
    curlMaxAllowedRespSize: Integer  
    queryPrepareAll: boolean  
    workerCount: Integer  
    handlerHeaders: List<String>  
    handlerFooters: List<String>  
    enableAppLogRotation: boolean  
    appLogDir: String  
    appLogMaxSize: Integer  
    appLogMaxFiles: Integer  
    checkpointInterval: Duration  
}
```

EventingFunctionDCPBoundary sets what data mutations to deploy the function for.

```
enum EventingFunctionDcpBoundary {  
    Everything  
    FromNow  
}
```

The DCP boundary maps as follows:

* Everything \=\> “everything”  
* FromNow \=\> “from\_now”

```
enum EventingFunctionDeploymentStatus {  
    Deployed,  
    Undeployed  
}
```

The deployment status maps as follows:

* Deployed \=\> true  
* Undeployed \=\> false

```
enum EventingFunctionProcessingStatus {  
    Running,  
    Paused  
}
```

The processing status maps as follows:

* Running \=\> true  
* Paused \=\> false

```
enum EventingFunctionLogLevel {  
    Info,  
    Error,  
    Warning,  
    Debug,  
    Trace  
}
```

The  log level maps as follows:

* Info \=\> “INFO”  
* Error \=\> “ERROR”  
* Warning \=\> “WARNING”  
* Debug \=\> “DEBUG”  
* Trace \=\> “TRACE”

```
enum EventingFunctionLanguageCompatibility {  
    Version_6_0_0,  
    Version_6_5_0,  
    Version_6_6_2,  
    Version_7_2_0,  
}
```

The language compatibility maps as follows:

* Version\_6\_0\_0 \=\> “6.0.0”  
* Version\_6\_5\_0 \=\> “6.5.0”  
* Version\_6\_6\_2 \=\> “6.6.2”  
* Version\_7\_2\_0 \=\> “7.2.0”

The `QueryScanConsistency` is defined separately in SDK 3, the mapping is as follows:

* NOT\_BOUNDED \=\> “none”  
* REQUEST\_PLUS \=\> “request”

This `QueryScanConsistency` is the same one that is also used for query APIs.

Note that all of the following properties are optional.

| Property | JSON Attribute | JSON Type |
| :---- | :---- | :---- |
| cppWorkerThreadCount | cpp\_worker\_thread\_count | Number |
| dcpStreamBoundary | dcp\_stream\_boundary | String |
| description | description | String |
| logLevel | log\_level | String |
| languageCompatibility | language\_compatibility | String |
| executionTimeout | execution\_timeout | Number |
| lcbInstCapacity | lcb\_inst\_capacity | Number |
| lcbRetryCount | lcb\_retry\_count | Number |
| lcbTimeout | lcb\_timeout | Number |
| queryConsistency | n1ql\_consistency | String |
| numTimerPartitions | num\_timer\_partitions | Number |
| sockBatchSize | sock\_batch\_size | Number |
| tickDuration | tick\_duration | Number |
| timerContextSize | timer\_context\_size | Number |
| userPrefix | user\_prefix | String |
| bucketCacheSize | bucket\_cache\_size | Number |
| bucketCacheAge | bucket\_cache\_age | Number |
| curlMaxAllowedRespSize | curl\_max\_allowed\_resp\_size | Number |
| workerCount | worker\_count | Number |
| queryPrepareAll | n1ql\_prepare\_all | Boolean |
| handlerHeaders | handler\_headers | Array (of Strings) |
| handlerFooters | handler\_footers | Array (of Strings) |
| enableAppLogRotation | enable\_applog\_rotation | Boolean |
| appLogDir | app\_log\_dir | String |
| appLogMaxSize | app\_log\_max\_size | Number |
| appLogMaxFiles | app\_log\_max\_files | Number |
| checkpointInterval | checkpoint\_interval | Number |
| deploymentStatus | deployment\_status | Boolean |
| processingStatus | processing\_status | Boolean |

There are two special fields within these settings which **must** be set on the JSON payload, if not set on the settings object then these should default to:

* processing\_status: false (paused)  
* deployment\_status: false (undeployed)

## EventingFunctionKeyspace

A EventingFunctionKeyspace is a triple of the following components:

* Bucket: String (required)  
* Scope: String (optional)  
* Collection: String (optional)

A Bucket is required, but the scope and collection can be optional. If they are not present, their default representation is assumed.

## EventingStatus

The `EventingFunctionsStatus` class encapsulates the status from all eventing functions on the current cluster. It is the language representation of the following JSON exemplary payload received from the server:

```json
{  
 "apps": [  
  {  
   "composite_status": "deployed",  
   "name": "aaa",  
   "num_bootstrapping_nodes": 0,  
   "num_deployed_nodes": 1,  
   "deployment_status": true,  
   "processing_status": true  
  }  
 ],  
 "num_eventing_nodes": 1  
}
```

Which maps to the following structure:

```
class EventingStatus {  
  numEventingNodes: int  
  functions: List<EventingFunctionState>  
}
```

```
class EventingFunctionState {  
	name: String  
	status: EventingFunctionStatus  
	numBootstrappingNodes: int  
	numDeployedNodes: int  
	deploymentStatus: EventingFunctionDeploymentStatus  
	processingStatus: EventingFunctionProcessingStatus  
}
```

```
enum EventingFunctionStatus {  
	Undeployed  
	Deploying  
	Deployed  
	Undeploying  
	Paused  
	Pausing  
}
```

The EventingFunctionStatus mapping from JSON is specified as follows:

| State | JSON Property |
| :---- | :---- |
| Undeployed | “undeployed” |
| Deploying | “deploying” |
| Deployed | “deployed” |
| Undeploying | “undeploying” |
| Paused | “paused” |
| Pausing | “pausing” |

> [!NOTE]
> The JSON representation for EventingFunctionState has a `“function_scope”` field, but this must not be part of the SDK representation. In the SDK, function scope is _contextual_, in other words, the scope is determined by the scope of the manager accessing the function.

When parsing the JSON representation for EventingFunctionState, the SDK must treat the absence of a function scope field the same as if the function were explicitly in the \*.\* scope. This ensures compatibility with pre-7.1 servers.

## Tracing Spans

The name of the trace spans are defined in the [Tracing RFC](/rfc/0067-extended-sdk-observability-rto-v2.md). Here is a duplicate of the names for convenience:

| Operation | Span name |
| :-------- | :-------- |
| UpsertFunction | manager_eventing_upsert_function |
| GetFunction | manager_eventing_get_function |
| DropFunction | manager_eventing_drop_function |
| DeployFunction | manager_eventing_deploy_function |
| GetAllFunctions | manager_eventing_get_all_functions |
| PauseFunction | manager_eventing_pause_function |
| ResumeFunction | manager_eventing_resume_function |
| UndeployFunction | manager_eventing_undeploy_function |
| FunctionsStatus | manager_eventing_functions_status |

# Error Reference

Eventing returns errors with different HTTP codes and a JSON payload. Here is an example payload:

```json
{  
 "name": "ERR_APP_ALREADY_DEPLOYED",  
 "code": 20,  
 "description": "Function already deployed",  
 "attributes": null,  
 "runtime_info": {  
  "code": 20,  
  "info": "Function: myfunc another function with same name is already deployed, skipping save request"  
 }  
}
```

Note that the “info” block can also be an object, so the deserialization at runtime must account for that:

```json
{
  "name": "ERR_HANDLER_COMPILATION",
  "code": 27,
  "description": "Function compilation failed",
  "attributes": null,
  "runtime_info": {
    "code": 27,
    "info": {
     "area": "handlerCode",
     "column_number": 19,
     "compile_success": false,
     "description": " doc\"ip_num_start\"] =     get_numip_first_3_octets(doc[\"ip_start\"]);\n ^^^^^^^^^^^^^^\nSyntaxError: Unexpected string\n",
     "index": 75,
     "language": "JavaScript",
     "line_number": 3
    }
  }
}
```

The following list enumerates all errors which are uniquely numbered (and are going to be cross referenced from the error handling RFC once accepted).

* 608 `EventingFunctionNotFound`: name is ERR\_APP\_NOT\_FOUND\_TS  
  * Occurs if the function is not found  
* 609 `EventingFunctionNotDeployed`: name is ERR\_APP\_NOT\_DEPLOYED  
  * Occurs if the function is not deployed, but the action expects it to  
* 610 `EventingFunctionCompilationFailure`: name is ERR\_HANDLER\_COMPILATION  
  * Occurs when the compilation of the function code failed  
* 611 `EventingFunctionIdenticalKeyspace`: name is ERR\_SRC\_MB\_SAME  
  * Occurs when source and metadata keyspaces are the same.  
* 612 `EventingFunctionNotBootstrapped`: name is ERR\_APP\_NOT\_BOOTSTRAPPED  
  * Occurs when a function is deployed but not “fully” bootstrapped  
* 613 `EventingFunctionDeployed`: name is ERR\_APP\_NOT\_UNDEPLOYED or ERR\_APP\_ALREADY\_DEPLOYED  
  * Occurs when a function is deployed but the action does not expect it to. For instance when dropping deployed function, or resuming already deployed function.  
* 614 `EventingFunctionPaused`: name is ERR\_APP\_PAUSED  
  * Occurs when a function is paused, but the action does not expect it to. For example, when pausing a function, that is already in "paused" state.  
* 11 `CollectionNotFound`: name is ERR\_COLLECTION\_MISSING  
  * The collection in the keyspace is not found by the eventing service  
* 10 `BucketNotFound`: name is ERR\_BUCKET\_MISSING  
  * The bucket in the keyspace is not found by the eventing service  

Note that the error name is used for matching instead of the code since it allows you to cheaply figure out what the error is without having to parse it into JSON. It is also possible to use the code as an alternative.

Any other / unknown errors must still be surfaced to the user in the generic parent exception type including the payload from above to give the user a chance to debug the issue.

# Appendix A: Adding support for eventing function scope

This section describes how to add support for scoped eventing functions to an SDK that already implemented a previous revision of this specification.

## Add ScopeEventingFunctionManager

Add a `ScopeEventingFunctionManager` class with the same methods as `EventingFunctionManager`. The two classes MUST NOT have an inheritance relationship, because in the future we may add methods to one class but not the other. To limit code duplication, the two classes SHOULD delegate to a common internal implementation class, using a platform-idiomatic strategy.

Add a `Scope.eventingFunctions()` method that returns a `ScopeEventingFunctionManager` for the scope.  
This method SHOULD be documented as requiring Couchbase Server 7.1 or later.

Add a private “function scope” property to `ScopeEventingFunctionManager`. A function scope has two properties: bucket name and scope name. These names come from the `Scope` used to access the manager.

TIP: You might find it convenient to add the scope property to the common internal implementation class. In that case, the SDK SHOULD use the explicit function scope \*.\* for the cluster-level `EventingFunctionManager`.

## DO NOT add scope property to EventingFunction

From the user's perspective, scope is purely contextual, and depends on the manager used to access the function. The SDK's EventingFunction class MUST NOT have a scope property. 

## Inject scope when upserting

When upserting a function, ScopeEventingFunctionManager MUST add a "function\_scope" property as a top-level field of the JSON representation of the function. The field looks like this:

```json
{
  "function_scope": {
    "bucket":"<bucketName>",
    "scope":"<scopeName>"
  },
  ...
}
```

where `<bucketName>` and `<scopeName>` come from the manager's scope.

The non-scoped `EventingFunctionManager` MUST NOT add this field. In other words, add this field only if the manager’s scope is not “\*.\*”.

## Filter out items from other scopes

The HTTP API does not currently have a way to limit the results of `GetAllFunctions` and `FunctionStatus` to specific scopes.

Both the cluster-level and scope-level managers MUST filter the results of `GetAllFunctions` and `FunctionStatus`, and retain only the results where the JSON has a "function\_scope" field matching the manager's scope.

For compatibility with pre-7.1 servers, the non-scoped `EventingFunctionManager` manager MUST retain items with an explicit scope of \*.\* and also items that have no "function\_scope" field.

## Add URI query parameters

`ScopeEventingFunctionManager` MUST append the following URI query parameters to all HTTP requests except `GetAllFunctions` and `FunctionStatus`:

```
?bucket=<bucketName>&scope=<scopeName>
```

Query parameter values MUST be URL-encoded.

`EventingFunctionManager` MUST NOT add these query parameters. In other words, add the parameters only if the manager’s scope is not “\*.\*”.

## Feature availability

The server does not provide a practical way to detect whether eventing function scope is supported. The SDK should not attempt to detect the presence of this feature, and should not attempt to throw `FeatureNotAvailable` when it is absent.

# Changes

* Revision \#14 (2024-03-18 \- David Nault):  
  * Added language compatibility “7.2.0” to EventingFunctionLanguageCompatibility.  
  * Noted that URI query parameter values MUST be url-encoded.  
* Revision \#13 (2024-03-15 \- David Nault):  
  * Added Appendix A, a guide to adding support for function scope.  
  * Added a note that Capella is out of scope.  
  * Added a “Concepts” heading, with subheadings describing function scope (added in Couchbase Server 7.1) and scoped eventing function managers.  
  * Moved the Eventing HTTP API documentation link from the EventingFunctionManager heading to the top-level References heading.  
  * Added an “Options” heading so the optional \`timeout\` parameter is described in only one place.  
* Revision \#12 (2021-10-04 \- Michael Nitschinger):  
  * Removed unused default column  
  * Renamed EventingFunctionBucketBinding property name to keyspace  
* Revision \#11 (2021-08-24 \- Michael Nitschinger):  
  * Renamed EventingFunctionsStatus to EventingStatus  
  * Switched the enum to EventingFunctionStatus and the outer class to EventingFunctionState based on bretts comment, which makes total sense.  
  * Clarified that the \*\*\* return from the auth fields should be ignored and treated as not present.  
  * Added a comment to clarify that the number of settings \> what exposed in the UI is intentional  
  * Added BucketNotFound for upsertFunction (if the bucket name does not exist)  
  * Turned TickDuration and CheckpointInterval into Duration  
  * Switched HandlerUUID to integer  
  * Removed deploymentConfig from the table since it is scattered throughout the other properties and not a single entity  
  * Marked properties version, handler uuid and function instance id as readonly, since the server fills it in once created  
* Revision \#10 (2021-08-23 \- Michael Nitschinger):  
  * Classified on the default 75s timeout if not provided otherwise on each timeout param  
  * Linked the eventing server API in the document  
  * Added EventingFunctionIdenticalKeyspace to upsertFunction  
* Revision \#9 (2021-08-15 \- Michael Nitschinger):  
  * Renamed EventingFunctionNotUndeployed to EventingFunctionDeployed  
  * Added EventingFunctionsStatus  
* Revision \#8 (2021-08-12 \- Michael Nitschinger):  
  * Changed Keyspace to EventingFunctionKeyspace and it’s not considered a “global” entity anymore (now specific to the eventing management API).  
  * Clarified on why the error name is used instead of the error code.  
* Revision \#7 (2021-08-10 \- Michael Nitschinger):  
  * Added list of errors and added them to each function  
* Revision \#6 (2021-08-04 \- Michael Nitschinger):  
  * Merge createFunction and updateFunction into upsertFunction since they really do the exact same thing  
  * Move the properties out of the EventingFunctionDeploymentConfig into the top level EventingFunction  
  * Added a section on tracing span names  
* Revision \#5 (2021-08-03 \- Michael Nitschinger):  
  * Clarified that only  7.0 and later is supported.  
  * Reordered and reorganized functions by alphabet so they are easier to locate.  
  * Reworked and cleaned up the EventingFunction spec including all sub-components.  
  * Rename DeleteFunction back to DropFunction for consistency reasons  
* Revision \#4 (2021-04-28 \- Michael Nitschinger):  
  * Significantly overhauled the function structure based off of the newly created spec by the eventing team  
* Revision \#3 (2020-07-07 \- Michael Nitschinger):  
  * Renamed DropFunction to DeleteFunction  
  * Added PauseFunction  
  * Added ResumeFunction  
* Revision \#2 (2020-02-03 \- Michael Nitschinger):  
  * Fleshed out the API, added all primary accessors of the initial draft  
* Revision \#1 (2020-01-31 \- Michael Nitschinger):  
  * Initial Version

# Signoff

| SDK | Representative | Date | Revision |
| :---- | :---- | :---- | :---- |
| C++ | Sergey Avseyev |  |  |
| Ruby | Sergey Avseyev |  |  |
| Go | Charles Dixon | 2024-03-24 | 14 |
| Rust | Charles Dixon | | |
| Java  | David Nault | 2024-03-18 | 14 |
| Kotlin | David Nault | 2024-03-18 | 14 |
| .NET | Jeff Morris |  |  |
| NodeJS | Jared Casey |  |  |
| PHP | Sergey Avseyev |  |  |
| Python | Jared Casey |  |  |
| Scala | Graham Pople |  |  |
