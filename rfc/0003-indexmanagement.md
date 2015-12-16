# Meta

 - RFC Name: Index Management
 - RFC ID: 0003-indexmanagement
 - Start Date: 2015-12-14
 - Owner: Simon Baslé (@simonbasle)
 - Current Status: Draft

# Summary
The aim of the "`Index Management`" RFC is to offer a simplified API in the `BucketManager` to list, create and drop indexes, for the majority of use cases. In this sense, it is focused on GSI indexes and assumes a few other givens. For cases that don't fit in this picture, the fallback for the user is to craft appropriate `N1QL` queries (eg. in Java using the `Index` fluent API).

# Motivation
The goal is to abstract away regularly needed queries relative to indexes and provide a simple API. This way the user can focus on his business related `N1QL` queries, without having to think twice about the boilerplate of creating indexes.

# General Design
The API will be added to the `BucketManager` class of each SDK. In its blocking form, the API is made up of at least **6** base methods:
 - `listIndex`: lists all the indexes relating to the current `Bucket` using the `system:indexes` N1QL view.
 - `createPrimaryIndex`: create a primary index on the current Bucket.
 - `createIndex`: create a secondary GSI index on the current Bucket.
 - `dropPrimaryIndex`: drop the current Bucket's primary index.
 - `dropIndex`: drop a specific secondary index of the current Bucket.
 - `buildDeferredIndexes`: trigger the build of indexes that were created in a deferred fashion (see below).

Additionally it is open for discussion wether or not this RFC should also include the following ***2*** methods in a cross-SDK fashion, or if they should be optional:
 - `watchIndex`: poll the `system:indexes` N1QL view until a given index changes its status to "online" (or a maximum amount of time has passed).
 - `watchIndexes`: poll the `system:indexes` N1QL view until a given list of index all become "online" (or a maximum amount of time has passed).


The underlying design would be to rely on existing `N1QL` verbs and, for the listing and polling part, to use the `system:indexes` keyspace. Each method will internally create and execute a specific N1QL query according to its parameter and the name of the current Bucket.

Each method will offer a bit of tuning with a couple of parameters, so let's detail a more complete signature for each one.

## Listing Indexes
Signature:
```java
List<IndexInfo> listIndexes()
```

Remarks:
The method relies on `system:indexes` with the following N1QL statement:
```sql
SELECT idx.* FROM system:indexes AS idx WHERE keyspace_id = `BUCKET_NAME_INJECTED_HERE` ORDER BY is_primary DESC, name ASC;
```

For SDK in language with no dynamic typing, each row will be converted to a `IndexInfo` object with the following attributes:

```java
String name
boolean isPrimary
IndexType type //enum type GSI/VIEW
String rawType //raw string type in case a new index type is introduced, for forward-compatibility
String state //eg. "online", "pending"
String keyspace
String namespace
JsonArray indexKey //or equivalent Json representation
```

## Creating Indexes
There are two specificities for index creation (both of primary and secondary indexes): the index could already exist and there is the possibility of deferring the build.

In the first specificity, it makes sense to offer two alternatives: either this is considered an error (as N1QL does) or the creation should be purely and simply ignored.
This choice can be made through a first `boolean` parameter, `ignoreIfExist`. If it is `false`, then a SDK-related exception should be raised (eg. in Java a `CouchbaseException`).

The second specificity can be triggered by another `boolean` parameter, `defer`. If set to true, the N1QL query will use the "with defer" syntax and the index will simply be "pending".

Signatures:
```java
boolean createPrimaryIndex(boolean ignoreIfExist, boolean defer)

boolean createIndex(String indexName, List<String> fields, boolean ignoreIfExist, boolean defer)
```

## Dropping Indexes
For index removal, the inverse of `ignoreIfExist` must be put in place: N1QL considers it an error to attempt dropping an index that doesn't exist. So the `ignoreIfNotExist` will drive the behavior of the API in such a case: either `false` and a `CouchbaseException` (or equivalent) will be thrown, or `true` and the invocation will do nothing.

Signatures:
```java
boolean dropPrimaryIndex(boolean ignoreIfNotExist)

boolean dropIndex(String name, boolean ignoreIfNotExist)
```

> **Note**: Unlike index creation, defer the drop of an index is not possible

## Building deferred indexes
Signature:
```java
List<String> buildDeferredIndexes()
```

Remarks:
The returned list is the list of indexes that were in **`pending`** state at the time of the call.

The method will first list indexes, only keeping the `pending` ones. It then issues the appropriate N1QL statement from the list of pending indexes (here `INDEX_NAME1` and `INDEX_NAME3`):

```sql
BUILD INDEX ON `BUCKET_NAME_INJECTED_HERE`(`INDEX_NAME1`, `INDEX_NAME3`)
```

# Watching Indexes
>**Note**: this part of the API could be considered optional.

Since the process of building a deferred index is asynchronous, the server immediately returns control to the client. It would be useful to have an API to poll the list of indexes until one (or several) indexes are seen as "online". In the blocking API, this would allow to make sure that an index is ready.

Signature:
```java
boolean watchIndex(String indexName, long watchTimeout, TimeUnit watchTimeUnit)

List<IndexInfo> watchIndexes(List<String> watchList, boolean watchPrimary, long watchTimeout, TimeUnit watchTimeUnit)
```

Remarks:
The implementation would be based on regularly polling the index view (using `listIndexes()` for instance) until a specific index (or a list of indexes) is seen as `online`.

`boolean watchPrimary`, if set to `true`, would have the effect of adding the default primary name **`#primary`** to the watchList. It is recommended to have that name as a constant somewhere and reference it in the method inline documentation (for instance in Java this is in `Index.PRIMARY_NAME` in the dsl).

The `watchTimeout` and `watchTimeUnit` are a way to express a maximum **duration** for the poll. This can be replaced by a language-idiomatic way of representing a duration in each SDK.

An index that doesn't exist is ignored, and in such a case `watchIndex` would return `false`.

`watchIndexes` returns the `List<IndexInfo>` of indexes that
 1. were already online or...
 2. became online during the watch timeframe.
Thus it can be empty if indexes were all "pending" and failed to be built during the watch timeframe, or if no index in the provided list exists.

# Language Specifics
## Java
 - For secondary index creation, since the Java SDK has an `Expression` type, it could accept `List<Object>` as a `fields` parameter. Each element must then either be an `Expression` or a `String` representing the field to be indexed.

 - For convenience, `createIndex` will offer a signature with varargs syntax (`boolean createIndex(String indexName, boolean ignoreIfExist, boolean defer, Object... fields)`).

 - For `watchIndexes`, the polling should be done with a linearly augmenting delay, from 50ms to 1000ms by steps of 500ms. Only the watch timeout should limit the polling (no maximum number of attempts)

 - The asynchronous version of `watchIndex` should return `Observable<IndexInfo>`, emitting the IndexInfo of the index once it becomes online.

 - Similarly, the async version of `watchIndexes` would return an `Observable<IndexInfo>` with a single emission for each index **that exists** and becomes "online".

## Unresolved SDK specifics
 * .NET
 * NodeJS
 * Go
 * C
 * PHP
 * Python
 * Ruby

## SDK without specifics (signed off)

# Unresolved Questions
 * Can the `watchIndex`/`watchIndexes` feature be implemented easily in each SDK and should it be part of the mandatory implementation of this RFC?

# Final signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |