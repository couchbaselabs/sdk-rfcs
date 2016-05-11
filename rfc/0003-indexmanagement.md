# Meta

 - RFC Name: Index Management
 - RFC ID: 0003-indexmanagement
 - Start Date: 2015-12-14
 - Owner: Simon Baslé (@simonbasle)
 - Current Status: ACCEPTED

# Summary
The aim of **RFC3**, **"`Index Management`"**, is to offer a simplified API in the `BucketManager` to list, create and
drop indexes, for the majority of use cases. In this sense, it is focused on GSI indexes and assumes a few other
givens. For cases that don't fit in this picture, the fallback for the user is to craft appropriate `N1QL` queries
(eg. in Java using the `Index` fluent API).

# Motivation and scope
The goal is to abstract away regularly needed queries relative to indexes and provide a simple API. This way the user
can focus on his business related `N1QL` queries, without having to think twice about the boilerplate of creating
indexes.

However, the scope is not comprehensive coverage of all index creation possibilities in N1QL. Most complex use cases
would still need careful crafting of `N1QL` queries from the user instead of relying on this API. Notably:

 - no support for index types other than GSI.

In case new index-related semantics are added in a version of Couchbase Server 4.0 (from which this RFC is designed), or
if support for one of the "out-of-scope" features is heavily requested by users, a new RFC will be needed to augment
this RFC.

Provision is made for future types of indexation (eg. FTS) by including the "N1ql" word in each method's name.

# General Design
The API will be added to the `BucketManager` class of each SDK. In its blocking form, the API is made up of at least
**6** base methods:

 - `listN1qlIndexes`: lists all the indexes relating to the current `Bucket` using the `system:indexes` N1QL view.
 - `createN1qlPrimaryIndex`: create a primary index on the current Bucket. If a name is provided through an overload,
   creates a custom-named primary index on the current Bucket.
 - `createN1qlIndex`: create a secondary GSI index on the current Bucket.
 - `dropN1qlPrimaryIndex`: drop the current Bucket's primary index. If a name is given in an overload, drop a named
   primary index from the current Bucket.
 - `dropN1qlIndex`: drop a specific secondary index of the current Bucket.
 - `buildN1qlDeferredIndexes`: trigger the build of indexes that were created in a deferred fashion (see below).

Additionally, the following method is considered optional but would provide good value to customers if it is
implemented:

  - `watchN1qlIndexes`: poll the `system:indexes` N1QL view until a given list of index all become "online" (or a
    maximum amount of time has passed).

Implementation of `watchN1qlIndexes` should be idiomatic to each SDK. Note that an asynchronous version of it makes a
lot of sense, but one can chose to implement it in a synchronous blocking fashion (or both) as well.

The underlying global design would be to rely on existing `N1QL` verbs and, for the listing and polling part, to use the
`system:indexes` keyspace. Each method will internally create and execute a specific N1QL query according to its
parameter and the name of the current Bucket.

Each method will offer a bit of tuning with a couple of parameters, so let's detail a more complete signature for each
one (syntax is pseudo-code closer to Java).

## Listing Indexes
Signature:
```java
List<IndexInfo> listN1qlIndexes()
```

```php
listN1qlIndexes() #-> returns array of associative arrays
```

Remarks:
The method relies on `system:indexes` with the following N1QL statement:
```sql
SELECT idx.* FROM system:indexes AS idx WHERE keyspace_id = `BUCKET_NAME_INJECTED_HERE` AND `using` = "gsi" ORDER BY is_primary DESC, name ASC;
```

For SDK in language with no dynamic typing, where value objects are idiomatic, each row will be converted to a
`IndexInfo` object with the following attributes:

```java
String name
boolean isPrimary
IndexType type //enum type GSI/VIEW
String rawType //raw string type in case a new index type is introduced, for forward-compatibility
String state //eg. "online", "pending" or "deferred"/"building"
String keyspace
String namespace
JsonArray indexKey //or equivalent Json representation
String condition //empty string if the index has no WHERE clause
```

## Creating Indexes
There are two specificities for index creation (both of primary and secondary indexes): the index could already exist
and there is the possibility of deferring the build.

In the first specificity, it makes sense to offer two alternatives: either this is considered an error (as N1QL does) or
the creation should be purely and simply ignored. This choice can be made through a first `boolean` parameter,
`ignoreIfExist`. If it is `false`, then a SDK-related exception, `IndexAlreadyExistsException`, should be raised for
SDKs that rely on exceptions.

The second specificity can be triggered by another `boolean` parameter, `defer`. If set to true, the N1QL query will use
the "with defer" syntax and the index will simply be "pending" (prior to 4.5) or "deferred" (at and after 4.5, see
[MB-14679](https://issues.couchbase.com/browse/MB-14679)).

Signatures:

```java
boolean createN1qlPrimaryIndex(boolean ignoreIfExist, boolean defer)

boolean createN1qlPrimaryIndex(String customName, boolean ignoreIfExist, boolean defer)

boolean createN1qlIndex(String indexName, List<String> fields, Expression whereClause, boolean ignoreIfExist, boolean defer)
```

```php
createN1qlPrimaryIndex(string $customName = '', boolean $ignoreIfExist = false, boolean $defer = true)

createN1qlIndex(string $indexName, array $fields, boolean $ignoreIfExist = false, boolean $defer = true)
```

Note that in the last signature, the `Expression` represents a way to declare the WHERE clause, or condition, of the
index. `null` should be used if not applicable.

N1QL statements examples:

```sql
CREATE PRIMARY INDEX ON `bucketName` WITH {"defer_build": true};

CREATE PRIMARY INDEX `indexName` ON `bucketName` WITH {"defer_build": true};

CREATE INDEX `indexName` ON `bucketName` (`field1`, `field2`) USING GSI WITH {"defer_build": true};

CREATE INDEX `indexName` ON `bucketName` (`field1`, `field2`) WHERE `field1` > 0 USING GSI WITH {"defer_build": true};
```

Note that the default is `USING GSI` but it can be explicitly added to the statements.

For languages that permit it, the list of fields in `createN1qlIndex` could also accept `Expression` objects (or any relevant equivalent). Rather than interpreting the entry as a field name to be escaped, the entry could then be used "as is", allowing for more complex constructions (like an index defined as "`DISTINCT ARRAY v FOR v IN schedule END`" for instance).

## Dropping Indexes
For index removal, the inverse of `ignoreIfExist` must be put in place: N1QL considers it an error to attempt dropping an index that doesn't exist. So the `ignoreIfNotExist` will drive the behavior of the API in such a case: either `false` and an `IndexDoesNotExistException` should be thrown (or equivalent for SDKs that don't rely on exceptions), or `true` and the invocation will do nothing.

Signatures:

```java
boolean dropN1qlPrimaryIndex(boolean ignoreIfNotExist)

boolean dropN1qlPrimaryIndex(String customName, boolean ignoreIfNotExist)

boolean dropN1qlIndex(String name, boolean ignoreIfNotExist)
```

```php
dropN1qlPrimaryIndex(string $customName = '', boolean $ignoreIfNotExist = false)

dropN1qlIndex(string $indexName, boolean $ignoreIfNotExist = false)
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
List<String> buildN1qlDeferredIndexes()
```

```php
buildN1qlDeferredIndexes()
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
List<IndexInfo> watchN1qlIndexes(List<String> watchList, long watchTimeout, TimeUnit watchTimeUnit)
```

Remarks:
The implementation would be based on regularly polling the index view (using `listIndexes()` for instance) until a specific index (or a list of indexes) is seen as `online`.

In order to watch the primary index, the default primary name **`#primary`** must be added to the watchList. It is recommended to have that name as a constant somewhere and reference it in the method inline documentation (for instance in Java this is in `Index.PRIMARY_NAME` in the dsl).

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
bucketManager.createN1qlPrimaryIndex(true, false);

//make sure "byDescAndToto" will be recreated from scratch by dropping it if it exists
bucketManager.dropN1qlIndex("byDescAndToto", true);

//convenience method with a vararg (limitation: no WHERE clause)
//defer creation of the secondary index on field toto and desc.
//note that DESC is a keyword, but the field is escaped when provided as a String.
bucketManager.createN1qlIndex("byDescAndToto", false, true, x("toto"), "desc");

//since one of the indexes was built with defer, trigger the build and wait for it to finish in maximum 1 minute
bucketManager.watchN1qlIndexes(bucketManager.buildDeferredIndexes(), 1, TimeUnit.MINUTES);

//list the indexes and their state after (max.) 1 minute
System.out.println(bucketManager.listN1qlIndexes());
```

*Java Specificities:*

 - For secondary index creation, since the Java SDK has an `Expression` type, it could accept `List<Object>` as a `fields` parameter. Each element must then either be an `Expression` or a `String` representing the field to be indexed. Expressions are not escaped and allow for complex index construction.

 - For convenience, `createN1qlIndex` will offer a signature with varargs syntax (`boolean createIndex(String indexName, boolean ignoreIfExist, boolean defer, Object... fields)`). This has the limitation that the WHERE clause cannot be set with this signature.

 - For `watchN1qlIndexes`, the polling is done with a linearly augmenting delay, from 50ms to 1000ms by steps of 500ms. Only the watch timeout should limit the polling (no maximum number of attempts)

 - The async version of `watchN1qlIndexes` would return an `Observable<IndexInfo>` with a single emission for each index **that exists** and becomes "online". The Observable can be empty if no such event could be observed (or no index exist).

 - `watchN1qlIndexes` logs polling attempts in the `indexWatch` logger at `DEBUG` level. Activate this logger to see traces of the polling cycles.

## .NET
    //create a primary index if there is none and default to deferred=false
    var result = bucketManager.CreateN1qlPrimaryIndex();

	//create a primary index if there is none and defer = true
	var result = bucketManager.CreateN1qlPrimaryIndex(true);

	//drop the index "byDescAndToto" if it exists
	var result = bucketManager.DropN1qlIndex("byDescAndToto")

	//defer creation of the secondary index on field toto and desc.
    //note that DESC is a keyword, but the field is escaped when provided as a string.
	var result = bucketManager.CreateN1qlIndex("byDescAndToto", true, "toto", "desc");

	//list the indexes
	manager.ListN1qlIndexes().ToList().ForEach(Console.WriteLine);

*.NET Specificities:*


- `ignoreIfExist` and `ignoreIfNotExists` are not included since the .NET SDK does not explicitly throw exceptions as a convention (good or bad). The IResult returned by the message does contain the exception that was raised and can be thrown by the calling application if desired.
- For methods which include a `boolean` `defer` parameter, default parameters are used, so if omitted the value for `defer` is `false`.
- `async` versions for all methods are included in the API and behave the same with the exception of require that they be *awaited* and that the return value is `Task<IResult>`.
- `WatchN1qlIndexes` is not implemented as of SDK version 2.2.7 (planned for 2.2.8).
- The whereClause expression is replaced with overloads for lambda expressions. These are forthcoming and will require a Type T field to use as for building the expression from the POCO properties: `var result = CreateN1qlIndex<Person>("personbyIdAndName_idx", true, x=>x.Id, x=>x.Name);`

## Python

```python
cb = Bucket()

mgr = cb.bucket_manager()
indexes = mgr.n1ql_indexes_list()
pprint(tuple(x.raw for x in indexes))

for ix in indexes:
    mgr.n1ql_index_drop(ix)

mgr.n1ql_index_create_primary()
mgr.n1ql_index_drop_primary()
mgr.n1ql_index_create_primary(defer=True)
mgr.n1ql_index_create('ix_fields', fields=['field1', 'field2'])
mgr.n1ql_index_drop('ix_fields')
to_build = mgr.n1ql_index_build_deferred()
print(to_build)
mgr.n1ql_index_watch(to_build)
mgr.n1ql_index_watch(['ix_fields'], watch_primary=True)
```

In Python one can also pass `other_buckets=True` to `list_indexes()`
which will cause it to enumerate other buckets as well, if it can access them.

The method naming is by default `n1ql_index_XXX` because the other bucketmanager
methods are in the form of `designXXX`. There are additional aliases so that
`create_n1ql_index = n1ql_create_inxex`

## Node.js

```javascript
var bucketManager = bucket.manager()

bucketManager.createPrimaryIndex({
  name: '',
  ignoreIfExists: false,
  deferred: false
}, function(err){})

bucketManager.createIndex(NAME, FIELDS, {
  ignoreIfExists: false,
  deferred: false
}, function(err){})

bucketManager.dropIndex(NAME, {
  ignoreIfNotExists: false
}, function(err){})

bucketManager.dropPrimaryIndex({
  name: '',
  ignoreIfNotExists: false
}, function(err){})

bucketManager.getIndexes(function(err, indexes){})

bucketManager.buildDeferredIndexes(function(err, indexes){})

bucketManager.watchIndexes(WATCH_LIST, {
  timeout: 0
}, function(err){})
```

## C

```c
typedef struct {
    /** Raw index definition */
    const char *rawjson;
    size_t nrawjson;
    const char *name;
    size_t nname;
    const char *keyspace;
    size_t nkeyspace;
    /** Output parameter only. State of index */
    const char *state;
    size_t nstate;

    /** Actual index text. For raw JSON use the `index_key` property */
    const char *fields;
    size_t nfields;

    /**
     * Modifiers for the index itself. This might be
     * LCB_IXSPEC_F_PRIMARY if the index is primary. For raw JSON,
     * use `"is_primary":true`
     *
     * For creation the LCB_IXSPEC_F_DEFER option is also accepted to
     * indicate that the building of this index should be deferred.
     */
    unsigned flags;

    /**
     * Type of this index, Can be T_DEFAULT for the default server type, or
     * an explicit T_GSI or T_VIEW.
     * When using JSON, specify `"using":"gsi"`
     */
    unsigned ixtype;
} lcb_N1INDEXSPEC;

typedef struct {
    lcb_N1INDEXSPEC spec;
    lcb_N1IXMGMTCALLBACK callback;
} lcb_CMDN1IXMGMT;

lcb_n1ixmgmt_list(lcb_t instance, const void *cookie, const lcb_CMDN1IXMGMT *cmd);
lcb_n1ixmgmt_mkindex(lcb_t instance, const void *cookie, const lcb_CMDN1IXMGMT *cmd);
lcb_n1ixmgmt_rmindex(lcb_t instance, const void *cookie, const lcb_CMDN1IXMGMT *cmd);
lcb_n1ixmgmt_build_begin(lcb_t instance, const void *cookie, const lcb_CMDN1IXMGMT *cmd);

typedef struct {
    const lcb_N1INDEXSPEC * const *specs;
    size_t nspec;
    lcb_U32 timeout;
    lcb_U32 interval;
    lcb_N1IXMGMTCALLBACK callback;
} lcb_N1CMDIXWATCH;
lcb_n1ixmgmt_build_watch(lcb_t instance, const void *cookie, const lcb_CMDN1IXWATCH *cmd);
```

### C SDK Specifics

C index management operations communicate by the `INDEXSPEC` structure which contains
discreet fields for analysis and also the `rawjson` field which represents how the
index entry is represented in JSON. Wrapping SDKs can simply serialize the parameters
in JSON and then set the `rawjson` field, eliminating the need to set discreet
parameters.

## PHP

```php
$bucketManager = $bucket.manager();

foreach ($bucketManager.listN1qlIndexes() as $index) {
  var_dump($index);
}

$bucketManager.createN1qlPrimaryIndex(true, false);

$bucketManager.dropN1qlIndex("byDescAndToto", true);

$bucketManager.createN1qlIndex("byDescAndToto", array("toto", "desc"), true, false);

$bucketManager.buildDeferredIndexes()
```

## Unresolved SDK specifics
 * NodeJS
 * Go

## SDK without specifics (signed off)

# Unresolved Questions
Should the PHP and Ruby SDKs cover this RFC?

# Final signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | REVIEWED (YYYY/MM/DD) | ACCEPTED Final Signoff |
| -------- | -------------- | --------------------------- | -------- |
| Java     | Simon Baslé    | 2016/03/31                  |          |
| .NET     | Jeffry Morris  | 2016/04/01                  |          |
| NodeJS   | Brett Lawson   | 2016/03/31 - *implicit*<sup>[1](#iad)</sup> |          |
| Go       | Brett Lawson   | 2016/03/31 - *implicit*<sup>[1](#iad)</sup> |          |
| C        | Mark Nunberg   | 2016/03/31 - *implicit*<sup>[1](#iad)</sup> |          |
| Python   | Mark Nunberg   | 2016/03/31 - *implicit*<sup>[1](#iad)</sup> |          |


<b id="iad">1</b> *implicit*: Implicitly accepted by lack of signoff by the end of the draft phase deadline.
