# Meta

 - RFC Name: Index Management
 - RFC ID: 0003-indexmanagement
 - Start Date: 2015-12-14
 - Owner: Simon Baslé (@simonbasle)
 - Current Status: REVIEW

# Summary
The aim of **RFC3**, **"`Index Management`"**, is to offer a simplified API in the `BucketManager` to list, create and drop indexes, for the majority of use cases. In this sense, it is focused on GSI indexes and assumes a few other givens. For cases that don't fit in this picture, the fallback for the user is to craft appropriate `N1QL` queries (eg. in Java using the `Index` fluent API).

# Motivation and scope
The goal is to abstract away regularly needed queries relative to indexes and provide a simple API. This way the user can focus on his business related `N1QL` queries, without having to think twice about the boilerplate of creating indexes.

However, the scope is not comprehensive coverage of all index creation possibilities in N1QL. Most complex use cases would still need careful crafting of `N1QL` queries from the user instead of relying on this API. Notably:

 - no support for index types other than GSI.
 - no support for `WHERE` clauses in secondary indexes (only the list of fields to index).

In case new index-related semantics are added in a version of Couchbase Server 4.0 (from which this RFC is designed), or if support for one of the "out-of-scope" features is heavily requested by users, a new RFC will be needed to augment this RFC.

# General Design
The API will be added to the `BucketManager` class of each SDK. In its blocking form, the API is made up of at least **8** base methods:

 - `listIndexes`: lists all the indexes relating to the current `Bucket` using the `system:indexes` N1QL view.
 - `createPrimaryIndex`: create a primary index on the current Bucket. If a name is provided through an overload, creates a custom-named primary index on the current Bucket.
 - `createIndex`: create a secondary GSI index on the current Bucket.
 - `dropPrimaryIndex`: drop the current Bucket's primary index. If a name is given in an overload, drop a named primary index from the current Bucket.
 - `dropIndex`: drop a specific secondary index of the current Bucket.
 - `buildDeferredIndexes`: trigger the build of indexes that were created in a deferred fashion (see below).

Additionally, the following method is considered optional but would provide good value to customers if it is implemented:

  - `watchIndexes`: poll the `system:indexes` N1QL view until a given list of index all become "online" (or a maximum amount of time has passed).

Implementation of `watchIndexes` should be idiomatic to each SDK. Note that an asynchronous version of it makes a lot of sense, but one can chose to implement it in a synchronous blocking fashion (or both) as well.


The underlying global design would be to rely on existing `N1QL` verbs and, for the listing and polling part, to use the `system:indexes` keyspace. Each method will internally create and execute a specific N1QL query according to its parameter and the name of the current Bucket.

Each method will offer a bit of tuning with a couple of parameters, so let's detail a more complete signature for each one (syntax is pseudo-code closer to Java).

## Listing Indexes
Signature:
```java
List<IndexInfo> listIndexes()
```

Remarks:
The method relies on `system:indexes` with the following N1QL statement:
```sql
SELECT idx.* FROM system:indexes AS idx WHERE keyspace_id = `BUCKET_NAME_INJECTED_HERE` AND `using` = "gsi" ORDER BY is_primary DESC, name ASC;
```

For SDK in language with no dynamic typing, each row will be converted to a `IndexInfo` object with the following attributes:

```java
String name
boolean isPrimary
IndexType type //enum type GSI/VIEW
String rawType //raw string type in case a new index type is introduced, for forward-compatibility
String state //eg. "online", "pending" or "deferred"/"building"
String keyspace
String namespace
JsonArray indexKey //or equivalent Json representation
```

## Creating Indexes
There are two specificities for index creation (both of primary and secondary indexes): the index could already exist and there is the possibility of deferring the build.

In the first specificity, it makes sense to offer two alternatives: either this is considered an error (as N1QL does) or the creation should be purely and simply ignored.
This choice can be made through a first `boolean` parameter, `ignoreIfExist`. If it is `false`, then a SDK-related exception, `IndexAlreadyExistsException`, should be raised for SDKs that rely on exceptions.

The second specificity can be triggered by another `boolean` parameter, `defer`. If set to true, the N1QL query will use the "with defer" syntax and the index will simply be "pending" (prior to 4.5) or "deferred" (at and after 4.5, see [MB-14679](https://issues.couchbase.com/browse/MB-14679)).

Signatures:

```java
boolean createPrimaryIndex(boolean ignoreIfExist, boolean defer)

boolean createPrimaryIndex(String customName, boolean ignoreIfExist, boolean defer)

boolean createIndex(String indexName, List<String> fields, boolean ignoreIfExist, boolean defer)
```

N1QL statements examples:

```sql
CREATE PRIMARY INDEX ON `bucketName` WITH {"defer_build": true};

CREATE PRIMARY INDEX `indexName` ON `bucketName` WITH {"defer_build": true};

CREATE INDEX `indexName` ON `bucketName` (`field1`, `field2`) USING GSI WITH {"defer_build": true};
```

Note that the default is `USING GSI` but it can be explicitly added to the statements.

For languages that permit it, the list of fields in `createIndex` could also accept `Expression` objects (or any relevant equivalent). Rather than interpreting the entry as a field name to be escaped, the entry could then be used "as is", allowing for more complex constructions (like an index defined as "`DISTINCT ARRAY v FOR v IN schedule END`" for instance).

## Dropping Indexes
For index removal, the inverse of `ignoreIfExist` must be put in place: N1QL considers it an error to attempt dropping an index that doesn't exist. So the `ignoreIfNotExist` will drive the behavior of the API in such a case: either `false` and an `IndexDoesNotExistException` should be thrown (or equivalent for SDKs that don't rely on exceptions), or `true` and the invocation will do nothing.

Signatures:

```java
boolean dropPrimaryIndex(boolean ignoreIfNotExist)

boolean dropPrimaryIndex(String customName, boolean ignoreIfNotExist)

boolean dropIndex(String name, boolean ignoreIfNotExist)
```

N1QL statements examples:

```sql
DROP PRIMARY INDEX ON `bucketName` USING GSI;

DROP INDEX `bucketName`.`primaryIndexName` USING GSI;

DROP INDEX `bucketName`.`indexName` USING GSI;
```

> **Note**: Unlike index creation, deferring the drop of an index is not possible

## Building deferred indexes
Signature:

```java
List<String> buildDeferredIndexes()
```

Remarks:
The returned list is the list of indexes that were in **`pending`** or **`deferred`** state at the time of the call.

The method will first list indexes, only keeping the `pending`/`deferred` ones. It then issues the appropriate N1QL statement from the list of pending indexes (here `INDEX_NAME1` and `INDEX_NAME3`):

```sql
BUILD INDEX ON `BUCKET_NAME_INJECTED_HERE`(`INDEX_NAME1`, `INDEX_NAME3`) USING GSI;
```

# Watching Indexes
>**Note**: this part of the API could be considered optional.

Since the process of building a deferred index is asynchronous, the server immediately returns control to the client. It would be useful to have an API to poll the list of indexes until one (or several) indexes are seen as "online". In the blocking API, this would allow to make sure that an index is ready.

Signature:

```java
List<IndexInfo> watchIndexes(List<String> watchList, boolean watchPrimary, long watchTimeout, TimeUnit watchTimeUnit)
```

Remarks:
The implementation would be based on regularly polling the index view (using `listIndexes()` for instance) until a specific index (or a list of indexes) is seen as `online`.

`boolean watchPrimary`, if set to `true`, would have the effect of adding the default primary name **`#primary`** to the watchList. It is recommended to have that name as a constant somewhere and reference it in the method inline documentation (for instance in Java this is in `Index.PRIMARY_NAME` in the dsl).

The `watchTimeout` and `watchTimeUnit` are a way to express a maximum **duration** for the poll. This can be replaced by a language-idiomatic way of representing a duration in each SDK.

An index that doesn't exist is ignored. `watchIndexes` returns the `List<IndexInfo>` of indexes that

 1. were already online or...
 2. became online during the watch timeframe.

Thus it can be empty if indexes were all "pending"/"deferred"/"building" and failed to be built during the watch timeframe, or if no index in the provided list exists.

If this is easy to implement, `watchIndexes` should rely on a polling with a linearly augmenting delay (50ms to 1000ms by steps of 500ms is deemed a good retry pattern). In such a case, each poll cycle may rely on a `IndexesNotReadyException` internally to signal that current cycle didn't observe all requested indexes.

For SDKs with an idiomatic asynchronous API (eg. Java with RxJava), an asynchronous version of the polling should be implemented, allowing to avoid blocking the rest of the code if an index takes a long time to become online.

For SDKs with configurable logging semantics, a mean of toggling a DEBUG log of the different polling attempts may be offered.

# Language Quick Example and Specifics
## Java

```java
//create a primary index, if there is none
bucketManager.createPrimaryIndex(true, false);

//make sure "byDescAndToto" will be recreated from scratch by dropping it if it exists
bucketManager.dropIndex("byDescAndToto", true);

//defer creation of the secondary index on field toto and desc.
//note that DESC is a keyword, but the field is escaped when provided as a String.
bucketManager.createIndex("byDescAndToto", false, true, x("toto"), "desc");

//since one of the indexes was built with defer, trigger the build and wait for it to finish in maximum 1 minute
bucketManager.watchIndexes(bucketManager.buildDeferredIndexes(), 1, TimeUnit.MINUTES);

//list the indexes and their state after (max.) 1 minute
System.out.println(bucketManager.listIndexes());
```

*Java Specificities:*

 - For secondary index creation, since the Java SDK has an `Expression` type, it could accept `List<Object>` as a `fields` parameter. Each element must then either be an `Expression` or a `String` representing the field to be indexed. Expressions are not escaped and allow for complex index construction.

 - For convenience, `createIndex` will offer a signature with varargs syntax (`boolean createIndex(String indexName, boolean ignoreIfExist, boolean defer, Object... fields)`).

 - For `watchIndexes`, the polling is done with a linearly augmenting delay, from 50ms to 1000ms by steps of 500ms. Only the watch timeout should limit the polling (no maximum number of attempts)

 - The async version of `watchIndexes` would return an `Observable<IndexInfo>` with a single emission for each index **that exists** and becomes "online". The Observable can be empty if no such event could be observed (or no index exist).

 - `watchIndexes` logs polling attempts in the `indexWatch` logger at `DEBUG` level. Activate this logger to see traces of the polling cycles.

## .NET
    //create a primary index if there is none and default to deferred=false
    var result = bucketManager.CreatePrimaryIndex();

	//create a primary index if there is none and defer = true
	var result = bucketManager.CreatePrimaryIndex(true);

	//drop the index "byDescAndToto" if it exists
	var result = bucketManager.DropIndex("byDescAndToto")

	//defer creation of the secondary index on field toto and desc.
    //note that DESC is a keyword, but the field is escaped when provided as a string.
	var result = bucketManager.CreateIndex("byDescAndToto", true, "toto", "desc");

	//list the indexes
	manager.ListIndexes().ToList().ForEach(Console.WriteLine);

*.NET Specificities:*


- `ignoreIfExist` and `ignoreIfNotExists` are not included since the .NET SDK does not explicitly throw exceptions as a convention (good or bad). The IResult returned by the message does contain the exception that was raised and can be thrown by the calling application if desired.
- For methods which include a `boolean` `defer` parameter, default parameters are used, so if omitted the value for `defer` is `false`.
- `async` versions for all methods are included in the API and behave the same with the exception of require that they be *awaited* and that the return value is `Task<IResult>`.
- WatchIndexes is not implemented as of SDK version 2.2.7 (planned for 2.2.8).
- Overloads for lambda expressions are forthcoming and will require a Type T field to use as for building the expression from the POCO properties: `var result = CreateIndex<Person>("personbyIdAndName_idx", true, x=>x.Id, x=>x.Name);`

*Python*

```python
cb = Bucket()

mgr = cb.bucket_manager()
indexes = mgr.list_indexes()
pprint(tuple(x.raw for x in indexes))

for ix in indexes:
    mgr.drop_index(ix)

mgr.create_primary_index()
mgr.drop_primary_index()
mgr.create_primary_index(defer=True)
to_build = mgr.build_deferred_indexes()
print(to_build)
mgr.watch_indexes(to_build)
```

In Python one can also pass `other_buckets=True` to `list_indexes()`
which will cause it to enumerate other buckets as well, if it can access them.

## Unresolved SDK specifics
 * .NET
 * NodeJS
 * Go
 * C

## SDK without specifics (signed off)

# Unresolved Questions
Should the PHP and Ruby SDKs cover this RFC?

# Final signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | Date (YYYY/MM/DD) |
| -------- | -------------- | ----------------- |
| Java     | Simon Baslé    | 2016/03/31        |
| .NET     | Jeffry Morris  | 2016/03/31 - *explicit* |
| NodeJS   | Brett Lawson   | 2016/03/31 - *implicit*<sup>[1](#iad)</sup> |
| Go       | Brett Lawson   | 2016/03/31 - *implicit*<sup>[1](#iad)</sup> |
| C        | Mark Nunberg   | 2016/03/31 - *implicit*<sup>[1](#iad)</sup> |
| Python   | Mark Nunberg   | 2016/03/31 - *implicit*<sup>[1](#iad)</sup> |


<b id="iad">1</b> *implicit*: Implicitly accepted by lack of signoff by the end of the draft phase deadline.
