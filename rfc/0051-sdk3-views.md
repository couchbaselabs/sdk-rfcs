# Meta

*   RFC Name: SDK3 Views
*   RFC ID: 0051-sdk3-views
*   Start Date: 2019-09-18
*   Owner: Sergey Avseyev
*   Current Status: ACCEPTED
*   Relates To:
    [0054-sdk3-management-apis](0054-sdk3-management-apis.md),
    [0058-sdk3-error-handling](0058-sdk3-error-handling.md),
    [0059-sdk3-foundation](0059-sdk3-foundation.md)

# Summary

This RFC is part of the bigger SDK 3.0 RFC and describes the Views APIs in detail.

# Technical Details

API entry endpoint:

```
interface IBucket {
    ...
    ViewResult ViewQuery(string designDocument, string viewName, [ViewOptions options]);
    ...
}
```

## IBucket::ViewQuery

The `IBucket::ViewQuery` method enables a user to query a view index and receive the results in a streaming manner. The SDK should perform
this by executing a HTTP GET request against the views service.

### Signature

```
ViewResult ViewQuery(string designDocument, string viewName, [ViewOptions options]);
```

### Parameters

* Required

    * `designDocument`: `string`

      Specifies the name of the design document containing the view which is to be queries.

    * `viewName`: `string`

      Specifies the name of the view to query.

* Optional (part of `ViewOptions`)

    * `includeDocuments` (`bool`) = undefined (`false`)

      If specified, the SDK must fetch the document for every key in the result set and expose it via ViewRow.Document. This is not
      mandatory to implement, and rather optimization for synchronous-only SDKs, that don't have background IO worker.

    * `scanConsistency` (`ViewScanConsistency`) = undefined (`NotBounded`)

      Specifies the level of consistency for the query. Sent within the HTTP request query as `stale` and is encoded as:
        * `RequestPlus`: `false`
        * `UpdateAfter`: `update_after`
        * `NotBounded`: `ok`

    * `skip` (`uint32`) = undefined (`0`)

      Specifies the number of results to skip from the start of the result set. Sent within the HTTP request query as `skip` and is encoded
      as a decimal string (`14`).

    * `limit` (`uint32`) = undefined (unlimited)

      Specifies the maximum number of results to return. Sent within the HTTP request query as `limit` and is encoded as a decimal string
      (`5`).

    * `startKey` (`JsonValue`) = undefined

      Specifies the key to skip too before beginning to return results. Sent within the HTTP request query as `startkey` and is encoded as a
      JSON string (`{}`).

    * `endKey` (`JsonValue`) = undefined

      Specifies the key to stop returning results at. Sent within the HTTP request query as `endkey` and is encoded as a JSON string (`{}`,
      `"forum_post_42"`).

    * `startKeyDocId` (`string`) = undefined

      Specifies the document id to start returning results at within a number of results should startKey have multiple entries within the
      index. Sent within the HTTP request query as `startkey_docid` and is encoded as direct string (`hello`).

    * `endKeyDocId` (`string`) = undefined

      Specifies the document id to stop returning results at within a number of results should endKey have multiple entries within the
      index. Sent within the HTTP request query as `endkey_docid` and is encoded as direct string (`hello`).

    * `inclusiveEnd` (`boolean`) = undefined (`false`)

      Specifies whether the endKey/endKeyDocId values above should be inclusive or exclusive. Sent within the HTTP request query as
      `inclusive_end` and is encoded as a boolean string (`true` or `false`).

    * `group` (`boolean`) = undefined (`false`)

      Specifies whether to enable grouping of the results. Sent within the HTTP request query as `group` and is encoded as a boolean string
      (`true` or `false`).

    * `groupLevel` (`uint32`) = undefined (`unlimited`)

      Specifies the depth within the key to group results. Sent within the HTTP request query as `group_level` and is encoded as a decimal
      string (`14`).

    * `key` (`JsonValue`) = undefined

      Specifies a specific key to fetch from the index. Sent within the HTTP request query as `key`.

    * `keys` (`[]JsonValue`) = undefined

      Specifies a specific set of keys to fetch from the index. Sent within the HTTP request query as `keys`.

    * `order` (`ViewOrdering`) = undefined (`Descending`)

      Specifies the order of the results that should be returned. Sent within the HTTP request query as `descending` and is encoded as:
        * `Ascending`: `false`
        * `Descending`: `true`

    * `reduce` (`boolean`) = undefined (`false`)

      Specifies whether to enable the reduction function associated with this particular query index. Sent within the HTTP request query as
      `reduce` encoded as a boolean string (`true` or `false`).

    * `onError` (`ViewErrorMode`) = undefined(`Stop`)

      Specifies the behaviour of the query engine should an error occur during the gathering of view index results which would result in
      only partial results being available. Sent within the HTTP request query as `on_error` and is encoded as:
        * `Continue`: `continue`
        * `Stop`: `stop`

    * `debug` (`boolean`) = undefined (`false`)

      Allows to return debug information as part of the view response. Sent within the HTTP request query as `debug` and is encoded as a
      boolean string (`true` or `false`).

    * `raw` (`string key, string value`)

      Allows the user to specify a custom parameter to be passed within the HTTP request's query string and should be URI encoded inside the
      SDK before being sent to the server. This is used as an escape hatch in case the user wishes to utilize an option which is not
      otherwise exposed.

    * `namespace` (`DesignDocumentNamespace`) = `Production`

      Specifies whether the SDK should prefix the design document name with a `"dev_"` prefix. See enum definition in
      [0054-sdk3-management-apis](0054-sdk3-management-apis.md).

    * `timeout` (`Duration`) = `$Cluster::viewOperationTimeout`

      Specifies how long to allow the operation to continue running before it is canceled (default value defined in
      [0059-sdk3-foundation](0059-sdk3-foundation.md)).

    * `serializer` (`JsonSerializer`) = `$Cluster::Serializer`

      Specifies the serializer which should be used for deserialization of keys and values which are read from the `ViewRow`s.
      `JsonSerializer` interface is defined in [0055-sdk3-transcoders-and-serializers](0055-sdk3-transcoders-and-serializers.md).

### Returns

A ViewResult that maps the result of the view query to an object.

### Throws

* Documented
    * `ViewNotFoundException` (#501)
    * `RequestTimeoutException` (#1)
    * `CouchbaseException` (#0)

* Undocumented
    * `RequestCanceledException` (#2)
    * `InvalidArgumentException` (#3)
    * `ServiceNotAvailableException` (#4)
    * `InternalServerException` (#5)
    * `AuthenticationException` (#6)

## Return Types

### ViewResult

Represents the resulting data from a view query.

```
struct ViewResult {
    Stream<ViewRow> Rows();
    Promise<ViewMetaData> MetaData();
}
```

### ViewRow

Represents a single row of data returned from a view index. The `JsonSerializer` which is specified as part of the originating query should
be used to perform decoding of the key and value contained within the row. Note that the `Document` field in this structure is only included
if the `IncludeDocument` parameter is set on the view query. Similarly, the `Id` field is excluded from view query results if a reduction is
being used.

```
struct ViewRow {
    Optional<String> Id();
    K Key<K>();
    V Value<V>();
    Optional<D> Document<D>();
}
```

### ViewMetaData

Represents the meta-data returned along with a view query.

```
struct ViewMetaData {
    uint64 TotalRows();
    Optional<JsonObject> debug();
}
```

# Changelog

* Sept 18, 2019 - Revision #1 (by Sergey Avseyev)
    * Initial Draft

* Sept 26, 2019 - Revision #2 (by Brett Lawson)
    * Expanded upon a number of behaviours to remove ambiguity.
    * Added missing Serializer option to ViewQuery options.
    * Added debug option to ViewQuery options and ViewMetaData.
    * Removed fullset from ViewQuery options
    * Renamed continueOnError to onError in ViewQuery options.
    * Reformatted the RFC
    * Converted to use DesignDocumentNamespace from management RFC #54

* Oct 23, 2019 - Revision #3 (by Sergey Avseyev)
    * Added Document accessor for ViewRow
    * Added includeDocuments options

* Jan 10, 2020 - Revision #4 (by Sergey Avseyev)
    * Clarified that ViewOptions#includeDocuments is optional to implement.

* Feb 06, 2020 - Revision #5 (by Sergey Avseyev)
    * Added Optional to ViewRow Id and Document fields which are optional depending on various configuration options for the view query.  Also clarified their behaviour in the description for ViewRow.

* April 30, 2020
    * Moved RFC to ACCEPTED state.

* August 27, 2020
    * Converted to Markdown.

# Signoff

| Language   | Team Member         | Signoff Date   | Revision |
|------------|---------------------|----------------|----------|
| Node.js    | Brett Lawson        | April 16, 2020 | #5       |
| Go         | Charles Dixon       | April 22, 2020 | #5       |
| Connectors | David Nault         | April 29, 2020 | #5       |
| PHP        | Sergey Avseyev      | Sept 27, 2019  | #5       |
| Python     | Ellis Breen         | April 29, 2020 | #5       |
| Scala      | Graham Pople        | April 30, 2020 | #5       |
| .NET       | Jeffry Morris       | April 22, 2020 | #5       |
| Java       | Michael Nitschinger | April 16, 2020 | #5       |
| C          | Sergey Avseyev      | Sept 27, 2019  | #5       |
| Ruby       | Sergey Avseyev      | Sept 27, 2019  | #5       |
