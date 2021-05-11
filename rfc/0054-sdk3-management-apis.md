# Meta

*   RFC Name: SDK3 Management APIs
*   RFC ID: 0054-sdk3-management-apis
*   Start Date: 2019-06-13
*   Owner: Charles Dixon
*   Current Status: ACCEPTED

# Summary

Defines the management APIs for SDK 3.0 that each SDK must implement.

# General Design

The management APIs are made up of several interfaces; Bucket Manager, User Manager, Search Index Manager, View Index Manager, Query Index
Manager, Analytics Index Manager, and Collection Manager.

On the cluster interface:

<big><pre>
[UserManager](#usermanager) Users();
[BucketManager](#bucketmanager) Buckets();
[QueryIndexManager](#queryindexmanager) QueryIndexes();
[AnalyticsIndexManager](#analyticsindexmanager) AnalyticsIndexes();
[SearchIndexManager](#searchindexmanager) SearchIndexes();
</pre></big>

On the bucket interface:

<big><pre>
[CollectionManager](#collectionmanager) Collections();
[ViewIndexManager](#viewindexmanager) ViewIndexes();
</pre></big>

# Service Not Configured

If a manager is requested for a service which is not configured on the cluster then a `ServiceNotConfiguredException` must be raised. This
can be detected by looking in the cluster config object, if the endpoint for the service has no servers listed then it is not configured.

# Feature Not Found

Some features, such as groups, certain functions, and collections are not supported on all server versions. If an endpoint is requested
against a server version that does not support that endpoint then the HTTP response will be a 404 containing the body `"Not Found"`. If this
is returned then the SDK must return a `FeatureNotFoundException` to the user.

# Timeouts

All optional timeout durations default to the client's management API timeout, unless otherwise specified.

# ViewIndexManager

The View Index Manager interface contains the means for managing design documents used for views.

A design document belongs to either the "development" or "production" namespace. A development document has a name that starts with
`dev_`. This is an implementation detail we've chosen to hide from consumers of this API. Document names presented to the user (returned
from the "get" and "get all" methods, for example) always have the `dev_` prefix stripped.

Whenever the user passes a design document name to any method of this API, the user may refer to the document using the `dev_` prefix
regardless of whether the user is referring to a development document or production document. The `dev_` prefix is always stripped from user
input, and the actual document name passed to the server is determined by the "namespace" argument.

All methods (except publish) have a required `namespace` argument indicating whether the operation targets a development document or a
production document. The type of this argument is an enum called [`DesignDocumentNamespace`](#designdocumentnamespace) with values
`PRODUCTION` and `DEVELOPMENT`.

```
interface ViewIndexManager {
    DesignDocument GetDesignDocument(string designDocName, DesignDocumentNamespace namespace, GetDesignDocumentOptions options);

    Iterable<DesignDocument> [GetAllDesignDocuments](#getalldesigndocuments)(DesignDocumentNamespace namespace, GetAllDesignDocumentsOptions options);

    void UpsertDesignDocument(DesignDocument indexData, DesignDocumentNamespace namespace, UpsertDesignDocumentOptions options);

    void DropDesignDocument(string designDocName, DesignDocumentNamespace namespace, DropDesignDocumentOptions options);

    void PublishDesignDocument(string designDocName, PublishDesignDocumentOptions options);
}
```

The following methods must be implemented:

## GetDesignDocument

Fetches a design document from the server if it exists.

### Signature

```
DesignDocument GetDesignDocument(string designDocName, DesignDocumentNamespace namespace, [options])
```

### Parameters

* Required:

  * `designDocName`: `string` - the name of the design document.

  * `namespace`: [`DesignDocumentNamespace`](#designdocumentnamespace) - `PRODUCTION` if the user is requesting a document from the
    production namespace, or `DEVELOPMENT` if from the development namespace.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An instance of [`DesignDocument`](#designdocument).

### Throws

* `DesignDocumentNotFoundException` (HTTP 404)

* Any exceptions raised by the underlying platform

### URI

    GET http://localhost:8092/<bucketname>/_design/<ddocname>

### Example response from server

    {
       "views":{
          "test":{
             "map":"function (doc, meta) {\n\t\t\t\t\t\t  emit(meta.id, null);\n\t\t\t\t\t\t}",
             "reduce":"_count"
          }
       }
    }

## GetAllDesignDocuments

Fetches all design documents from the server.

When processing the server response, the client must strip the `_design/` prefix from the document ID (as well as the `_dev` prefix if
present). For example, a `doc.meta.id` value of `"_design/foo"` must be parsed as `"foo"`, and `"_design/dev_bar"` must be parsed as
`"bar"`.

### Signature

```
Iterable<DesignDocument> GetAllDesignDocuments(DesignDocumentNamespace namespace, [options])
```

### Parameters

* Required:

  * `namespace`: [`DesignDocumentNamespace`](#designdocumentnamespace) - `PRODUCTION` if the user is requesting a document from the
    production namespace, or `DEVELOPMENT` if from the development namespace.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An iterable of [`DesignDocument`](#designdocument).

### Throws

* Any exceptions raised by the underlying platform

### URI

    GET http://localhost:8091/pools/default/buckets/<bucket-name>/ddocs


### Example response from server

    {
       "rows":[
          {
             "doc":{
                "meta":{
                   "id":"_design/dev_test",
                   "rev":"1-ae5e21ec"
                },
                "json":{
                   "views":{
                      "test":{
                         "map":"function (doc, meta) {\n\t\t\t\t\t\t  emit(meta.id, null);\n\t\t\t\t\t\t}",
                         "reduce":"_count"
                      }
                   }
                }
             },
             "controllers":{
                "compact":"/pools/default/buckets/default/ddocs/_design%2Fdev_test/controller/compactView",
                "setUpdateMinChanges":"/pools/default/buckets/default/ddocs/_design%2Fdev_test/controller/setUpdateMinChanges"
             }
          }
       ]
    }

## UpsertDesignDocument

Updates, or inserts, a design document.

### Signature

```
void UpsertDesignDocument(DesignDocument designDocData, DesignDocumentNamespace namespace, [options])
```

### Parameters

* Required:

  * `designDocData`: [DesignDocument](#designdocument) - the data to use to create the design document

  * `namespace`: [`DesignDocumentNamespace`](#designdocumentnamespace) - `PRODUCTION` if the user is requesting a document from the
    production namespace, or `DEVELOPMENT` if from the development namespace.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* Any exceptions raised by the underlying platform

### URI

    PUT http://localhost:8092/<bucketname>/_design/<ddocname>

## DropDesignDocument

Removes a design document.

### Signature

```
void DropDesignDocument(string designDocName, DesignDocumentNamespace namespace,  [options])
```

### Parameters

* Required:

  * `designDocName`: string - the name of the design document.

  * `namespace`: [`DesignDocumentNamespace`](#designdocumentnamespace) - `PRODUCTION` if the user is requesting a document from the
    production namespace, or `DEVELOPMENT` if from the development namespace.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `DesignDocumentNotFoundException` (HTTP 404)

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    DELETE http://localhost:8092/<bucketname>/_design/<ddocname>

## PublishDesignDocument

Publishes a design document. This method is equivalent to getting a document from the development namespace and upserting it to the
production namespace.

### Signature

```
void PublishDesignDocument(string designDocName, [options])
```

### Parameters

* Required:

  * `designDocName`: string - the name of the design document.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `DesignDocumentNotFoundException` (HTTP 404)

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

# QueryIndexManager

The Query Index Manager interface contains the means for managing indexes used for queries.

```
public interface QueryIndexManger {
    Iterable<QueryIndex> GetAllIndexes(string bucketName, GetAllQueryIndexOptions options);

    void CreateIndex(string bucketName, string indexName, []string fields, CreateQueryIndexOptions options);

    void CreatePrimaryIndex(string bucketName, CreatePrimaryQueryIndexOptions options);

    void DropIndex(string bucketName, string indexName, DropQueryIndexOptions options);

    void DropPrimaryIndex(string bucketName, DropPrimaryQueryIndexOptions options);

    void WatchIndexes(string bucketName, []string indexNames, timeout duration, WatchQueryIndexOptions options);

    void BuildDeferredIndexes(string bucketName, BuildQueryIndexOptions options);
}
```

The following methods must be implemented:

## GetAllIndexes

Fetches all indexes from the server.

### Signature

```
Iterable<QueryIndex> GetAllIndexes(string bucketName, [options])
```

### Parameters

* Required:

  * `bucketName`: string - the name of the bucket.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### N1QL

```
SELECT idx.* FROM system:indexes AS idx
WHERE keyspace_id = "bucketName" AND `using`="gsi"
ORDER BY is_primary DESC, name ASC
```

### Returns

An array of [`QueryIndex`](#queryindex).

### Throws

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## CreateIndex

Creates a new index.
[CREATE INDEX](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/createindex.html)

### Signature

```
void CreateIndex(string bucketName, string indexName, []string fields,  [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - the name of the bucket.

  * `indexName`: `string` - the name of the index.

  * `fields`: `[]string` - the fields to create the index over.

* Optional:

  * `IgnoreIfExists` (`bool`) - Don't error/throw if the index already exists. Default to false.

  * `NumReplicas` (`int`) - The number of replicas that this index should have. Uses the WITH keyword and num_replica. Default to omitting
    the parameter from the server request.

        CREATE INDEX indexName ON bucketName WITH { "num_replica": 2 }

  * `Deferred` (`bool`) - Whether the index should be created as a deferred index. Default to omitting the parameter from the server request.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `IndexExistsException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## CreatePrimaryIndex

Creates a new primary index.
[CREATE PRIMARY INDEX](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/createprimaryindex.html)

### Signature

```
void CreatePrimaryIndex(string bucketName, [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - name of the bucket.

* Optional:

  * `indexName`: `string` - name of the index.

  * `IgnoreIfExists` (`bool`) - Don't error/throw if the index already exists. Default to false.

  * `NumReplicas` (`int`) - The number of replicas that this index should have. Uses the WITH keyword and num_replica. Default to omitting
    the parameter from the server request.

        CREATE PRIMARY INDEX indexName ON bucketName WITH { "num_replica": 2 }

  * `Deferred` (`bool`) - Whether the index should be created as a deferred index. Default to omitting the parameter from the server
    request.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `QueryIndexAlreadyExistsException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## DropIndex

Drops an index.
[DROP INDEX](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/dropindex.html)

### Signature

```
void DropIndex(string bucketName, string indexName, [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - name of the bucket.

  * `indexName`: `string` - name of the index.

* Optional:

  * `IgnoreIfNotExists` (`bool`) - Don't error/throw if the index does not exist.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `QueryIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## DropPrimaryIndex

Drops a primary index.
[DROP PRIMARY INDEX](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/dropprimaryindex.html)

### Signature

```
void DropPrimaryIndex(string bucketName, [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - name of the bucket.

* Optional:

  * `IndexName`: `string` - name of the index.

  * `IgnoreIfNotExists` (`bool`) - Don't error/throw if the index does not exist.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `QueryIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## WatchIndexes

Watch polls indexes until they are online.

### Signature

```
void WatchIndexes(string bucketName, []string indexNames, timeout duration, [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - name of the bucket.

  * `indexNames`: `[]string` - name(s) of the index(es).

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

* Optional:

  * `WatchPrimary` (`bool`) - whether or not to watch the primary index.

### Returns

Nothing

### Throws

* `QueryIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## BuildDeferredIndexes

Build Deferred builds all indexes which are currently in deferred state.

### Signature

```
void BuildDeferredIndexes(string bucketName, [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - name of the bucket.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `InvalidArgumentException`

* Any exceptions raised by the underlying platform

# SearchIndexManager

The `SearchIndexManager` interface contains the means for managing indexes used for search. Search index definitions are purposefully left
very dynamic as the index definitions on the server are complex. As such it is left to each SDK to implement its own dynamic type for this,
`JSONObject` is used here to signify that. Stability level is uncommitted?

```
public interface SearchIndexManager{
    SearchIndex GetIndex(string indexName, GetSearchIndexOptions options);

    Iterable<SearchIndex> GetAllIndexes(GetAllSearchIndexesOptions options);

    void UpsertIndex(SearchIndex indexDefinition, UpsertSearchIndexOptions options);

    void DropIndex(string indexName, DropSearchIndexOptions options);

    int GetIndexedDocumentsCount(string indexName, GetIndexedSearchIndexOptions options);

    void PauseIngest(string indexName, PauseIngestSearchIndexOptions);

    void ResumeIngest(string indexName, ResumeIngestSearchIndexOptions);

    void AllowQuerying(string indexName, AllowQueryingSearchIndexOptions);

    void DisallowQuerying(string indexName, DisallowQueryingSearchIndexOptions);

    void FreezePlan(string indexName, FreezePlanSearchIndexOptions);

    void UnfreezePlan(string indexName, UnfreezePlanSearchIndexOptions);

    Iterable<JSONObject> AnalyzeDocument(string indexName, JSONObject document, AnalyzeDocumentOptions options);
}
```

The following methods must be implemented:

## GetIndex

Fetches an index from the server if it exists.

### Signature

```
SearchIndex GetIndex(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An instance of [`SearchIndex`](#searchindex).

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    GET http://localhost:8094/api/index/<name>

## GetAllIndexes

Fetches all indexes from the server.

### Signature

```
Iterable<SearchIndex> GetAllIndexes([options])
```

### Parameters

* Required: None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An array of [`SearchIndex`](#searchindex).

### Throws

* Any exceptions raised by the underlying platform

### URI

    GET http://localhost:8094/api/index

## UpsertIndex

Creates, or updates, an index.

### Signature

```
void UpsertIndex(SearchIndex indexDefinition, [options])
```

### Parameters

* Required:

  * `indexDefinition`: [`SearchIndex`](#searchindex) - the index definition

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `InvalidArgumentsException`

  If any of the following are empty:

  * Name
  * Type
  * SourceType

* Any exceptions raised by the underlying platform

### URI

    PUT http://localhost:8094/api/index/<index_name>

Should be sent with request header "cache-control" set to "no-cache".

## DropIndex

Drops an index.

### Signature

```
void DropIndex(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    DELETE http://localhost:8094/api/index/<index_name>

## GetIndexedDocumentsCount

Retrieves the number of documents that have been indexed for an index.

### Signature

```
uint32 GetIndexedDocumentsCount(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    GET http://localhost:8094/api/index/<index_name>/count

## PauseIngest

Pauses updates and maintenance for an index.

### Signature

```
void PauseIngest(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST /api/index/{indexName}/ingestControl/pause

## ResumeIngest

Resumes updates and maintenance for an index.

### Signature

```
void ResumeIngest(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST /api/index/{indexName}/ingestControl/resume

## AllowQuerying

Allows querying against an index.

### Signature

```
void AllowQuerying(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST /api/index/{indexName}/queryControl/allow

## DisallowQuerying

Disallows querying against an index.

### Signature

```
void DisallowQuerying(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST /api/index/{indexName}/queryControl/disallow

## FreezePlan

Freeze the assignment of index partitions to nodes.

### Signature

```
void FreezePlan(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST /api/index/{indexName}/planFreezeControl/freeze

## UnfreezePlan

Unfreeze the assignment of index partitions to nodes.

### Signature

```
void UnfreezePlan(string indexName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST /api/index/{indexName}/planFreezeControl/unfreeze

## AnalyzeDocument

Allows users to see how a document is analyzed against a specific index.

### Signature

```
Iterable<JSONObject> AnalyzeDocument(string indexName, JSONObject document, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

  * `document`: `JSONObject` - the document to be analyzed.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An iterable of `JSONObject`. The response from the server returns an object containing two top level keys: status and analyzed. The SDK must
return the value containing within the analyze key.

### Throws

* `SearchIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST /api/index/{indexName}/analyzeDoc

# AnalyticsIndexManager

Stability level is not volatile.

```
public interface IAnalyticsIndexManager{
     void CreateDataverse(string dataverseName, CreateDataverseAnalyticsOptions options);

     void DropDataverse(string dataverseName, DropDataverseAnalyticsOptions options);

     void CreateDataset(string datasetName, string bucketName, CreateDatasetAnalyticsOptions options);

     void DropDataset(string datasetName, DropDatasetAnalyticsOptions options);

     Iterable<AnalyticsDataset> GetAllDatasets(GetAllDatasetAnalyticsOptions options);

     void CreateIndex(string indexName, string datasetName, map[string]string fields, CreateIndexAnalyticsOptions options);

     void DropIndex(string indexName, string datasetName, DropIndexAnalyticsOptions options);

     Iterable<AnalyticsIndex> GetAllIndexes(GetAllIndexesAnalyticsOptions options);

     void ConnectLink(ConnectLinkAnalyticsOptions options);

     void DisconnectLink(DisconnectLinkAnalyticsOptions options);

     map[string]map[string]int GetPendingMutations(GetPendingMutationsAnalyticsOptions options);

     void CreateLink(AnalyticsLink link, CreateLinkAnalyticsOptions options);

     void ReplaceLink(AnalyticsLink link, ReplaceLinkAnalyticsOptions options);

     void DropLink(string linkName, string dataverseName, DropLinkAnalyticsOptions options);

     List<AnalyticsLink> GetAllLinks(GetAllLinkAnalyticsOptions options);
}
```

## CreateDataverse

Creates a new dataverse.

### Signature

```
void CreateDataverse(string dataverseName, [options])
```

### Parameters

* Required:

  * `dataverseName`: `string` - name of the dataverse.
    * This name may contain one or more `/`, the `/` must be split on and retokenized before sending.
    * Retokenizing follows the following rule: `bucket.name/scope.name` becomes `` `bucket.name` ``.`` `scope.name` ``.
    * Note that there can be multiple instance of `/` within a name and all must be retokenized.

* Optional:

  * `IgnoreIfExists` (`bool`) - ignore if the dataverse already exists (send `"IF NOT EXISTS"` as part of the dataset creation query,
    e.g. `CREATE DATAVERSE IF NOT EXISTS test ON default`). Default to `false`.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `DataverseAlreadyExistsException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## DropDataverse

Drops a dataverse.

### Signature

```
void DropDataverse(string dataverseName,  [options])
```

### Parameters

* Required:

  * `dataverseName`: `string` - name of the dataverse.
    * This name may contain one or more `/`, the `/` must be split on and retokenized before sending.
    * Retokenizing follows the following rule: `bucket.name/scope.name` becomes `` `bucket.name` ``.`` `scope.name` ``.
    * Note that there can be multiple instance of `/` within a name and all must be retokenized.

* Optional:

  * `IgnoreIfNotExists` (`bool`) - ignore if the dataset doesn't exists (send `"IF EXISTS"` as part of the dataset creation query,
    e.g. `DROP DATAVERSE IF EXISTS test`).

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `DataverseNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## CreateDataset

Creates a new dataset.

### Signature

```
void CreateDataset(string datasetName, string bucketName, [options])
```

### Parameters

* Required:

  * `datasetName`: `string` - name of the dataset.

  * `bucketName`: `string` - name of the bucket.

* Optional:

  * `IgnoreIfExists` (`bool`) - ignore if the dataset already exists (send `"IF NOT EXISTS"` as part of the dataset creation query,
    e.g. `CREATE DATASET IF NOT EXISTS test ON default`). Default to `false`.

  * `Condition` (`string`) - Where clause to use for creating dataset.

  * `DataverseName` (`string`) - The name of the dataverse to use, default to none. If set then will be used as `CREATE DATASET
    dataverseName.datasetName`. If not set then will be `CREATE DATASET datasetName`.
    * This name may contain one or more `/`, the `/` must be split on and retokenized before sending.
    * Retokenizing follows the following rule: `bucket.name/scope.name` becomes `` `bucket.name` ``.`` `scope.name` ``.
    * Note that there can be multiple instance of `/` within a name and all must be retokenized.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.
  
  * `LinkName` (`string`) - The name of the link to use, default to none. If set then is applied in the form of `CREATE DATASET datasetName AT linkName`.

### Returns

Nothing

### Throws

* `DatasetAlreadyExistsException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## DropDataset

Drops a dataset.

### Signature

```
void DropDataset(string datasetName,  [options])
```

### Parameters

* Required:

  * `datasetName`: `string` - name of the dataset.

* Optional:

  * `IgnoreIfNotExists` (`bool`) - ignore if the dataset doesn't exists (send `"IF EXISTS"` as part of the dataset creation query,
    e.g. `DROP DATASET IF EXISTS test`).

  * `DataverseName` (`string`) - The name of the dataverse to use, default to none. If set then will be used as `DROP DATASET
    dataverseName.datasetName`. If not set then will be `DROP DATASET datasetName`.
    * This name may contain one or more `/`, the `/` must be split on and retokenized before sending.
    * Retokenizing follows the following rule: `bucket.name/scope.name` becomes `` `bucket.name` ``.`` `scope.name` ``.
    * Note that there can be multiple instance of `/` within a name and all must be retokenized.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `DatasetNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## GetAllDatasets

Gets all datasets

    SELECT d.* FROM Metadata.`Dataset` d WHERE d.DataverseName <> "Metadata"

### Signature

```
Iterable<AnalyticsDataset> GetAllDatasets([options])
```

### Parameters

* Required: None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Iterable of [`AnalyticsDataset`](#analyticsdataset)

### Throws

* Any exceptions raised by the underlying platform

## CreateIndex

Creates a new index.

### Signature

```
void CreateIndex(string indexName, string datasetName, map[string]string fields,  [options])
```

### Parameters

* Required:

  * `datasetName`: `string` - name of the dataset.

  * `indexName`: `string` - name of the index.

  * `fields`: `map[string]string` - the fields to create the index over. This is a map of the name of the field to the type of the field.

* Optional:

  * `IgnoreIfExists` (`bool`) - don't error/throw if the index already exists. (send `"IF NOT EXISTS"` as part of the dataset creation query,
    e.g. `CREATE INDEX test IF NOT EXISTS ON default`) (Note: `IF NOT EXISTS` is not is in the same place as it is in `CREATE DATASET`)
    Default to `false`.

  * `DataverseName` (`string`) - The name of the dataverse to use, default to none. If set then will be used as `DROP INDEX
    dataverseName.datasetName.indexName`. If not set then will be `DROP INDEX datasetName.indexName`.
    * This name may contain one or more `/`, the `/` must be split on and retokenized before sending.
    * Retokenizing follows the following rule: `bucket.name/scope.name` becomes `` `bucket.name` ``.`` `scope.name` ``.
    * Note that there can be multiple instance of `/` within a name and all must be retokenized.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the
    client.

### Returns

Nothing

### Throws

* `AnalyticsIndexAlreadyExistsException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## DropIndex

Drops an index.

### Signature

```
void DropIndex(string indexName, string datasetName, [options])
```

### Parameters

* Required:

  * `indexName`: `string` - name of the index.

* Optional:

  * `IgnoreIfNotExists` (`bool`) - Don't error/throw if the index does not exist.

  * `DataverseName` (`string`) - The name of the dataverse to use, default to none. If set then will be used as `DROP INDEX
    dataverseName.datasetName.indexName`. If not set then will be `DROP INDEX datasetName.indexName`.
    * This name may contain one or more `/`, the `/` must be split on and retokenized before sending.
    * Retokenizing follows the following rule: `bucket.name/scope.name` becomes `` `bucket.name` ``.`` `scope.name` ``.
    * Note that there can be multiple instance of `/` within a name and all must be retokenized.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `AnalyticsIndexNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## GetAlIndexes

Gets all indexes

    SELECT d.* FROM Metadata.`Index` d WHERE d.DataverseName <> "Metadata".

### Signature

```
Iterable<AnalyticsIndex> GetAllIndexes([options])
```

### Parameters

* Required: None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* Any exceptions raised by the underlying platform

## ConnectLink

Connects a link.

### Signature

```
void ConnectLink([options])
```

### Parameters

* Required: None

* Optional:

  * `DataverseName` (`string`) - name of the dataverse to connect to. Default to `"Default"`.
    * This name may contain one or more `/`, the `/` must be split on and retokenized before sending.
    * Retokenizing follows the following rule: `bucket.name/scope.name` becomes `` `bucket.name` ``.`` `scope.name` ``.
    * Note that there can be multiple instance of `/` within a name and all must be retokenized.

  * `LinkName` (`string`) - name of the link. Default to `"Local"`.

  * `Force` (`boolean`) - whether to force link creation even if the bucket UUID changed, for example due to the bucket being deleted and
    recreated. Default to `false`.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `AnalyticsLinkNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## DisconnectLink

Disconnects a link.

### Signature

```
void DisconnectLink([options])
```

### Parameters

* Required: None

* Optional:

  * `DataverseName` (`string`) - name of the dataverse to connect to. Default to `"Default"`.
    * This name may contain one or more `/`, the `/` must be split on and retokenized before sending.
    * Retokenizing follows the following rule: `bucket.name/scope.name` becomes `` `bucket.name` ``.`` `scope.name` ``.
    * Note that there can be multiple instance of `/` within a name and all must be retokenized.

  * `linkName`: `string` - name of the link. Default to `"Local"`.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `AnalyticsLinkNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

## GetPendingMutations

Gets the pending mutations for all datasets. This is returned from the server as an object e.g.:

```
{
    "Default": {
        "travel": 20688,
        "thing": 0,
        "default": 0
    },
    "Notdefault": {
        "default": 0
    }
}
```

Note that if a link is disconnected then it will return no results. If all links are disconnected then an empty object is returned.

### Signature

```
map[string]map[string]int GetPendingMutations( [options])
```

### Parameters

* Required: None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Map where top level keys are dataverse names, and values are a map of dataset names to number of pending mutations.

### Throws

* Any exceptions raised by the underlying platform

### URI

    GET http://localhost:8095/analytics/node/agg/stats/remaining

## CreateLink

Creates a new link.

### Signature

```
void CreateLink(AnalyticsLink link, [options])
```

### Parameters

* Required:

  * `link`: `AnalyticsLink` - the link to be created.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `LinkAlreadyExistsException`

* `AnalyticsScopeNotFoundException`

* `InvalidArgumentsException`

### URI

Data is sent as `"application/x-www-form-urlencoded"`.

* The link's `dataverse` can contain one or more `/`.
* The path to use depends on the `dataverse`.
* If `dataverse` contains a `/` then the name is url encoded to escape any `/` within the name and URI is:
  * POST http://localhost:8095/analytics/link/<dataverse>/<linkname>
* If `dataverse` does not contain a `/` then the `dataverse` and `name` are included within the payload of the request (as `dataverse` and `name`)
  * The URI is:
    * POST http://localhost:8095/analytics/link


## ReplaceLink

Replaces an existing link.

### Signature

```
void ReplaceLink(AnalyticsLink link, [options])
```

### Parameters

* Required:

  * `link`: `AnalyticsLink` - the link to be replaced.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `AnalyticsScopeNotFoundException`

* `LinkNotFoundException`

* `InvalidArgumentsException`

### URI

Data is sent as `"application/x-www-form-urlencoded"`.

* The link's `dataverse` can contain one or more `/`.
* The path to use depends on the `dataverse`.
* If `dataverse` contains a `/` then the name is url encoded to escape any `/` within the name and URI is:
  * PUT http://localhost:8095/analytics/link/<dataverse>/<linkname>
* If `dataverse` does not contain a `/` then the `dataverse` and `name` are included within the payload of the request (as `dataverse` and `name`)
  * The URI is:
    * POST http://localhost:8095/analytics/link

## DropLink

Drops an existing link from a scope.

### Signature

```
void DropLink(string linkName, string scopeName, [options])
```

### Parameters

* Required:

  * `linkName`: `string` - the link to be removed.
  * `scopeName`: `string` - the scope in which the link belongs, in the format `bucket/scope`.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `AnalyticsScopeNotFoundException`

* `LinkNotFoundException`

* `InvalidArgumentsException`

### URI

* The link's `dataverse` can contain one or more `/`.
* The path to use depends on the `dataverse`.
* If `dataverse` contains a `/` then the name is url encoded to escape any `/` within the name and URI is:
  * DELETE http://localhost:8095/analytics/link/<dataverse>/<linkname>
* If `dataverse` does not contain a `/` then the `dataverse` and `name` are included within the payload of the request (as `dataverse` and `name`)
  * The URI is:
    * POST http://localhost:8095/analytics/link

## GetAllLinks

Gets all existing links.

### Signature

```
List<AnalyticsLink> GetAllLinks([options])
```

### Parameters

* Required:

None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.
  * `Dataverse` (string) - the name of the dataverse to restrict links to.
  * `Name` (string) - the name of the link to fetch.
  * `LinkType` (AnalyticsLinkType) - the type of links to restrict returned links to.
  
Note: If `Name` is set then `Dataverse` must be set, otherwise an `InvalidArgumentsException` must be thrown.

### Returns

`List<AnalyticsLink>` - a list of the interface type which must be cast to the relevant underlying type.

### Throws

* `DataverseNotFoundException`

* `ScopeNotFoundException`

* `InvalidArgumentsException`

### URI
The URI is dependent on the options.
If `dataverse` is not set then:

    GET http://localhost:8095/analytics/link?type=<linktype>

* If `dataverse` is set and contains a `/` then the name is url encoded to escape any `/` within the name and URI is:
  *  If `dataverse` is set and `name` is not:
    * `GET http://localhost:8095/analytics/link/<dataverse>?type=<linktype>`
  *  If `dataverse` is and `name` are both set:
    * `GET http://localhost:8095/analytics/link/<scope>/<linkname>?type=<linktype>`
  
* If `dataverse` does not contain a `/` then the `dataverse` and `name` are included within the querystring of the request (as `dataverse` and `name`)
  *  Only include `name` if `name` is set.
  * Uri is:
    * `GET http://localhost:8095/analytics/link?dataverse=<dataverse>&name=<name>&type=<linktype>`

# BucketManager

```
public interface BucketManager{
    void CreateBucket(CreateBucketSettings settings, CreateBucketOptions options);

    void UpdateBucket(BucketSettings settings, UpdateBucketOptions options);

    void DropBucket(string bucketName, DropBucketOptions options);

    BucketSettings GetBucket(string bucketName, GetBucketOptions options);

    Iterable<BucketSettings> GetAllBuckets(GetAllBucketOptions options);

    void FlushBucket(string bucketName, FlushBucketOptions options); // using the ns_server REST interface
}
```

## CreateBucket

Creates a new bucket.

### Signature

```
void CreateBucket(CreateBucketSettings settings, [options])
```

### Parameters

* Required: [`BucketSettings`](#bucketsettings) - settings for the bucket.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `BucketAlreadyExistsException` (HTTP 400 and content contains `Bucket with given name already exists`)

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST http://localhost:8092/pools/default/buckets

## UpdateBucket

Updates a bucket. In the docstring the SDK must include a section about how every setting must be set to what the user wants it to be after
the update. Any settings that are not set to their desired values may be reverted to default values by the server.

### Signature

```
void UpdateBucket(BucketSettings settings, [options])
```

### Parameters

* Required: [`BucketSettings`](#bucketsettings) - settings for the bucket.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `InvalidArgumentsException`

* `BucketNotFoundException`

* Any exceptions raised by the underlying platform

### URI

    POST http://localhost:8092/pools/default/buckets/<bucketName>

## DropBucket

Removes a bucket.

### Signature

```
void DropBucket(string bucketName, [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - the name of the bucket.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `BucketNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    DELETE http://localhost:8092/pools/default/buckets/<bucketname>

## GetBucket

Gets a bucket's settings.

### Signature

```
BucketSettings GetBucket(bucketName string, [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - the name of the bucket.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

[`BucketSettings`](#bucketsettings), settings for the bucket. Note: the ram quota returned is in bytes, not megabytes so requires `x / 1024`
twice. Also Note: `FlushEnabled` is not a setting returned by the server, if flush is enabled then the `doFlush` endpoint will be listed and
should be used to populate the field.

### Throws

* `BucketNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    GET http://localhost:8092/pools/default/buckets/<bucketname>

## GetAllBuckets

Gets all bucket settings. Note: the ram quota returned is in bytes, not megabytes so requires `x  / 1024` twice.

### Signature

```
Iterable<BucketSettings> GetAllBuckets([options])
```

### Parameters

* Required: None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An iterable of settings for each bucket.

### Throws

* Any exceptions raised by the underlying platform

### URI

    GET http://localhost:8092/pools/default/buckets

## FlushBucket

Flushes a bucket (uses the ns_server REST interface).

### Signature

```
void FlushBucket(string bucketName, [options])
```

### Parameters

* Required:

  * `bucketName`: `string` - the name of the bucket.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `BucketNotFoundException`

* `InvalidArgumentsException`

* `BucketNotFlushableException`

* Any exceptions raised by the underlying platform

### URI

    POST http://localhost:8092//pools/default/buckets/<bucketname>/controller/doFlush

# UserManager

Programmatic access to the [user management REST API](https://docs.couchbase.com/server/current/rest-api/rbac.html).

Unless otherwise indicated, all objects SHOULD be immutable.

## Role

A role identifies a specific permission.

```
interface Role {
    String name()

    Optional<String> bucket()

    Optional<String> Collection()   // Uncommitted

    Optional<String> Scope()   // Uncommitted
}
```

## RoleAndDescription

Associates a role with its name and description. This is additional information only present in the "list available roles" response.

NOTE: This class MAY extend `Role` instead of having a method that returns the `Role`.

```
interface RoleAndDescription {
    Role role()

    String displayName()

    String description()
```

## Origin

Indicates why the user has a specific role. If the type is "user" it means the role is assigned directly to the user. If the type is "group"
it means the role is inherited from the group identified by the "name" field.

```
interface Origin {
    String type()

    Optional<String> name()
}
```

## RoleAndOrigins

Associates a role with its origins. This is how roles are returned by the "get user" and "get all users" responses.

NOTE: This class MAY extend Role instead of having a method that returns the Role.

```
interface RoleAndOrigins {
    Role role()

    List<Origin> origins()
```

## User

Mutable. Models the user properties that may be updated via this API. All properties of the User class MUST have associated setters except
for "username" which is fixed when the object is created.

```
interface User {
    String username()

    String displayName()

    // names of the groups
    Set<String> groups()

    // only roles assigned directly to the user (not inherited from groups)
    Set<Role> roles()

    // From the user's perspective the password property is "write-only".
    // The accessor SHOULD be hidden from the user and be visible only to the manager implementation.
    String password()
}
```

## UserAndMetadata

Models the "get user" / "get all users" response. Associates the mutable properties of a user with derived properties such as the effective
roles inherited from groups.

```
interface UserAndMetadata {
    // AuthDomain is an enumeration with values "local" and "external". It MAY alternatively be represented as String.
    AuthDomain domain()

    // returns a new mutable User object each time this method is called.
    // Modifying the fields of the returned User MUST have no effect on the UserAndMetadata object it came from.
    User user()

    // all roles associated with the user, including information about whether each role role is innate or inherited from a group.
    List<RoleAndOrigins> effectiveRoles()

    Optional<Instant> passwordChanged()

    Set<String> externalGroups()
}
```

## Group

Mutable. Defines a set of roles that may be inherited by users. All properties of the Group class MUST have associated setters except for
"name" which is fixed when the object is created.

```
interface Group {
    String name()

    String description()

    // Role as defined in the User Manager section
    Set<Role> roles()

    Optional<String> ldapGroupReference()
}
```

## Service Interface

```
public interface IUserManager{
    UserAndMetadata GetUser(string username, GetUserOptions options);

    Iterable<UserAndMetadata> GetAllUsers(GetAllUsersOptions options);

    void UpsertUser(User user, UpsertUserOptions options);

    void DropUser(string userName, DropUserOptions options);

    Iterable<RoleAndDescription> GetRoles(GetRolesOptions options);

    Group GetGroup(string groupName, GetGroupOptions options);

    Iterable<Group> GetAllGroups(GetAllGroupsOptions options);

    void UpsertGroup(Group group, UpsertGroupOptions options);

    void DropGroup(string groupName, DropGroupOptions options);
}
```

## GetUser

Gets a user.

### Signature

```
UserAndMetadata GetUser(string username, [options])
```

### Parameters

* Required:

  * `username`: `string` - ID of the user.

* Optional:

  * `domainName`: `string` - name of the user domain (`"local"` or `"external"`). Defaults to `"local"`.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An instance of [`UserAndMetadata`](#userandmetadata).

### Throws

* `UserNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform


Implementation Notes

When parsing the "get" and "getAll" responses, take care to distinguish between roles assigned directly to the user (role origin with
`type="user"`) and roles inherited from groups (role origin with `type="group"` and `name=<group name>`).

If the server response does not include an "origins" field for a role, then it was generated by a server version prior to 6.5 and the SDK
MUST treat the role as if it had a single origin of `type="user"`.

If a `Role` `Scope()` or `Collection()` properties are set to `"*"` then they should reset to an empty string.
This is to provide a consistent experience across server versions.

## GetAllUsers

Gets all users.

### Signature

```
Iterable<UserAndMetadata> GetAllUsers([options])
```

### Parameters

* Required: None

* Optional:

  * `domainName`: `string` - name of the user domain (`"local"` or `"external"`). Defaults to `"local"`.

* `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An iterable collection of [`UserAndMetadata`](#userandmetadata).

### Throws

* Any exceptions raised by the underlying platform

## UpsertUser

Creates or updates a user.

### Signature

```
void UpsertUser(User user, [options])
```

### Parameters

* Required:

  * `user`: [`User`](#user) - the new version of the user.

* Optional:

  * `domainName`: `string` - name of the user domain (`"local"` or `"external"`). Defaults to `"local"`.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

Implementation Notes

When building the PUT request to send to the REST endpoint, implementations MUST omit the `"password"` property if it is not present in the
given User domain object (so that the password is only changed if the calling code provided a new password).

For backwards compatibility with Couchbase Server 6.0 and earlier, the "groups" parameter MUST be omitted if the group list is
empty. Couchbase Server 6.5 treats the absent parameter the same as an explicit parameter with no value (removes any existing group
associations, which is what we want in this case).

For backwards compatibility with Couchbase Server 6.5 and earlier if a `Role` `Scope()` or `Collection()` properties are 
set to `"*"` then they should reset to an empty string.

## DropUser

Removes a user.

### Signature

```
void DropUser(string username, [options])
```

### Parameters

* Required:

  * `username`: `string` - ID of the user.

* Optional:

  * `domainName`: `string` - name of the user domain (`"local"` or `"external"`). Defaults to `"local"`.

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `UserNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### GetRoles

Returns the roles supported by the server.

### Signature

```
Iterable<RoleAndDescription> GetRoles([options])
```

### Parameters

* Required: None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An iterable collection of [`RoleAndDescription`](#roleanddescription).

### Throws

* Any exceptions raised by the underlying platform

## GetGroup

Gets a group.

REST Endpoint:

    GET /settings/rbac/groups/<name>

### Signature

```
Group GetGroup(string groupName, [options])
```

### Parameters

* Required:

  * `groupName`: `string` - name of the group to get.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An instance of [`Group`](#group).

### Throws

* `GroupNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform


## GetAllGroups

Gets all groups.

REST Endpoint:

    GET /settings/rbac/groups

### Signature

```
Iterable<Group> GetAllGroups([options])
```

### Parameters

* Required: None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

An iterable collection of Group.

### Throws

* Any exceptions raised by the underlying platform

## UpsertGroup

Creates or updates a group.

REST Endpoint:

    PUT /settings/rbac/groups/<name>

This endpoint accepts `application/x-www-form-urlencoded` and requires the data be sent as form data. The `name`/`id` should not be included
in the form data. Roles should be a comma separated list of strings. If, only if, the role contains a bucket name then the rolename should
be suffixed with `[<bucket_name>]` e.g. `bucket_full_access[default],security_admin`.

### Signature

```
void UpsertGroup(Group group, [options])
```

### Parameters

* Required:

  * `group`: [`Group`](#group) - the new version of the group.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform


## DropGroup

Removes a group.

REST Endpoint:

    DELETE /settings/rbac/groups/<name>

### Signature

```
void DropGroup(string groupName, [options])
```

### Parameters

* Required:

  * `groupName`: `string` - name of the group.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `GroupNotFoundException`

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

# CollectionsManager

Note: due to [MB-35386](https://issues.couchbase.com/browse/MB-35386) `CollectionExists`, `ScopeExists`, and `GetScope` must support both
forms of the manifest until SDK beta. That is, going forward:

    {"uid":"0","scopes":[{"name":"_default","uid":"0","collections":[{"name":"_default","uid":"0"}]}]}

And for the sake of support for the server beta which did not have the fix in place:

    {"uid":0,"scopes":{"_default":{"uid":0,"collections":{"_default":{"uid":0}}}}}

It is recommended to try to parse the first form and fallback to the second so that it can be easily removed later.

```
public interface ICollectionManager{
    ScopeSpec GetScope(string scopeName, GetScopeOptions options);

    Iterable<ScopeSpec> GetAllScopes(GetAllScopesOptions options);

    void CreateCollection(CollectionSpec collection, CreateCollectionOptions options);

    void DropCollection(CollectionSpec collection, DropCollectionOptions options);

    void CreateScope(string scopeName, CreateScopeOptions options);

    void DropScope(string scopeName, DropScopeOptions options);
}
```

## GetAllScopes

Gets all scopes. This will fetch a manifest and then pull the scopes out of it.

### Signature

```
iterable<ScopeSpec> GetAllScopes([options])
```

### Parameters

* Required: None

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Iterable collection of [`ScopeSpec`](#scopespec).

### Throws

* Any exceptions raised by the underlying platform

### URI

    GET /pools/default/buckets/<bucket>/collections

## CreateCollection

Creates a new collection.

### Signature

```
void CreateCollection(CollectionSpec collection, [options])
```

### Parameters

* Required:

  * `collection`: [`CollectionSpec`](#collectionspec) - specification of the collection.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

  * `maxExpiry` (`duration`) - Value for the max expiry of new documents created without an expiry.

### Returns

Nothing

### Throws

* `InvalidArgumentsException`

* `CollectionAlreadyExistsException`

* `ScopeNotFoundException`

* Any exceptions raised by the underlying platform

### URI

    POST http://localhost:8091/pools/default/buckets/<bucket>/collections/<scope_name> -d name=<collection_name> -d maxTTL=<maxExpiry>

## DropCollection

Removes a collection.

### Signature

```
void DropCollection(CollectionSpec collection, [options])
```

### Parameters

* Required:

  * `collection`: [`CollectionSpec`](#collectionspec) - specification of the collection.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `CollectionNotFoundException`

* Any exceptions raised by the underlying platform

### URI

    DELETE http://localhost:8091/pools/default/buckets/<bucket>/collections/<scope_name>/<collection_name>

## CreateScope

Creates a new scope.

### Signature

```
void CreateScope(string scopeName, [options])
```

### Parameters

* Required:

  * `scopeName`: `string` - name of the scope.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `InvalidArgumentsException`

* Any exceptions raised by the underlying platform

### URI

    POST http://localhost:8091/pools/default/buckets/<bucket>/collections -d name=<scope_name>

## DropScope

Removes a scope.

### Signature

```
void DropScope(string scopeName, [options])
```

### Parameters

* Required:

  * `scopeName`: `string` - name of the scope.

* Optional:

  * `Timeout` or `timeoutMillis` (`int`/`duration`) - the time allowed for the operation to be terminated. This is controlled by the client.

### Returns

Nothing

### Throws

* `ScopeNotFoundException`

* Any exceptions raised by the underlying platform

### URI

DELETE http://localhost:8091/pools/default/buckets/<bucket>/collections/<scope_name>

# Types

## DesignDocumentNamespace

```
enum DesignDocumentNamespace {
    PRODUCTION,

    DEVELOPMENT
}
```

## DesignDocument

`DesignDocument` provides a means of mapping a design document into an object. It contains within it a map of [`View`](#view).

```
interface DesignDocument {
    String Name();

    Map<String, View> Views();
}
```

## View

```
interface View {
    String Map();

    String Reduce();
}
```

## QueryIndexType

```
enum QueryIndexType {
    VIEW("view"),

    GSI("gsi")
}
```

## QueryIndex

The `QueryIndex` interface provides a means of mapping a query index into an object.

```
interface QueryIndex {
    String Name();

    Bool IsPrimary();

    QueryIndexType Type();

    String State();

    String Keyspace();

    Iterable<String> IndexKey();

    Optional<String> Condition();
    
    Optional<String> Partition()
}
```

## SearchIndex

`SearchIndex` provides a means to map search indexes into object. The `Params`, `SourceParams`, and `PlanParams` fields are left to the SDK
to decide what is idiomatic (e.g. `JSONObject`), the definitions of these fields is purposefully vague.

```
type SearchIndex {
    // UUID is required for updates. It provides a means of ensuring consistency, the UUID must match the UUID value
    // for the index on the server.
    UUID string `json:"uuid"`

    Name string `json:"name"`

    // SourceName is the name of the source of the data for the index e.g. bucket name.
    SourceName string `json:"sourceName,omitempty"`

    // Type is the type of index, e.g. fulltext-index or fulltext-alias.
    Type string `json:"type"`

    // IndexParams are index properties such as store type and mappings.
    Params map[string]interface{} `json:"params"`

    // SourceUUID is the UUID of the data source, this can be used to more tightly tie the index to a source.
    SourceUUID string `json:"sourceUUID,omitempty"`

    // SourceParams are extra parameters to be defined. These are usually things like advanced connection and tuning
    // parameters.
    SourceParams map[string]interface{} `json:"sourceParams,omitempty"`

    // SourceType is the type of the data source, e.g. couchbase or nil depending on the Type field.
    SourceType string `json:"sourceType"`

    // PlanParams are plan properties such as number of replicas and number of partitions.
    PlanParams map[string]interface{} `json:"planParams,omitempty"`
}
```

## AnalyticsDataset

`AnalyticsDataset` provides a means of mapping dataset details into an object.

```
interface AnalyticsDataset{
    String Name();

    String DataverseName();

    String LinkName();

    String BucketName();
}
```

## AnalyticsIndexInterface

`AnalyticsIndex` provides a means of mapping analytics index details into an object.

```
interface AnalyticsIndex{
    String Name();

    String DatasetName();

    String DataverseName();

    Bool IsPrimary();
}
```

## AnalyticsLinkInterface

`AnalyticsLink` provides a means of mapping analytics link details into an object.
The interface methods are primarily designed for creation and modification of links.
Note: Analytics link management is only supported for server 7.0+.

```
interface AnalyticsLink {
  List<Byte> FormEncode();
  
  Validate();
  
  AnalyticsLinkType LinkType();
}
```

### CouchbaseRemoteAnalyticsLink

`CouchbaseRemoteAnalyticsLink` provides a means of mapping remote couchbase analytics link details into an object.
The following fields are named according to how they are sent when form/json encoded.
The object names should be implemented in a way idiomatic to the implementing language.

* `dataverse` (string) - The dataverse that the link belongs to. Form can be one part `dataversename` or two parts `bucket.name/scope.name`.
* `name` (string) - The name of the link.
* `hostname` (string) - The hostname of the target couchbase cluster. This is `activeHostname` in the returned JSON payload for a get.
* `encryption` (CouchbaseAnalyticsEncryptionSettings) - The encryption settings for the link. The contents of this field are flattened to the top level of the payload.
* `username` (string) - The username to use for authentication.
* `password` (string) - The password to use for authentication.

A value for `type` of `couchbase` must be sent in the form payload, this is returned in the same manner in the JSON payload.
This field is not exposed on the object and is either inferred from the object type or by calling `LinkType()`.

The `Validate()` implementation must verify that:
* `dataverse` is set.
* `name` is set.
* `hostname` is set.
* When encryption level is set to "none" or "half":
  * `username` is set.
  * `password` is set.
* When encryption level is set to "full":
  * `certificate` is set.
  * Either both `username` and `password` are set OR both `clientCertificate` and `clientKey` are set.

The `LinkType()` implementation must return a value that corresponds to `"couchbase"`.

On fetching a `CouchbaseRemoteAnalyticsLink`:
  * The `hostname` property is present as `activeHostname`.
  * The `password` property must be blanked out/left unset.
  * The `clientKey` property must be blanked out/left unset.
  * The `dataverse` property must be set by first checking the `dataverse` field in the response body and if empty falling back to the `scope` field.

### CouchbaseAnalyticsEncryptionSettings

`CouchbaseAnalyticsEncryptionSettings` are the settings available for setting encryption level on an analytics link.

* `encryption` (AnalyticsEncryptionLevel) - The encryption level to apply.
* `certificate` (List<Byte>) - The certificate to use for the encryption when encryption level is set to "full".
* `clientCertificate` (List<Byte>) - The client certificate to use for authenticating when encryption level is set to "full".
* `clientKey` (List<Byte>) - The client key to use for authenticating when encryption level is set to "full".

### S3ExternalAnalyticsLink

`S3ExternalAnalyticsLink` provides a means of mapping external s3 analytics link details into an object.
The following fields are named according to how they are sent when form/json encoded.
The object names should be implemented in a way idiomatic to the implementing language.

* `dataverse` (string) - The dataverse that the link belongs to. Form can be one part `dataversename` or two parts `bucket.name/scope.name`.
* `name` (string) - The name of the link.
* `accessKeyID` (string) - The AWS S3 access key.
* `secretAccessKey` (string) - The AWS S3 secret key.
* `sessionToken` (string) - The AWS S3 token if temporary credentials are provided. Only available in server 7.0+.
* `region` (string) - The AWS S3 region.
* `serviceEndpoint` (string) - The AWS S3 service endpoint.

A value for `type` of `s3` must be sent in the form payload, this is returned in the same manner in the JSON payload.
This field is not exposed on the object and is either inferred from the object type or by calling `LinkType()`.

The `Validate()` implementation must verify that:
* `dataverse` is set.
* `name` is set.
* `accessKeyID` is set.
* `secretAccessKey` is set.
* `region` is set.

The `LinkType()` implementation must return a value that corresponds to `"s3"`.

On fetching a `S3ExternalAnalyticsLink`:
* The `secretAccessKey` property must be blanked out/left unset.
* The `sessionToken` property must be blanked out/left unset.
* The `dataverse` property must be set by first checking the `dataverse` field in the response body and if empty falling back to the `scope` field.

### AzureBlobExternalAnalyticsLink

`AzureBlobExternalAnalyticsLink` provides a means of mapping external azure analytics link details into an object.
The `AzureBlobExternalAnalyticsLink` is available in server 7.0 Developer Preview mode and must be marked as `volatile` API stability.
The following fields are named according to how they are sent when form/json encoded.
The object names should be implemented in a way idiomatic to the implementing language.

* `dataverse` (string) - The dataverse that the link belongs to. Form is `bucket.name/scope.name`.
* `name` (string) - The name of the link.
* `connectionString` (string) - The connection string can be used as an authentication method, connectionString contains other authentication methods embedded inside the string. Only a single authentication method can be used. (e.g. “AccountName=myAccountName;AccountKey=myAccountKey”).
* `accountName` (string) - The Azure blob storage account name.
* `accountKey` (string) - The Azure blob storage account key.
* `sharedAccessSignature` (string) - Token that can be used for authentication.
* `blobEndpoint` (string) - The Azure blob storage endpoint.
* `endpointSuffix` (string) - The Azure blob endpoint suffix.

A value for `type` of `azureblob` must be sent in the form payload, this is returned in the same manner in the JSON payload.
This field is not exposed on the object and is either inferred from the object type or by calling `LinkType()`.

The `Validate()` implementation must verify that:
* `dataverse` is set.
* `name` is set.
* Either `connectionString` is set, or `accountName` and `accountKey` are both set, or `accountName` and `sharedAccessSignature` are both set.

The `LinkType()` implementation must return a value that corresponds to `"azureblob"`.

On fetching an `AzureBlobExternalAnalyticsLink`:
* The `connectionString` property must be blanked out/left unset.
* The `accountKey` property must be blanked out/left unset.
* The `sharedAccessSignature` property must be blanked out/left unset.
* The `dataverse` property must be set from the `scope` field in the response body.

### AnalyticsLinkType

`AnalyticsLinkType` describes the type of link that an object corresponds to.

```
enum AnalyticsLinkType {
  S3External("s3")
  
  AzureBlobExternal("azureblob")
  
  CouchbaseRemote("couchbase")
}
```

### AnalyticsEncryptionLevel

`AnalyticsEncryptionLevel` describes the encryption level for a couchbase analytics link.

```
enum AnalyticsEncryptionLevel {
  NONE("none")
  
  HALF("half")
  
  FULL("full")
}
  
```

## CompressionMode

```
enum CompressionMode {
    OFF("off"),

    PASSIVE("passive"),

    ACTIVE("active")
}
```

## EvictionPolicy

```
enum EvictionPolicy {
    FULL("fullEviction"),

    VALUE_ONLY("valueOnly"),

    NOT_RECENTLY_USED("nruEviction"),

    NO_EVICTION("noEviction")
}
```

## BucketType

```
enum BucketType {
    COUCHBASE("membase"),

    MEMCACHED("memcached"),

    EPHEMERAL("ephemeral")
}
```

## BucketSettings

`BucketSettings` provides a means of mapping bucket settings into an object.

* `Name` (`string`) - The name of the bucket.

* `FlushEnabled` (`bool`) - Whether or not flush should be enabled on the bucket. Default to `false`.

* `RamQuotaMB` (`int`) - Ram Quota in megabytes for the bucket. (`rawRAM` in the server payload)

* `NumReplicas` (`int`) - The number of replicas for documents.

* `ReplicaIndexes` (`bool`) - Whether replica indexes should be enabled for the bucket.

* `BucketType` ([`BucketType`](#buckettype)) - The type of the bucket. Default to `COUCHBASE`.

* `EjectionMethod` {`fullEviction` | `valueOnly`}. (**DEPRECATED**) The eviction policy to use.

* `EvictionPolicyType` ([`EvictionPolicy`](#evictionpolicy)). The eviction policy to use. Options exposed as {`FULL`, `VALUE_ONLY`,
  `NOT_RECENTLY_USED`, `NO_EVICTION`}. The first 2 options are valid only on couchbase buckets and the last 2 options are valid only on
  ephemeral buckets.

* `maxTTL` (`int`) (**DEPRECATED**) - Value for the `maxTTL` of new documents created without a ttl.

* `maxExpiry` (`duration`) - Value for the max expiry of new documents created without an expiry.

* `compressionMode` ([`CompressionMode`](#compressonmode)) - The compression mode to use.

* `MinimumDurabilityLevel` (`DurabilityLevel`) - The minimum durability level to use for all KV operations.

## ConflictResolutionType

```
enum ConflictResolutionType {
    TIMESTAMP("lww"),

    SEQUENCE_NUMBER("seqno")
}
```

## CreateBucketSettings

`CreateBucketSettings` is a superset of [`BucketSettings`](#bucketsettings) providing one extra property:

* `ConflictResolutionType` ([`ConflictResolutionType`](#conflictresolutiontype)). The conflict resolution type to use.

The reasoning for this is that on Update `ConflictResolutionType` cannot be present in the JSON payload at all.

## CollectionSpec

```
interface CollectionSpec {
    String Name();

    String ScopeName();

    Duration MaxExpiry();
}
```

`MaxExpiry` is the time in seconds for the TTL for new documents in the collection. Omit if not set.


## ScopeSpec

```
interface ScopeSpec {
    String Name();

    Iterable<CollectionSpec> Collections();
}
```

# References

* Query index management

  * https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0003-indexmanagement.md

  * https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/

* Search index management

  * https://docs.google.com/document/d/1C4yfTj5u6ahRgk3ZIL_AkwPMeu9-hHY_lZcsDNeIP74

  * https://docs.couchbase.com/server/6.0/rest-api/rest-fts-indexing.html

  * http://labs.couchbase.com/cbft/dev-guide/index-definitions/ Outdated but contains some useful definitions for fields.

* User Management

  * https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0022-usermgmt.md

* Bucket Management

  * https://docs.couchbase.com/server/current/rest-api/rest-bucket-intro.html

* Collection Management

  * https://github.com/couchbase/ns_server/commit/4c1a9ea20fbdb148f68ec2ad7be9eed4c071bf9e

  * https://docs.google.com/document/d/1X-v8GWQjplrMMaYwwWOzEuP4AUoDNIAvS39NmEjQ3_E/edit#heading=h.gprum229kohk

* Views REST API

  * https://docs.couchbase.com/server/6.0/rest-api/rest-views-intro.html

* Groups and LDAP Groups

  * https://docs.google.com/document/d/1wsjEmke80RW_sbW0ycS8yQl6rq5lzBbRWuelPZ1m9ZI/edit#

  * https://docs.google.com/document/d/1cFZOz9n6ZKyq_thxCSteuLrZ0rJEh7AMT3MFbOHCchM/edit

  * https://issues.couchbase.com/browse/MB-16035

* Analytics management

  * https://docs.couchbase.com/server/current/analytics/5_ddl.html
  * https://docs.google.com/document/d/1tpcHrcsTZj2bme8yqL5QEQeMMOG9LerOaDD3caXgJ3U
  * https://docs-staging.couchbase.com/server/7.0/analytics/rest-links.html

# Changes

* Sept 3, 2019 - Revision #1 (by Charles Dixon)

  * Add `numReplicas` option to `QueryIndexManager.CreatePrimaryIndex`

* Sept 18, 2019 - Revision #2 (by Charles Dixon)

  * Add `FlushNotEnabledException` to bucket Flush operation

* Sept 30, 2019 - Revision #3 (by David Nault)

  * Remove dataverse name checking from all Analytics options blocks so that if the dataverse name is set we do not check if the dataset
    already contains it.

  * Changed `CreateScope` to take a string name not a spec.

  * Added Condition to IQueryIndex.

  * Changed User AvailableRoles to GetRoles.

  * Change ConnectLink and DisconnectLink so that the name is optional, defaulting to Local.

* Nov 10, 2019 - Revision #4 (by David Nault)

  * Add optional arguments “DataverseName” and “Force” to Analytics ConnectLink.

  * Remove the FlushCollection method from CollectionManager, since server-side flushing was not implemented. The user can drop and
    recreate the collection instead.

* Clarify that “IndexType” is string/enum with values “View” and “Gsi”.

  * Clarify that all optional timeouts default to the client’s global management request timeout.

  * Clarify that all optional “IgnoreIfExists” parameters default to false.

  * Clarify that the Query Index Manager's optional "NumReplicas" and "Deferred" parameters default to omitting the parameter from the
    server request and letting the server decide the default value.

* Nov 11, 2019 - Revision #5 (by David Nault)

  * Renamed “IndexType” to “QueryIndexType” to disambiguate.

* Nov 20, 2019 - Revision #6 (by David Nault)

  * Collection Manager:

     * Removed “scopeExists” and “collectionExists” methods.

  * User Manager:

     * RoleAndOrigins and RoleAndDescription MAY extend Role, at implementor’s discretion.

     * Removed the version of UserAndMetadata.effectiveRoles() that returned Set<Role>.

     * Renamed UserAndMetadata.effectiveRolesAndOrigins() to just effectiveRoles().

* Mar 10, 2020 - Revision #7 (by Charles Dixon)

  * Collection Manager:

     * Added “MaxTTL” to the CollectionSpec.

     * Added how to send “MaxTTL” to the CreateCollection uri.

* Mar 26, 2020 - Revision #8 (by Charles Dixon)

  * Collection Manager:

     * Change “maxTTL” to “maxExpiry”

  * Bucket Manager:

     * Deprecate maxTTL (int) and add maxExpiry (duration)

* Apr 16, 2020 - Revision #9 (by Brett Lawson)

  * GetAllIndexes (n1ql): added mandatory using = gsi to exclude fts indexes

  * Changed AnalyzeDocument to take AnalyzeDocumentOptions instead of AnalyzeDocOptions

  * Analytics CreateIndex must take dataset name (fixed in the summary table)

  * Removed `GetScope` from collection manager as a high-level decision was made to avoid obscuring the underlying behaviours of the
    server. `GetScope` internally had to call `GetAllScopes`, which simply caused data to be thrown away unnecessarily.

* April 30, 2020

  * Moved RFC to ACCEPTED state.

* July 16, 2020 - Revision #10 (by Charles Dixon)

  * Deprecated `EjectionMethod` in favour of `EvictionPolicyType`.

  * Added 2 new eviction policies: `NOT_RECENTLY_USED` and `NO_EVICTION` for use with ephemeral buckets.

* August 28, 2020

  * Converted to markdown

* September 16, 2020 - Revision #11 (by Charles Dixon)

  * Changed the return value of Analytics `GetPendingMutations` from `map[string]int` to `map[string]map[string]int`.
  
* September 22, 2020 - Revision #12 (by Charles Dixon)

  * Add `Collection` and `Scope` to `Role`.

* November 25, 2020 - Revision #13 (by Charles Dixon)

  * Added `MinimumDurabilityLevel` to `BucketSettings`.

* March 3, 2021 - Revision #14 (by Charles Dixon)

  * Added `Partition` to `QueryIndex`.

* April 29, 2021 - Revision #15 (by Charles Dixon)

  * Added Analytics Links management to `AnalyticsIndexManager`.
  * Added LinkName to `AnalyticsIndexManager` `CreateDataset` options.

* May 11, 2021 - Revision #16 (by Charles Dixon)

  * Rework Analytics Links management to only support server 7.0+.
    * Removed `Dataverse` links functions and from links objects.
    * Removed `GetLink` functions all together.
    * Added `name` to `GetAllLinksOptions`.
  
* May 17, 2021 - Revision #17 (by Charles Dixon)
  
  * Rework Analytics Link management to once again support server 6.6
    * Updated all link and method parameters to refer to `dataverse` only.
    * Clarified behaviour to follow on parsing link response bodies.
    * Added detail on how to parse `dataverse` names and which URLs to use depending on the name.
  * Update Analytics management API to include information on how to parse `dataverse` names containing `/`.

# Signoff

| Language   | Team Member         | Signoff Date   | Revision |
|------------|---------------------|----------------|----------|
| Node.js    | Brett Lawson        | April 16, 2020 | #9       |
| Go         | Charles Dixon       | April 29, 2021 | #17      |
| Connectors | David Nault         | April 29, 2020 | #9       |
| PHP        | Sergey Avseyev      | April 22, 2020 | #9       |
| Python     | Ellis Breen         | April 29, 2020 | #9       |
| Scala      | Graham Pople        | April 30, 2020 | #9       |
| .NET       | Jeffry Morris       | April 22, 2020 | #9       |
| Java       | Michael Nitschinger | April 16, 2020 | #9       |
| C          | Sergey Avseyev      | April 22, 2020 | #9       |
| Ruby       | Sergey Avseyev      | April 22, 2020 | #9       |



