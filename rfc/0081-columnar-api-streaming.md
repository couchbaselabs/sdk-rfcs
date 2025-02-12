# Meta

* RFC Name: Columnar API Streaming
* RFC ID: 81
* Start Date: 2024-04-23
* Owners: Dimitris Christodoulou, Matt Wozakowski
* Current Status: REVIEW
* Revision: \#4
* Relates to:
    * [SDK-RFC#79 (Columnar API Foundation)][sdk-rfc-0079]
    * [SDK-RFC#82 (Columnar API Error Handling and Retries)][sdk-rfc-0082]                

# Motivation

Query operations in the Columnar SDKs can return a large number of results. We should expose them to the user via streaming, to avoid buffering a large amount of data in the SDK.

# API Concepts

The [Columnar API Foundation RFC][sdk-rfc-0079] describes two types of objects, *active* and *metadata* objects. The former can perform network operations, while the latter cannot and are the objects returned from a method call on an active object.

To be able to return a streaming result, we additionally need to introduce the concept of an *iterator* result, that can perform network operations while also being the result of a method call on an active object.

## Iterator Objects

Iterator objects are objects that can be used to perform further network operations in order to yield a collection of metadata objects. An Iterator Object can be accessed via an operation on an active object. Iterator objects provide an accessor for the streaming iterator itself (or expose an equivalent interface to iterate in an idiomatic way), which performs network operations, as well as accessors for other metadata objects that might be set during or after the streaming has completed.

Accessing iterator objects may or may not return instantly and may or may not perform a network call depending on what is idiomatic to each SDK. However, a consistent approach should be followed throughout each SDK. The preferred approach is to make some initial network calls to read the first result(s) and return any errors immediately ([Error Handling](#error-handling)). As mentioned, SDKs can deviate from this if considered appropriate.

Accessing an iterator object from the active cluster object:

```
QueryResult result = cluster.executeQuery(“SELECT 1=1”)
```

This operation can raise a pre-stream error if the SDK performs any initial network operations at this stage.

Accessing the iterator itself:

```
Stream<JsonObject> rows = result.rowsAs<JsonObject>()
```

This should not raise any errors.

Iterating through the collection of metadata objects provided by the iterator can either be done through accessors, through a range-based for loop, or both, depending on what is possible or idiomatic in each SDK/language. SDKs should seek to expose a similar interface for iterating for all iterator objects they provide. Errors can occur while iterating, and should be exposed in an idiomatic way (either raised or returned).

# Query API

This section will describe in more detail the API for `QueryResult`, in addition to what is outlined in [Columnar API Foundation RFC][sdk-rfc-0079].

The Query API should satisfy the following requirements:

* Provide access to rows without buffering them in the QueryResult instance.  
* Rows can be deserialized when being accessed, in a way that is idiomatic to the language.  
* Provide access to the metadata, only when the entire analytics query response has been read, otherwise raise an error where that is applicable, i.e. where it is possible to call the accessor before the metadata has been populated.  
* Ensure that the user can control the rate at which rows are read.  
* If no alternative way of cancellation is provided (e.g. cancellation tokens), it should provide a way to cancel the operation at any point during or before iteration.


With these guidelines in mind, we propose two API approaches for row streaming: pull-based, and push-based. SDKs should implement at least one of these two approaches. SDKs should aim to follow the proposed APIs unless there is good reason not to (i.e. it is not possible or idiomatic in the SDK/language).

Optionally, a buffered API can also be provided, as long as a streaming alternative is also available.

## Pull-based row streaming

### Method signature

```
(QueryResult, error)    executeQuery(String statement, [QueryOptions options])
```

SDKs can implement this method either via an SDK-sync or an SDK-async API, or both. `QueryResult` is an [Iterator object](#iterator-objects) that may perform network operations to fetch the query result rows.

### QueryResult

The `QueryResult` API can take one of the following forms, depending on what is idiomatic for the language or the SDK.

**Static row typing:**
```
type QueryResult {
    Stream<T>               rowsAs<T>()
    (QueryMetadata, error)  metadata()
    void                    cancel()    // Optional, if idiomatic
}
```

**Static individual row typing:**
```
type QueryResult {
    Stream<Row>             rows()
    (QueryMetadata, error)  metadata()
    void                    cancel()    // Optional, if idiomatic
}
    
type Row {
    T   contentAs<T>()
}
```

**Dynamic row typing (i.e. the deserializer determines the type, as well as the value):**
```
type QueryResult {
    Stream<Any>             rows()
    (QueryMetadata, error)  metadata()
    void                    cancel()    // Optional, if idiomatic
}
```

Cancellation can be either exposed via the `cancel()` operation depicted here, or in any other way that is idiomatic to the language/SDK.

In languages where there is no built-in concept of Stream, a custom iterator can be provided that provides accessors to fetch the next row. Where applicable, we should ensure that the custom iterator can be iterated using idiomatic syntax in the language. In that case, a user should be able to use the custom iterator with a range-based for loop. For example:

```
for (T row : res.rowsAs<T>()) {
    ...
}
```

In languages with dynamic row typing, if there is a need for conversion to a specific user-defined type, this can be achieved by the use of special constructors or factory methods after the row has been deserialized. For example:

```
for (Any row : res.rows()) {
    user = User.fromJson(row)
    ... 
}
```

## Push-based row streaming

### Method signature

```
(QueryMetadata, error)   executeQuery(String statement, RowHandler handler, [QueryOptions options])
```

where `RowHandler` is a function of type `Row -> error`.

SDKs can implement this method either via an SDK-sync or an SDK-async API, or both.

Rows should be processed one at a time on a single thread, calling the handler requires the handler for the previous row to have returned. This ensures that if processing the rows is slow, the rate at which rows are pushed is adjusted accordingly.

Note that cancellation is platform-idiomatic, and should be possible at any point of the operation, including before the first row is read. Additionally, if an error is raised or returned from the row handler, that should trigger cancellation and the error is propagated to the operation-level error.

### Row

**Static row typing:**
```
type Row {
    T   contentAs<T>()
}
```

**Dynamic row typing:**
```
type Row {
    Any content()
}
```

## Buffered API

The API for the buffered version of `executeQuery` is almost identical to the API for pull-based row streaming, with the difference being that `rows()` or `rowsAs<T>` return a `List`, or equivalent, instead of a `Stream`.

## Metadata

### QueryMetadata

```
type QueryMetadata {
    String              requestId()     // “requestID”
    List<QueryWarning>  warnings()      // “warnings”
    QueryMetrics        metrics()       // “metrics”
}
```

### QueryMetrics

```
type QueryMetrics {
    Duration    elapsedTime()           // “elapsedTime”  
    Duration    executionTime()         // “executionTime”
    Uint64      resultCount()           // “resultCount”
    Uint64      resultSize()            // “resultSize”
    Uint64      processedObjects()      // “processedObjects”
}
```

### QueryWarning

```
type QueryWarning {
    Int32   code()      // “code”
    String  message()   // “msg”
}
```

## Deserialization

Rows are deserialized according to the custom deserializer specified in the `QueryOptions` (or the default deserializer) in order to convert the encoded rows into the type `T` specified in the `RowsAs\<T\>()` call, or in the case of dynamic row typing, the type that is returned by the deserializer in the `rows()` call.

A `JsonDeserializer` should be included, and used as the default deserializer. It should be taken into consideration that Capella Columnar may not necessarily be returning JSON in the future, so it should be possible to accommodate that by using a different implementation of the `Deserializer` interface. For languages with dynamic row typing (where `RowsAs<Byte[]>` is not possible) a `PassthroughDeserializer` that simply emits the raw bytes of the row, should also be provided.

Any valid deserializers must implement the following interface:

**Static row typing:**
```
interface Deserializer {
    (T, error)      Deserialize<T>(Byte[] encoded)
}
```
_Note:_ If `T == Byte[]` the deserializer simply passes through the encoded value.

**Dynamic row typing:**
```
interface Deserializer {
    (Any, error)    Deserialize(Byte[] encoded)
}
```

# Response Parsing

The response body is a JSON object that consists of a set of metadata fields, as well as a JSON array containing the rows in the “results” field. The SDK reads the response up to the first row before returning and handles any errors, as described in [Error Handling](#error-handling). The rows are then streamed back to the user. After all rows have been streamed, the rest of the body is read in order to populate all fields of `QueryMetadata`.

Metadata fields can be received in the response either at the beginning or end of the HTTP response from the server. The SDK needs to store the metadata fields internally while reading the streamed response, while not exposing the incomplete metadata to the user. The `metadata()` accessor should raise or return a platform-idiomatic “illegal state” exception (see [Columnar Error Handling and Retries RFC][sdk-rfc-0082]) if a user attempts to read the metadata before all fields have been read from the network.

**Example response body**

```javascript
{
"requestID": "94c7f89f-92b6-4aba-a90d-be715ca47309",
"signature": {
    "*": "*"
},
"results": [
    {
        "airline": {
            "id": 10,
            "type": "airline",
            "name": "40-Mile Air",
            "iata": "Q5",
            "icao": "MLA",
            "callsign": "MILE-AIR",
            "country": "United States"
        }
    },
    {
        "airline": {
            "id": 10123,
            "type": "airline",
            "name": "Texas Wings",
            "iata": "TQ",
            "icao": "TXW",
            "callsign": "TXW",
            "country": "United States"
        }
    }
],
"plans": {},
"status": "success",
"metrics": {
    "elapsedTime": "14.927542ms",
    "executionTime": "12.875792ms",
    "compileTime": "4.178042ms",
    "queueWaitTime": "0ns",
    "resultCount": 2,
    "resultSize": 300,
    "processedObjects": 2,
    "bufferCacheHitRatio": "100.00%"
}
}
```

# Error Handling

Errors can occur at multiple points during the operation

* Prior to sending the server request, during input validation (e.g. Invalid argument)  
* At the initial server response (e.g. Non-existent scope/collection, parsing failure etc.).  
* Mid-stream (e.g. Network issue, timeout)

Most common errors fall in the first two categories. The SDK should wait to get an initial server response from the server before returning from the operation call, to ensure that it raises any initial errors at the operation call. It should attempt to read the first row from the stream, and if it is empty, or the “results” field does not exist in the response body, the entire body of the response should be read, including the errors field, if present. Either the appropriate error should be raised, or the request should be retried according to the [Columnar Error Handling and Retries][sdk-rfc-0082] RFC.  If the SDK will not raise exceptions at the point of the initial operation, reading the first row can be omitted.

It should be noted that the response can include errors, even if the HTTP status is 200 OK.

If any error is encountered mid-stream (i.e. the results are followed by an “errors” field), the SDK should fast-fail and raise or return the error.

**Example errored response body:**

```javascript
{
  "requestID": "c794ae3c-5436-4b56-9cbb-dd6d396fa0e8",
  "errors": [
    {
      "code": 24045,
      "msg": "Cannot find analytics collection nonexistent in analytics scope Default nor an alias with name nonexistent (in line 1, at column 15)"
    }
  ],
  "status": "fatal",
  "metrics": {
    "elapsedTime": "10.033833ms",
    "executionTime": "8.926333ms",
    "compileTime": "0ns",
    "queueWaitTime": "0ns",
    "resultCount": 0,
    "resultSize": 0,
    "processedObjects": 0,
    "bufferCacheHitRatio": "0.00%",
    "errorCount": 1
  }
}
```

# Flow Control

HTTP/1.1 connections rely on the transport layer (i.e. TCP) for flow control which uses a sliding window protocol. The server’s HTTP response is sent back to the client across multiple TCP packets. The client informs the server on its window size when it acknowledges the receipt of each individual packet, so the server does not overwhelm the client.

SDKs typically use a high-level HTTP library that exposes the HTTP response body via a streaming interface. The SDK should not buffer the entire response. Instead, it should buffer a certain number of rows or bytes, that is high enough to be performant but low enough not to consume excessive memory.

As long as these guidelines are being followed, in the case of slow consumers, if parts of the body have not been read, the HTTP library will not acknowledge any further incoming TCP packets and any appropriate backpressure will be handled at the TCP layer.

# Cancellation

The stream should be able to be canceled at any point before all rows have been iterated. The way this is done is dependent on the SDK and API. For example, in SDKs which will return an iterator, this can be done via its `cancel()` method, or, if idiomatic, through the use of cancellation tokens, cancellable promises, etc.

Upon cancellation, the SDK should close the connection. Cancellation is “fire-and-forget”, it does not raise or return any errors and should return instantly.

# Timeout

The timeout should apply to the whole stream. This is similar to the approach in gRPC where the deadline applies to the whole stream, and also ensures that there are no TCP connections lingering indefinitely if, for example, a user does not iterate through the rows and does not cancel the operation.

# Changelog

* 2024-06-05 \- Revision \#1 (Dimitris Christodoulou, Matt Wozakowski)  
* Initial draft  
* 2024-06-06 \- Revision \#2 (Dimitris Christodoulou, Matt Wozakowski)  
* Added optional Buffered API for `executeQuery`  
* 2024-07-31 \- Revision \#3 (Dimitris Christodoulou, Matt Wozakowski)  
* Removed option to raise a `ColumnarException` when metadata is not yet available, as that exception is now reserved for errors resulting from a failed client-server interaction  
* 2024-08-20 \- Revision \#4 (Dimitris Christodoulou, Matt Wozakowski)  
* Renamed `QueryResultRow` to `Row` for SDKs implementing individual row typing

# Signoff

| Language | Team Member | Signoff Date | Revision |
| :---- | :---- | :---- | :---- |
| Node.js | Jared Casey | 2025-02-03 | 4 |
| Go | Charles Dixon |  |  |
| PHP | Sergey Avseyev |  |  |
| Python | Jared Casey |  2025-02-03 | 4 |
| Scala | Graham Pople | 2024-08-22 | 4 |
| .NET | Jeffry Morris |  |  |
| Java | David Nault | 2025-01-22 | 4 |
| C++ | Sergey Avseyev |  |  |
| Ruby | Sergey Avseyev |  |  |
| Kotlin | David Nault | 2025-01-22 | 4 |

[sdk-rfc-0079]: /rfc/0079-columnar-api-foundation.md
[sdk-rfc-0082]: /rfc/0082-columnar-api-error-handling-retries.md
