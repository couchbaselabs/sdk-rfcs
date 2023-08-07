# Meta

- RFC Name: SDK3 Foundation
- RFC ID: 0059-sdk3-foundation
- Start Date: September 14, 2019
- Owner: Brett Lawson \<brett@couchbase.com\>
- Current Status: ACCEPTED
- [Original Google Drive Doc](https://docs.google.com/document/d/1pt8wrSu7xvaqjG5vxcQSZN1epw6oP4MyTcZZMvSYwQo)

# Motivation

Provide a specification for an SDK API for Couchbase Server 5.0 and greater where features such as durability, collections (possibly as a DP first), and eventing features like Feeds and watches arrive and need to fit in the API.

# API Concepts

## New Scope and Collection Resources

The Couchbase 3.0 SDKs support the new Collection paradigm and the natural hierarchy of resources and actions which can be done against Couchbase Server 6.5/7.

Couchbase has traditionally used the concepts of Cluster as a sequence of interjoined nodes and Buckets as unit of multi tenancy within a Cluster. For Couchbase 6.5/7 the resource paradigm has been augmented to include Scopes, which are groupings of collections, and represent multi tenancy within a Bucket, and Collections within a Scope to segment documents. These are all "resource boundaries" for authentication.

For example, here is an example of connecting to a collection using C#:

```csharp
//create a cluster and connect to it
var cluster = Cluster.connect(
  "couchbase://localhost",
  new ClusterOptions{
    Username="Administrator",
    Password="password"
});

//get the bucket and people collection with scope name
var bucket = cluster.Bucket("default");
var coll = bucket.Scope("comxyz").Collection( "people");
```

And a similar example in Javascript:

```javascript
//create a cluster and connect to it
var cluster = new couchbase.Cluster(
  'couchbase://localhost', {
    username: 'Administrator',
    password: 'password'
});

//get the bucket and people collection with scope name
var bucket = cluster.bucket('default');
var coll = bucket.scope('comxyz').collection('people');
```

In the example above a Cluster object is created and then authenticated. It will use the "default" configuration. After that, the default bucket is fetched from the cluster and then a collection is opened with the scope "comxyz" and the name "people". Once the collection is fetched, it can be used for basic CRUD and a new, terse collection specific query syntax API.

The exact naming and use of the cluster configuration object as left as implementation dependent. For example, Java will use the name Environment instead of Configuration as its implementation involves this object owning some long-running resources.

Also, as outlined in the bootstrapping RFC, the bootstrap process itself is lazy w.r.t error propagation. Errors are deferred into the operation, other than illegal config options on Cluster.connect. This makes sure that the user only has to have error handling in a minimal amount of places.

### Global vs Local Scope

The structure of the API follows the natural path that a developer will encounter as they "dot operator" navigate using their IDE and it's "intellisense" equivalent. This structure also follows the Couchbase resource boundaries defined by their cluster configuration. It also defines a means of building operations with repeatable defaults. In general, the pattern is to apply global attributes or settings on the resource and then overwrite them with finer detail on the more specific resource. Cluster level settings affect all buckets, all collections and finally all operations, however at the operation level these settings can be overridden with new value and then it will apply from that point on.

For example, assume Timeout properties defined on different services in a cluster. This will apply to all resources under that cluster object: buckets, collections and operations.If this needs to be overridden for a single operation, then it can be and will only affect that one operation.

Note that a property may exist at all levels or may be constrained at a resource boundary or operation. For example, a CAS value is specific to the operation level only, while kvTimeout or queryTimeout may be different on throughout the chain.

Finally, some operations against Couchbase Server can be either "global" in that they can be done on the Cluster level or "local" in they can be done on Buckets and Collections level.

## Core API

The core API revolves around using a "builder" pattern to associate zero or more parameters with a single operation while allowing operations to be reusable to minimize redundancy. Required parameters are always on the left hand-side, while optional parameters will be on the right. The method name is a verb that explicitly defines what the action (Insert, Query, etc) will be on the resource (bucket or collection).

### Key/Value API using the Bucket and Collection

The basic format for a Key Value (K/V) method has this basic signature:

`Result Verb(id, [content], [verbOptions])`

Where:

- `Verb` is the method named (Get, Insert, etc) by operation to execute by the server. Note the name may not match the exact Binary Memcached command, for example a Set command is the method named Insert in the API.  It has been this way since the 2.x series of SDK releases to align with the N1QL/SQL paradigm.

- `id` is the required key or identifier for the data stored or retrieved from Couchbase server.

- `content` is the actual data stored in Couchbase server for a given key. For some options, it may be not present depending upon the operation.

- `options` are the optional metadata associated with an operation: timeout, expiration, CAS etc. They are specific to Memcached Binary (extended) commands with the exception of timeout -  a general options object is not permitted. If no options are provided, null or an equivalent overload is permissible excluding this parameter. Named parameters or an options object are permitted.

- `Result` is the response from the server and includes the status of a given operation, value, CAS value, etc.

K/V operations are resource level operations meaning that they exist only on the collection level. Performing a K/V operation on the bucket or cluster object is not allowed. Note that if using an older server like 5.5.0 that does not support collections, the "default" collection will be used - which will actually just use the bucket - hiding the fact that the server does not support collections.

For languages that support parametric polymorphism (generics, templates, etc), providing a Type of T for translating the document body is acceptable and falls under "Idiomatic Concerns".

Casing and special naming concerns for platforms are acceptable in certain cases. For example, some languages use underscores to separate compound parameter names and other use camel or pascal casing. These deviations from the specification are acceptable if they are a convention or forced by the platform.

### The Builder Pattern and Parameters

Methods for APIs such as Key Value (K/V), Query, etc. use a "builder" pattern where required fields are separate from optional parameters, which can either be an object, struct or named parameters, depending upon the language. If an object is used, it should have a structure similar to this:

```csharp
struct XxxOptions{

  ...

  XxxOptions Timeout(int),
  XxxOptions Expiration(int)
}
```

To make it easier for developers to associate builders with specific operations, the options should only contain fields that the operation can use and should have a prefix name indicating the operation. For example:

```csharp
struct GetOptions{
  ...
}
```

In this case, GetOptions only contains the optional parameters that are supported by the memcached Get command. This makes it much easier for developers to only supply parameters that a command will accept at compile time if type safety is supported for a language.

If a language supports named parameters, then the method signature can use named parameters instead of GetOptions - what is required is that all of the optional parameters are defined in the signature, whether they are used or not.

```csharp
result = Get(string id, timeOut=2500, expiration=0);
```

Instead of

```csharp
result =  Get("theKey", new GetOptions().Timeout(new TimeSpan(0, 0, 0, 2500)));`
```

In this case the id parameter is a required parameter; timeOut is an optional parameter with a default that can be overridden by explicitly calling them or an options object which can be omitted if the default options are used.

```csharp
var result = coll.Get("theKey, timeoutMillis:=1000);
```

Here we have explicitly override the 2500ms timeout value with a new value of 1000. Note that the type stored for timeOut is idiomatic specific; for example, C# would be a TimeSpan because that is what a .NET developer would expect. Note that the parameter name timeoutMillis can optionally be named simply timeout if the parameter type represents a platform idiomatic duration or time type.

Note, if a build structure or class is used, it should be based on the fluent interface paradigm where you create an immutable object of parameters and pass that as the options.

### What is Optional<T>?

Optional<T> means that the return result may be a value or null; Optional<T> is a container class which allows for the presence of a value to be determined by using a method that checks for null.

### What Happened to the Document API

The Document API introduced in SDK v2.0 has been removed to simplify the public API and to reduce the number of overloads required for key/value pairs. For example, the CAS value from a previous call can be added to the options block in the next call if it requires optimistic concurrency.

# SDK 3.0 API Definition

## The Cluster interface

The Cluster class/object is the root object for connecting to a remote grouping of Couchbase Servers. The Cluster class allows a configuration to be passed in and connects to a given structure, performing any necessary authentication and server negotiation of the various supported services and features that must be enabled. The Cluster supports the Diagnostics API for determining the health of the cluster and any global actions that can be performed on a service. Additionally, the cluster Management API is also on the Cluster object.

An example of an interface would be as follows:

```csharp
interface ICluster{
  static Cluster Connect(string connStr, ClusterOptions options);

  // Optional overload for simplicity
  static Cluster Connect(string connStr, string Username, string Password);

  IBucket Bucket(string bucketName);

  void Disconnect(DisconnectOptions options=null);

  ...

}
```

### Creation

Must be done via static Connect(..) method:

var cluster = Cluster.Connect(string connstr, ClusterOptions options=null);

Or

var cluster = Cluster.Connect(string connstr, string username, string password);

### Destruction 

Must be either an explicit Disconnect(...) method or a platform specific "Closable" or equivalent:

```csharp
cluster.Disconnect();
```

Or

```csharp
cluster.Dispose();
```

Or

```typescript
cluster.close();
```

### Methods

The following methods must be implemented:

#### Bucket

Returns an IBucket implementation or equivalent object.

Signature

```csharp
IBucket Bucket(string bucketName);
```

Parameters

- Required
  - String bucketName - the name of the bucket whose resources you are using.

Returns

- An IBucket implementation.

Throws

- Never

#### Disconnect (or Close/Dispose)

Closes and cleans up any resources used by the Cluster and any objects it owns. Note the name is platform idiomatic.

Signature

```csharp
void Disconnect([DisconnectOptions])
```

Parameters

- Required

  - None

- Optional
  - DisconnectOptions - any platform specific diagnostics options.  There are no options available today, but the options are reserved for possible use in the future.

Returns

- Void

Throws

- Any exceptions raised by the underlying platform

### Caveats and notes:

- It is acceptable for a Cluster object to have a static Connect method which takes a ConfigOptions or acceptable alternative for ease of use.
- To facilitate testing/mocking, it's acceptable for this structure to derive from an interface at the implementers discretion.
- The "Get" and "Set" prefixes are considered platform idiomatic and can be adjusted to various platform idioms.
- The Configuration is passed in via the ctor; overloads for connection strings and various other platform specific configuration are also passed this way. If a language does not support ctor overloading, then an equivalent method can be used on the object.

## The Bucket interface

The Bucket interface represents a storage unit and allows access to Scopes which represent a unit of multi-tenancy within that storage. Scopes group Collections, which are part of this  multi-tenancy allowing for higher application density over creating Buckets for an application when one user's data must be kept separate from other users' data.

```csharp
interface IBucket{
  string Name();

  IScope Scope(string scopeName);

  ICollection Collection(string collectionName);
  ICollection DefaultCollection();

  ...

}
```

### Creation

Opened by calling the Bucket method on ICluster:

```csharp
var bucket = cluster.Bucket("theBucket");
```

### Methods

#### Name

Gets the name of the bucket.

Signature

```csharp
string Name()
```

Parameters

- None

Returns

- A string name of the bucket.

Throws

- Never

#### Scope

Gets a scope for a given name.

Signature

```csharp
IScope Scope(string scopeName)
```

Parameters

- Required

  - scopeName - a string name for a given scope.

- Optional
  - None

Returns

- An IScope implementation for the scope for a given name.

Throws

- Never

#### Collection

Gets a collection from the default scope by name.

Signature

```csharp
ICollection Collection(string collectionName)
```

Parameters

- Required

  - string collectionName - the name of the collection within the default scope to access.

- Optional
  - None

Returns

- An ICollection implementation for a given name.

Throws

- Never

#### DefaultCollection

Gets the default collection for a given bucket; the Scope associated with the default collection is the default scope.

Signature

```csharp
ICollection DefaultCollection()
```

Parameters

- Required

  - None

- Optional

  - None

Returns

- An ICollection implementation for the default collection.

Throws

- Never

## The IScope interface

The Scope interface provides a means of multi-tenancy by grouping Collections by name. It enables high-application density by removing the chance of a naming collision.

```csharp
interface IScope{

  string Name();

  ICollection Collection(string collectionName);

  ...

}
```

### Creation

Created by calling the Scope method off of a IBucket instance:

```csharp
var scope = bucket.Scope("theScope");
```

### Methods

#### Name

Gets the name of the IScope.

Signature

```csharp
string Name()
```

Parameters

- None

Returns

- A string value that is the name of the collection.

Throws

- Never

#### Collection

Gets an ICollection instance given a collection name.

Signature

```csharp
ICollection Collection(string collectionName)
```

Parameters

- Required:

  - collectionName - string identifier for a given collection.

- Optional

  - None

Returns

- An ICollection implementation for a collection with a given name.

Throws

- Never

## The ICollection interface

The Collection interface is the main programming API for document and sub-document CRUD. Note that the Collection Identifier value is not exposed publicly, but should be logged internally at the INFO level when it has changed.

```csharp
interface ICollection{

  string Name();

  ...

}
```

### Creation

```csharp
var collection = scope.Collection("theCollectionName");
```

### Methods

The following methods must be implemented:

#### Name

Gets a string representing the name of this collection.

Signature

```csharp
string Name()
```

Parameters

- Required

  - None

- Optional
  - None

Returns

- A string containing the name of the collection.

Throws

- Never

## MutationToken

Returned after a mutation if Enhanced Durability is enabled; used to enforce consistency.

```csharp
interface IMutationToken{
  Uint64 PartitionUuid();
  Uint64 SequenceNumber();
  Uint16 PartitionId();
  String BucketName();
}
```

## MutationState

```csharp
interface MutationState {
  private constructor();

  static from(tokens: ...MutationToken...) -> MutationState
  add(tokens: ...MutationToken) -> MutationState
}
```

## Configuration

Configuration is handled by top-level properties that apply to all operations and can be overridden at the operation level via the options parameters. All configuration settings have sensible defaults that apply when they are not overridden.  Each entry in the table below contains two names, the first being the name which is expected to be use with the SDKs options structures, and the second is the name as it should appear in a connection string.

#### KVConnectTimeout

Number of seconds the client should wait while attempting to connect to a node's KV service via a socket. Initial connection, reconnecting, node added, etc.

- Connection String: `kv_connect_timeout`
- Type: `Duration`
- Default: `10s`

#### KVTimeout

Number of milliseconds to wait before timing out a KV operation by the client.

- Connection String: `kv_timeout`
- Type: `Duration`
- Default: `2.5s`

#### KVDurableTimeout

Number of milliseconds to wait before timing out a KV operation that is either using synchronous durability or observe-based durability.

- Connection String: `kv_durable_timeout`
- Type: `Duration`
- Default: `10s`

#### ViewTimeout

Number of seconds to wait before timing out a View request by the client..

- Connection String: `view_timeout`
- Type: `Duration`
- Default: `75s`

#### QueryTimeout

Number of seconds to wait before timing out a Query or N1QL request by the client.

- Connection String: `query_timeout`
- Type: `Duration`
- Default: `75s`

#### AnalyticsTimeout

Number of seconds to wait before timing out an Analytics request by the client.

- Connection String: `analytics_timeout`
- Type: `Duration`
- Default: `75s`

#### SearchTimeout

Number of seconds to wait before timing out a Search request by the client.

- Connection String: `search_timeout`
- Type: `Duration`
- Default: `75s`

#### ManagementTimeout

Number of seconds to wait before timing out a Management API request by the client.

- Connection String: `management_timeout`
- Type: `Duration`
- Default: `75s`

#### EnableTLS

Whether or not to use TLS/SSL for all client network interaction with server. This is for app.config or programmatic config. Connection string scheme takes precedence.

- Connection String: `enable_tls`
- Type: `boolean`
- Default: `false`

#### Transcoder

Configures the default transcoder to use for various operations.

- Connection String: N/A
- Type: `Transcoder`
- Default: `new JSONTranscoder()`

#### JsonSerializer

Configures the default serializer to use for various operations.

- Connection String: N/A
- Type: `JsonSerializer`
- Default: `new DefaultJsonSerializer()`

#### Tracer

Configures which tracer to use.

- Connection String: N/A
- Type: `Tracer`
- Default: `new ThresholdLoggingTracer()`

#### EnableMutationTokens

Request mutation tokens at connection negotiation time. Turning this off will save 16 bytes per operation response.

- Connection String: `enable_mutation_tokens`
- Type: `boolean`
- Default: `true`

#### TcpKeepAliveTime

Specifies the timeout, in milliseconds, with no activity until the first keep-alive packet is sent. This applies to all services, but is advisory: if the underlying platform does not support this on all connections, it will be applied only on those it can be.

- Connection String: `tcp_keepalive_time`
- Type: `Duration`
- Default: `60s`

#### EnableTcpKeepAlives

Gets or sets a value indicating whether enable TCP keep-alives.

- Connection String: `enable_tcp_keepalives`
- Type: `boolean`
- Default: `true`

#### ForceIPV4

Sets the SDK configuration to do IPv4 Name Resolution on underlying platform libraries.

- Connection String: `force_ipv4`
- Type: `boolean`
- Default: `false`

#### ConfigPollInterval

- Connection String: `config_poll_interval`
- Type: `Duration`
- Default: `2.5s`

#### ConfigPollFloorInterval

- Connection String: `config_pool_floor_interval`
- Type: `Duration`
- Default: `50ms`

#### ConfigIdleRedialTimeout

Specifies the period of time to wait after the last configuration is received on a streaming HTTP configuration connection before the SDK forces the HTTP connection to reconnect.

- Connection String: `config_idle_redial_timeout`
- Type: `Duration`
- Default: `5m`

#### NumKvConnections

The number of KV connections to establish to each node for each bucket.

- Connection String: `num_kv_connections`
- Type: `integer`
- Default: `1`

#### MaxHttpConnections

The maximum number of HTTP connections allowed on a per-host and per-port basis. 0 indicates an unlimited number of connections are permitted.

- Connection String: `max_http_connections`
- Type: `integer`
- Default: `0`

#### IdleHttpConnectionTimouet

The period of time an HTTP connection can be idle before it is forcefully disconnected.

- Connection String: `idle_http_connection_timeout`
- Type: `Duration`
- Default: `1s`

> Any time above 4.5s should log an INFO level warning indicating that the SDK may produce trivial warnings due to the time being set above the idle timeout of various services.

## Logging and Error Handling

In SDK 2.0, errors that occur while creating clusters or opening buckets ("bootstrap" exceptions/errors") where immediately raised or thrown if fatal (cannot connect, invalid password, etc). In SDK 3.0 this has changed; errors are handled and propagated to the operation level (K/V, Query, Analytics, FTS or View), for example when doing an insert or delete of a key and document.

- The connections or any resource allocations (server negotiation, etc) required can be done asynchronously when the cluster or bucket is created, however, any error or exceptions should be handled internally (cached) and propagated when the first operation is called.

- A special Initialize() method which is optional, can be implemented so connections can be made and resources "warmed up" so that errors can be raised before the initial operation is called. This method may provide additionally ping or diagnostics information to the caller

Specific information about error types and behaviours can be found in the Error Handling RFC.

# Terminology

Pipelines versus endpoints versus connections.  Goal: have a common Couchbase term that may have idiomatic implementations.

Idiomatic (Concerns/deviation): using language or platform features to augment the existing, defined specification for an implementation of the SDK.

### Types

The following represents a list of types which should be implemented by each SDK with their appropriate idiomatic types.  Note that the SDK is not responsible for reproducing these names faithfully, but instead they are simply used to accurately describe the underlying functionality.

- `Int32`: A 32-bit signed integer

- `Uint32`: A 32-bit unsigned integer.

- `Int64`: A 64-bit signed integer.

- `Uint64`: A 64-bit unsigned integer.

- `Duration`: A datatype representing a duration in time.  This could be an integer in libcouchbase representing the number of milliseconds, or a time.Duration in Golang.

- `String`: A string representing a group of characters.

- `Stream`: A stream represents the functionality of receiving a number of pieces of data over a period of time, where the application is able to receive the data as it is received.  The stream must have an explicit and detectible ending point.

- `Promise`: A promise represents the functionality to receive a result at a later point in time once other preconditions have been met.

# Appendix

## LEB128 Test Data

The KV team has provided some test data that describes a range of CID values and the expected encoded byte array. The test data can be seen here: <https://github.com/couchbase/kv_engine/blob/master/docs/Collections.md#leb128-encoded-examples>

## Interface Taxonomy and Stability

Couchbase uses four interface stability classifiers. You may find these classifiers appended as annotations or comments within documentation for each API:

- **Committed**: This stability level is used to indicate the most stable interfaces that are guaranteed to be supported and remain stable between SDK versions.

- **Uncommitted**: This level is used to indicate APIs that are unlikely to change, but may still change as final consensus on their behavior has not yet been reached. Uncommitted APIs usually end up becoming stable APIs.

- **Volatile**: This level is used to indicate experimental APIs that are still in flux and may likely be changed. It may also be used to indicate inherently private APIs that may be exposed, but "YMMV" (your mileage may vary) principles apply. Volatile APIs typically end up being promoted to Uncommitted after undergoing some modifications.

- **Internal**: This level is used to indicate you should not rely on this API as it is not intended for use outside the module, even to other Couchbase components.

APIs that are marked as Committed have a stable implementation. Uncommitted and Volatile APIs should be stable within the bounds of any known and often documented issues, but Couchbase has not made a commitment to these APIs and may not respond to reported defects with the same priority.

# References

- Binary memcached protocol
  <https://github.com/couchbase/memcached/blob/master/docs/BinaryProtocol.md>

- Collections design doc
  <https://docs.google.com/document/d/1X-v8GWQjplrMMaYwwWOzEuP4AUoDNIAvS39NmEjQ3_E/edit?ts=5bbc638d>

- Getting started with current build (as of 10/9/2018 - no REST API)
  <https://docs.google.com/presentation/d/1hh3Nz9uB3AIxLUWwL0t8Qk9ZvNazi1Xl3yuO6s4Wkvs/edit#slide=id.g42bdc1bb21_0_7>

- LEB128
  <https://en.wikipedia.org/wiki/LEB128>

- Virtual XAttrs
  <https://docs.couchbase.com/server/6.0/learn/data/extended-attributes-fundamentals.html#virtual-extended-attributes>

- Sub doc commands
  <https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit>

- Synchronous Replication
  <https://docs.google.com/document/d/1DCIzCNXmm3XjkylF19zwNvxcwJFp3bLqSIsQZzTakZ0/edit>

- N1QL Error Codes
  <https://docs.google.com/spreadsheets/d/1tua4Rwrs6Kb-2KlvxixwJHhNxLjPmjnXRyYbujworg0/edit#gid=751451640>

- Query REST API
  <https://docs.google.com/document/d/1220AjacedEbzzboFp6gW5Savz912nPHe09wQQJUj0To/edit#heading=h.lfqenz86v2rl>

- Collection Changes
  <https://github.com/couchbase/kv_engine/commit/fab8284ef4e1218186c8ee11a81c64ceb581a6a2>

- Internal Wiki page
  <https://hub.internal.couchbase.com/confluence/pages/viewpage.action?spaceKey=cbeng&title=Collections>

# Changelog

- Sept 14, 2019 - Revision #1 (by Jeff Morris)

  - Initial Draft

- Sept 26, 2019 - Revision #2 (by Brett Lawson)

  - Added notes regarding the structure and behaviour of MutationToken and MutationState.

- Oct 4, 2019 - Revision #3 (by Brett Lawson)

  - Clarified connection string as a required parameter for Connect
  - Clarified Disconnect method options block contents.
  - Adjusted MutationState input parameters
  - Clarified that Bucket/Scope/Collection/DefaultCollection never throw.

- Nov 18, 2019 - Revision #4 (by Brett Lawson & Michael Nitchsinger)

  - Added missing changelog meta-data
  - Removed explicit Cluster constructor in favor of Cluster.Connect methods.
  - Authenticator information now moved to bootstrap RFC.
  - Diagnostics methods removed and moved to Diagnostics RFC.
  - Configuration options updated to be more consistent.
  - Added connection string wording for all configuration options.
  - Explicitly indicated that MutationState should not support empty construction.
  - Added NumKvConnections, MaxHttpConnections and IdleHttpConnectionTimeout configuration options to table.

- Dec 17, 2019 - Revision #5 (by Brett Lawson)

  - Updated IdleHttpConnectionTimeout to 30s to avoid triggering SDK warnings when FTS forcefully disconnects the SDK.  Also added a note that there should be a warning produced if a time over 55s is specified.

- Feb 6, 2020 - Revision #6 (by Brett Lawson)

  - Updated the KVDurableTimeout description to be slightly more clear about intent.
  - Added description for why the connection options table has two names per entry.

- April 30, 2020 (by Brett Lawson)

  - Moved RFC to ACCEPTED state.

- September 20, 2020 (by Brett Lawson)

  - Converted to Markdown.
  - Reorganized Configuration section to be more readable.

# Signoff

| Language   | Team Member         | Signoff Date | Revision |
| ---------- | ------------------- | ------------ | -------- |
| Node.js    | Brett Lawson        | 2020-04-16   | 6        |
| Go         | Charles Dixon       | 2020-04-22   | 6        |
| Connectors | David Nault         | 2020-04-29   | 6        |
| PHP        | Sergey Avseyev      | 2020-04-22   | 6        |
| Python     | Ellis Breen         | 2020-04-29   | 6        |
| Scala      | Graham Pople        | 2020-04-30   | 6        |
| .NET       | Jeffry Morris       | 2020-04-23   | 6        |
| Java       | Michael Nitschinger | 2020-04-16   | 6        |
| C          | Sergey Avseyev      | 2020-04-22   | 6        |
| Ruby       | Sergey Avseyev      | 2020-04-22   | 6        |

