# Meta

*   RFC Name: SDK3 Full Text Search
*   RFC ID: 0052-sdk3-full-text-search
*   Start Date: 2019-09-18
*   Owner: Sergey Avseyev
*   Current Status: ACCEPTED
*   Relates To:
    [0059-sdk3-foundation](0059-sdk3-foundation.md)

# Summary

This RFC is part of the bigger SDK 3.0 RFC and describes the FTS APIs in detail.

# Technical Details

## API Entry Point

The full text search API is located at Cluster level (for querying global FTS indexes):

```
interface ICluster {
    ...
    SearchResult SearchQuery(string indexName, SearchQuery query, [SearchOptions options]);
    ...
}
```

and at the Scope level (for querying scope FTS indexes):

```
interface IScope {
    ...
    SearchResult SearchQuery(string indexName, SearchQuery query, [SearchOptions options]);
    ...
}
```

## ICluster::SearchQuery

Executes a Search or FTS query against the remote cluster and returns a ISearchResult implementation with the results of the query.

### Signature

```
SearchResult SearchQuery(string indexName, SearchQuery query, [SearchOptions options]);
```

### Parameters

* Required

    * `indexName` (`string`)

      the name of the search index to query

    * `query`  (`SearchQuery`)

      the instance of class implementing `SearchQuery` interface. See ["SearchQuery Implementations"](#searchquery-implementations) section
      below for more details.

* Optional (part of `SearchOptions`):

    * `limit` (`uint32`) = undefined (`10`). JSON path: `size`

      The `limit` option limits the number of matches returned from the complete result set. The actual HTTP query field is called `size`,
      but to match it with N1QL and Views it makes sense to name it similarly (also aliased to `limit` on server side since MadHatter (6.5),
      see [MB-32666](https://issues.couchbase.com/browse/MB-32666)).

    * `skip` (`uint32`) = undefined (`0`). JSON path: `from`

      The `skip` option indicates how many matches are skipped on the result set before starting to return the matches. The actual HTTP
      query field is called `from`, but to match it with N1QL and Views it makes sense to name it similarly (also aliased to `offset` on
      server side since MadHatter (6.5), see [MB-32666](https://issues.couchbase.com/browse/MB-32666)).

    * `explain` (`bool`) = `false`. JSON path: `explain`

      The boolean `explain` field triggers inclusion of additional search result score explanations.

    * `highlight` (`Optional<HighlightStyle> style, Optional<string[]> fields`) = undefined (unset). JSON path: `highlight`

          enum HighlightStyle {
              HTML("html"),
              ANSI("ansi")
          }

       The `HighlightStyle` is an enumeration (or similar with a fixed set of possible values) which describes the highlighters the Server
       supports. Both `style` and `field` are optional. If no `field` is provided FTS will just highlight all fields where there is a
       match. Some languages might need to support a `null` style for the combination where no style is requested, but specific fields
       are. When none style, nor fields specified, the server will highlight all fields using default style.

    * `fields` (`string[]`) = undefined (unset). JSON path: `fields`

      This option describes a list of field values which should be retrieved for result documents, provided they were stored while indexing.

    * `scanConsistency` (`SearchScanConsistency`) = undefined (`NotBounded`). JSON path: `ctl.consistency.level`

          enum SearchScanConsistency {
              NotBounded("")
          }

       overrides `consistentWith`

    * `consistentWith` (`MutationState`) = undefined. JSON paths:
      * `ctl.consistency.level` = `"at_plus"`
      * `ctl.consistency.vectors`

        contains the consistency vector for each index (a JSON object keyed by FTS index name). A single vector is also a JSON object. The
        keys are in the `"vbucketId/vbucketUUID"` format (can be built from a `MutationToken`) and the values are `uint64` seqno sequence
        numbers.

    * `sort` (`string[]`) = undefined (`["-_score"]`). JSON path: `sort`

      The `_score` and `_id` fields are special since they allow to either sort on the document score or on the document ID. Also, if a
      field is prefixed with `"-"` the sorting order is descending.

    * `sort` (`SearchSort[]`) = undefined (undefined). JSON path: `sort`

      In addition to the simple string sort outlined above, an advanced sorting mechanism should be supported which allows the user to
      provide more detailed requirements on sorting that can't be expressed in a simple string. The method overrides variant with list of
      the fields. See possible implementations of `SearchSort` below

        * `SearchSortScore`. Sorts by hit score. JSON paths:
          * `by` = `"score"`
          * `desc` (`boolean`) = undefined (`false`)

        * `SearchSortId`. Sorts by the document ID. JSON paths:
          * `by` = `"id"`
          * `desc` (`boolean`) = undefined(`false`): if descending order should be applied

        * `SeachSortField`. Sorts by a field in the hits. JSON paths:
          * `by` =  `"field"`
          * `desc` (`boolean`) = undefined(`false`): if descending order should be applied
          * `field` (`string`): the name of the field to sort on
          * `type` (`string`) = undefined (`"auto"`): possible values are `"auto"`, `"string"`, `"number"`, `"date"`
          * `mode`  (`string`) = undefined (`"default"`): possible values are `"default"`, `"min"`, `"max"`
          * `missing` (`string`) = undefined (`"last"`): possible values are `"first"`, `"last"`

        * `SearchSortGeoDistance`. Sorts by geographic location distance in the hits. JSON paths:
          * `by` = `"geo_distance"`
          * `desc` (`boolean`) = undefined(`false`): if descending order should be applied
          * `field` (`string`): the name of the field to sort on
          * `location` (`float32[2]`): similar to geo querying: `[lon, lat]`
          * `unit` (`string`) = undefined (`Meters`): the unit multiplier which should be used for sorting. Default multiplier is
            `Meters`. Acceptable values.

                enum SearchGeoDistanceUnits {
                    Meters("meters"),
                    Miles("miles"),
                    Centimeters("centimeters"),
                    Millimeters("millimeters"),
                    NauticalMiles("nauticalmiles"),
                    Kilometers("kilometers"),
                    Feet("feet"),
                    Yards("yards"),
                    Inch("inch")
                }


    * `facets` (`Map<string name, SearchFacet facet>`) = undefined (undefined). JSON path: `facets`

      Facets allow to aggregate information collected on a particular result set.

        * `TermFacet`. Gives number of occurrences of the most recurring terms in all rows. JSON paths:
          * `field` (`string`): required name of the field
          * `size` (`uint32`) = undefined: optional number of facets/categories returned

        * `NumericRangeFacet`. Categorizes rows into numerical ranges (or buckets). JSON paths:
          * `field` (`string`): required name of the field
          * `size` (`uint32`) = undefined: optional number of facets/categories returned
          * `numeric_ranges` (`array`): non empty list of the ranges defined as following. The SDK might expose it as single function with
            three arguments, or define class NumericRange that will encapsulate single range, and then pass it to the builder.
            * `name` (`string`): required name of the range
            * `min` (`float` or `uint32`) = undefined: optional lower bound of the range
            * `max` (`float` or `uint32`) = undefined: optional upper bound of the range

        * `DateRangeFacet`. Categorizes rows inside date ranges (or buckets). JSON paths:
          * `field` (`string`): required name of the field
          * `size` (`uint32`) = undefined: optional number of facets/categories returned
          * `date_ranges` (`array`): non empty list of the ranges defined as following
            * `name` (`string`): required name of the range
            * `start` (`Date`): (encoded as a string to be parsed by golang on the server side) = undefined: optional lower bound of the
              range
            * `end` (`string`) = undefined: optional upper bound of the range

    * `timeout` (`Duration`) = `$Cluster::searchOperationTimeout`. JSON path: `ctl.timeout`

      Specifies how long to allow the operation to continue running before it is cancelled (default value defined in
      [0059-sdk3-foundation](0059-sdk3-foundation.md)). Also sets server-side timeout in milliseconds.

    * `serializer` (`JsonSerializer`) = `$Cluster::Serializer`.

      Specifies the serializer which should be used for deserialization of the fields member which is read from the
      `SearchRow`. `JsonSerializer` interface is defined in
      [0055-sdk3-transcoders-and-serializers](0055-sdk3-transcoders-and-serializers.md).

    * `raw` (`string key, string value`)

      This is an escape hatch to support unknown commands and be forward compatible in future.

    * `includeLocations` (`bool`) = `false`. JSON path: `includeLocations`

      The boolean `includeLocations` field triggers inclusion of search row locations.

### Returns

A `ISearchResult` object with the results of the query or error message if the query failed on the server.

### Throws

* Documented
  * `RequestTimeoutException` (#1)
  * `CouchbaseException` (#0)

* Undocumented
  * `RequestCanceledException` (#2)
  * `InvalidArgumentException` (#3)
  * `ServiceNotAvailableException` (#4)
  * `InternalServerException` (#5)
  * `AuthenticationException` (#6)

## IScope::SearchQuery

The API and implementation are identical to `ICluster::SearchQuery`, except it uses a different endpoint internally.

The user provides `scope.searchQuery("indexName", query, [options])` (rather than `scope.searchQuery("bucket.scope.indexName", query, [options])`).

The SDK will use endpoint `/api/bucket/{bucketName}/scope/{scopeName}/index/{indexName}/query` for the query.
This will execute the query against a scoped FTS index rather than a global index.
See the description of `ScopeQueryIndexManager` in [SDK RFC 54](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0054-sdk3-management-apis.md) for more details of scoped indexes.
All information from there applies here.

## SearchQuery implementations

### MatchQuery


A match query analyzes the input text and uses that analyzed text to query the index. An attempt is made to use the same analyzer that was
used when the field was indexed.

JSON paths:
* `match` (`string`): the input string to be matched against. The match string is required.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched. This can also affect the used analyzer
  if one isn't specified explicitly.

* `analyzer` (`string`) = undefined: analyzers are used to transform input text into a stream of tokens for indexing. The Server comes with
  built-in analyzers and the users can create their own. The string here is the name of the analyzer used.

* `prefix_length` (`uint32`) = undefined: require that the term also have the same prefix of the specified length (must be positive).

* `fuzziness` (`uint32`) = undefined: perform fuzzy matching. If the fuzziness parameter is set to a non-zero integer the analyzed text will
  be matched with the specified level of fuzziness.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

* `operator` (`MatchOperator`) = undefined: Defines how the individual match terms should be logically concatenated.
    * `MatchOperator` is an enum containing
      * OR (json:`"or"`) = undefined. Individual match terms are concatenated with a logical OR - this is the default if not provided.
      * AND (json: `"and"`) = undefined. Individual match terms are concatenated with a logical AND.
### MatchPhraseQuery

The input text is analyzed and a phrase query is built with the terms resulting from the analysis. This type of query searches for terms
occurring in the specified positions and offsets. This depends on term vectors, which are consulted to determine phrase distance.

JSON paths:
* `match_phrase` (`string`): the input phrase to be matched against. The match phrase string is required.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched. This can also affect the used analyzer
  if one isn't specified explicitly.

* `analyzer` (`string`) = undefined: analyzers are used to transform input text into a stream of tokens for indexing. The Server comes with
  built-in analyzers and the users can create their own. The string here is the name of the analyzer used.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### RegexpQuery

Finds documents containing terms that match the specified regular expression.

JSON paths:
* `regexp` (`string`): the regular expression to be analyzed and used. The regexp string is required.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### QueryStringQuery

The query string query allows humans to describe complex queries using a simple syntax.

JSON paths:
* `query` (`string`): The query string to be analyzed and used against. The query string is required.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### WildcardQuery

A wildcard query is a query in which term the character `*` will match `0..n` occurrences of any characters and `?` will match `1`
occurrence of any character.

JSON paths:
* `wildcard` (`string`): the wildcard-containing term to be analyzed and searched. The wildcard string is required.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### DocIdQuery

A docId query is a query that directly matches the documents whose ID have been provided. It can be combined within a `ConjunctionQuery` to
restrict matches on the set of documents.

JSON paths:
* `ids` (`array<string>`): The list of document IDs to be restricted against. At least one ID is required at execution time, but a
  `DocIdQuery` should be constructible without one to allow progressive programmatic population of the ID list.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### BooleanFieldQuery

Allow to match `true`/`false` in a field mapped as boolean.

JSON paths:
* `bool` (`true` or `false`): The boolean value to be looked for, required.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched. It should generally be encouraged to use the field, which must be a boolean field.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### DateRangeQuery

The date range query finds documents containing a date value in the specified field within the specified range. Either start or end can be
omitted, but not both.

JSON paths:
* `start` (`string`) = unset (unset): The lower end of the range.

* `inclusive_start` (`bool`) = unset (`true`): whether to include lower end of the range.

* `end` (`string`) = unset (unset): The higher end of the range.

* `inclusive_end` (`bool`) = unset (`true`): whether to include higher end of the range.

* `datetime_parser` (`string`) = unset: enable customer date time parser.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).


### NumericRangeQuery

The numeric range query finds documents containing a numeric value in the specified field within the specified range. Either min or max can
be omitted, but not both.

JSON paths:
* `min` (`float` or `uint32`) = unset (unset): The lower end of the range.

* `inclusive_min` (`bool`) = unset (`true`): whether to include lower end of the range.

* `max` (`float` or `uint32`) = unset (unset): The higher end of the range.

* `inclusive_max` (`bool`) = unset (`true`): whether to include higher end of the range.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### TermRangeQuery

The term range query finds documents containing a string value in the specified field within the specified range. Either min or max can be
omitted, but not both.

JSON paths:
* `min` (`string`) = unset (unset): The lower end of the range.

* `inclusive_min` (`bool`) = unset (`true`): whether to include lower end of the range.

* `max` (`string`) = unset (unset): The higher end of the range.

* `inclusive_max` (`bool`) = unset (`true`): whether to include higher end of the range.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### GeoDistanceQuery

This query finds all matches from a given location (`geopoint`) within the given distance. Both the point and the distance are required.

JSON paths:
* `location` (`float32[2]`): The location represents a point from which the distance is measured.

  The server accepts many formats for geo points, but the most terse one that should be used is as a JSON array where the first element is
  the longitude and the second element the latitude both encoded as JSON numbers:

      {"location": [lon, lat]}

* `distance` (`string`): The distance describes how far from the location the radius should be matched. For example, `"11km"`,
  `"11kilometers"`, `"3nm"`, `"3nauticalmiles"`, `"17mi"`, `"17miles"`, `"19m"`, `"19meters"`.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### GeoBoundingBoxQuery

This query finds all `geopoint` indexed matches within a given box (identified by the upper left and lower right corner coordinates). Both
coordinate points are required so the server can identify the right bounding box.

JSON paths:
* `top_left` (`float32[2]`): The top left coordinates signify the bounding box area and are required (`[lon, lat]`).

* `bottom_right` (`float32[2]`): The bottom right coordinates signify the bounding box area and are required (`[lon, lat]`).

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### ConjunctionQuery

The conjunction query is a compound query. The result documents must satisfy all of the child queries. It is possible to recursively nest
compound queries.

At execution, a conjunction query that has no child queries should fail fast (eg. throw an exception before a request is sent to the
server).

When idiomatic to the language, variable-length argument or lists of queries can also be accepted for each method (allowing to add several
queries to the "AND" in one go).

JSON paths:
* `conjuncts` (`array`) = undefined: list of the `SearchQuery` implementations.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### DisjunctionQuery

The disjunction query is a compound query. The result documents must satisfy a configurable minimal (`min`) number of child queries. By
default this `min` is set to 1.

At execution, a disjunction query that has fewer child queries than the configured minimum should fail fast (eg. throw an exception before a
request is sent to the server). Similarly, a disjunction query without child queries should also fail fast.

When idiomatic to the language, variable-length argument or lists of queries can also be accepted for each method (allowing to add several
queries to the "OR" in one go).

JSON paths:
* `disjuncts` (`array`) = undefined: list of the `SearchQuery` implementations.

* `min` (`uint32`) = 1: The minimum number of child queries that must be satisfied for the disjunction query.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### BooleanQuery

The boolean query is a useful combination of conjunction and disjunction queries. A boolean query takes three lists of queries:

* **must** - result documents must satisfy all of these queries.
* **should** - result documents should satisfy these queries.
* **must not** - result documents must not satisfy any of these queries.

At execution, a boolean query that has no child queries in any 3 categories should fail fast (eg. throw an exception before a request is
sent to the server).

The inner representation of child queries in the must/must_not/should sections are respectively a ConjunctionQuery and two DisjunctionQuery.

JSON paths:
* `must` (`ConjunctionQuery`) = undefined: a `ConjunctionQuery` object of the **must queries

* `should` (`DisjunctionQuery`) = undefined: a `DisjunctionQuery` object of the **should** queries

* `should.min` (`uint32`) = 1: If a hit satisfies at least `min` queries in the **should** section, its score will be boosted. By default,
  this is set to 1. This can be omitted if the underlying `DisjunctionQuery` is directly exposed and editable, or if a direct mean of
  setting the payload `should.min` is available (for instance in a language like Python).

* `must_not` (`DisjunctionQuery`) = undefined: a `DisjunctionQuery` object of the **must not** queries

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### TermQuery

A query that looks for **exact** matches of the term in the index (no analyzer, no stemming). Useful to check what the actual content of the
index is. It can also apply fuzziness on the term. Usual better alternative is `MatchQuery`.

JSON paths:
* `term` (`string`): The mandatory term is the exact string that will be searched into the index. Note that the index can (and usually will)
  contain terms that are derived from the text in documents, as analyzers can apply process like stemming. For example, indexing
  "programming" could store "program" in the index. As a term query doesn't apply the analyzers, one would need to look for "program" to
  have a match on that index entry.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `fuzziness` (`uint32`) = undefined: perform fuzzy matching. If the fuzziness parameter is set to a non-zero integer the analyzed text will
  be matched with the specified level of fuzziness.

* `prefix_length` (`uint32`) = undefined: require that the term also have the same prefix of the specified length (must be positive). The
  prefix length only makes sense when `fuzziness` is enabled (see above). It allows to apply the fuzziness only on the part of the term that
  is after the `prefix_length` character mark.

  For example, with the term "something" and a prefix length of 4, only the "thing" part of the term will be fuzzy-searched, and hits must
  start with "some".

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).


### PrefixQuery

The prefix query finds documents containing terms that start with the provided prefix. Usual better alternative is `MatchQuery`.

JSON paths:
* `prefix` (`string`): The prefix to be analyzed and used against. The prefix string is required.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### PhraseQuery

A query that looks for **exact** match of several terms (in the exact order) in the index. The provided terms must exist in the correct
order, at the correct index offsets, in the specified field (as no analyzer are applied to the terms). Queried field must have been indexed
with inncludeTermVectors set to true. It is generally more useful in debugging scenarios, and the `MatchPhraseQuery` should usually be
preferred for real-world use cases.

JSON paths:
* `terms` (`array`): The mandatory list of terms that must exactly match in the index. Note that the index can (and usually will) contain
  terms that are derived from the text in documents, as analyzers can apply process like stemming. Array size > 1, entries must not be
  empty.

  For example, indexing "programming books" could store "program" and "book" in the index. As a phrase query doesn't apply the analyzers,
  one would need to look for "program book" to have a match on that indexed document.

* `field` (`string`) = undefined: if a field is specified, only terms in that field will be matched.

* `boost` (`float`) = undefined. The boost parameter is used to increase the relative weight of a clause (with a boost greater than 1) or
  decrease the relative weight (with a boost between 0 and 1).

### MatchAllQuery

A query that matches all indexed documents.

JSON paths:
* `match_all` = `null`: The JSON representation of a `MatchAllQuery` is simply a `"match_all": null` entry in the query JSON Object.

### MatchNoneQuery

A query that matches nothing.

JSON paths:
* `match_none` = `null`: The JSON representation of a `MatchNoneQuery` is simply a `"match_none": null` entry in the query JSON Object.

## Vector search
This is a feature being added to Couchbase Server 7.6 in the FTS service.

The feature will initially be released in a Private Preview, and all SDK additions related to it should be annotated with the platform equivalent of @Stability.Volatile as changes may well be required following user feedback. 

References:
* [Server design spec](https://docs.google.com/document/d/1Vx4PPcl2byy2F-lM6wRIUPLAsAFsVDusMj5nvrKA13g/edit#heading=h.9f1ja3elibc0) (the first page provides a good overview of the feature)
* [PRD](https://docs.google.com/document/d/1zrNtgqLDNOFVYoyO5SOAD5Jtpkm43SMIKDc3yN1UbsM/edit#heading=h.f5uv7ep78x87)
* [PRD appendices](https://docs.google.com/document/d/1Ht6V74xRPSTSGlttz9ORAZTcozGCCJpXAaopG-pal3g/edit#heading=h.wxy3ni7kqtfp)

Given a sample KV document:

```
{
  "cityName": "Tokyo",
  "cityDesc": "<A long travel guide description of Tokyo, mentioning its nightlife>,
  // A vector field encoding what's in the "city" field
  "cityVector": [0.0023234, -0.0465719, 0.0213213, ...],
  
  "otherDocumentFields": "..."
}
```

An example of the query payload we need to be able to produce:

```
{
  "//": "The `query` field contains any regular FTS query, and will be combined with the vector search(es).  Here we just want cities starting with 'S'.",
  "query": {
    "prefix": "S",
    "field": "cityName",
    "boost": 1.0
  },
  "//": "The `knn` field is new, optional, and contains the vector search(es).  In this example two queries are being sent.",
  "knn": [
    {
      "//": "The `vector`, `field` and `k` fields are all mandatory.",
      "//": "Here the 'cityVector' field in the user's documents contains a vector.  It could vectorise a long travel-guide description of the city.",
      "field": "cityVector",
      "//": "This field is a query such as 'city with great nightlife' converted into a vector.  In practice the `vector` field is much longer.",
      "vector": [
        -0.00810323283,
        0.0727998167,
        0.0211895034,
        -0.0254271757
      ],
      "//": "Find the 3 nearest neighbours (cities) to 'city with great nightlife'.",
      "k": 3,
      "//": "This is a more important consideration than the other vector query, so give it a higher score boost.",
      "boost": 0.7
    },
    {
      "field": "cityVector",
      "//": "This is another query such as 'city with good museums'.",
      "vector": [
        -0.005610323283,
        0.023427998167,
        0.0132511895034,
        0.03466271757
      ],
      "k": 2,
      "boost": 0.3
    }
  ],
  "//": "The `knn_operator` field is new and determines how multiple vector searches are combined.",
  "knn_operator": "or",
  "//": "The `fields`, `sort` and `limit` fields come from `SearchOptions`, as usual.",
  "fields": [
    "citName",
    "cityDesc"
  ],
  "sort": [
    "-_score"
  ],
  "limit": 5
}
```

Important SDK considerations:

* There is some uncertainty on whether it is mandatory to provide a `query` field containing a 'normal' FTS query.  This is being followed up with the FTS team.  As it makes sense to be able to perform solely a vector search, we will assume for now that will be the case.
* The `knn` field can contain multiple vector queries.  It must always contain at least 1.
  * How they are combined is handled with a top-level `knn_operator` parameter.
* There may be other vector search types in future, such as Approximate Nearest Neighbour (ANN).  So we may want to aim for a level of abstraction and avoid explicitly mentioning `knn` and perhaps `k`.
* Inside an individual vector query:
  * It contains a single vector.  FAISS allows multiple vectors to be sent in one query and the FTS team say that may want to be exposed later.
  * The `field` parameter is essentially mandatory.  While FTS will not reject a query that does not contain it, it prevents that vector search from having any effect. 
  * The `k` parameter is mandatory.  The FTS team suggest 3 is a sensible default to send from the SDKs.
* There are new FTS index definition fields for defining the vector index, but they are inside the `plan` top-level field.  
So there is nothing extra required from the SDKs to support these, as we already simply turn the untyped `plan` parameters into JSON.
* The SDK (and Couchbase in general), at least in this first phase, have nothing to do with generating the vectors - both those stored in the documents, and those used for queries.
The user will be responsible for generating these, for example using the OpenAI Embeddings API.
* The feature works with both global and scoped FTS indexes.  As this is a 7.6+ feature, we will make scoped indexes the primary target for documentation, examples, etc.

SDK PM advises that the most common vector search use-cases are likely to be, in this order:

* Performing a vector search in isolation (without a traditional FTS query).
* A SQL++ statement is used to also perform a vector search.
  * This is just passed in the string statement and is expected to return the same query results JSON as at present, so from the SDK POV it's just a passthrough with no SDK modifications needed.
* A vector search combined with a traditional FTS query.

## API

The FTS JSON request has fundamentally changed, as it now allows traditional FTS queries and/or vector queries.
In addition, we need to consider that it may allow other query types in the future.

Such a fundamental conceptual change necessitates a similarly fundamental change at the SDK level.

Let us name the concepts we are working with:

* The top-level search request.  This is currently an unnamed concept in the SDKs - let us term it a `SearchRequest`.
* The traditional FTS query, which is already named `SearchQuery` in the SDKs.
Includes the query itself and any options that apply only to the FTS query such as "boost".
It's mapped to the top-level `query` JSON field.
* An individual vector query, which we can call a `VectorQuery`.  These map to the members of the new top-level `knn` array.
It can contain options applicable just to that vector query such as "boost" and "k".
* A way to perform multiple `VectorQuery`s, which we can call a `VectorSearch`.
Maps to the top-level `knn` array.
* Any options applicable to all forms of query (both the `SearchQuery` and the `VectorSearch`), which are in `SearchOptions` (skip, limit, explain, highlight, fields, etc.)  We can continue to use the existing `SearchOptions` object for this.
* Any options applicable only to the `VectorSearch`, which we can call `VectorSearchOptions`.

So the user will be performing a `SearchRequest` containing a `SearchQuery` and/or a `VectorSearch` and/or a `FutureSearchType`, optionally with some `SearchOptions` applying to all of those.  
A `VectorSearch` comprises 1+ `VectorQuery`s and optionally some `VectorSearchOptions`.

Neither `VectorSearch` nor `VectorQuery` will extend the `SearchQuery` interface, which we will reserve for traditional FTS queries.

Hence, none of this is compatible with the existing search interface `cluster.searchQuery(SearchQuery, SearchOptions)`.

This takes us to a new API, `cluster.search(String indexName, SearchRequest, SearchOptions)`, which has the flexibility to be able to perform traditional FTS and/or vector queries, and/or any future top-level queries.

The current `cluster.searchQuery()` API will not be deprecated at this time, given the volatility of this new feature.
But the intent is that the new `cluster.search()` will ultimately be how new users perform any sort of FTS service query, including just a standard FTS `SearchQuery` without vector search.

### API Examples
All examples will use the reference Java SDK implementation.

1. A traditional FTS query without vector search.  With the existing API:

```java
SearchQuery query = SearchQuery.matchAll();

scope.searchQuery("search_index_name", query);
cluster.searchQuery("search_index_name", query);
```

and with the new API:

```java
SearchRequest request = SearchRequest.searchQuery(SearchQuery.matchAll());

cluster.search("search_index_name", request);
scope.search("search_index_name", request);
```

2. A vector search with a single vector query and no traditional FTS query:

```java
// Using an imaginary API provided by OpenAI to generate the vector.
float[] vector = OpenAI.createVectorFor("some query");

SearchRequest request = SearchRequest
        .vectorSearch(VectorSearch.create(VectorQuery.create("vector_field", vector)));

scope.search("search_index_name", request);
cluster.search("search_index_name", request);
```

3. A combined search with both vector search and a traditional FTS query:

```java
float[] vector = OpenAI.createVectorFor("some query");

SearchRequest request = SearchRequest
        .searchQuery(SearchQuery.matchAll())
        .vectorSearch(VectorSearch.create(VectorQuery.create("vector_field", vector)));

scope.search("search_index_name", request);
cluster.search("search_index_name", request);
```

4. A more complex example showing a vector search containing multiple vector queries, setting all parameters:

```java
// Sending multiple `VectorQuery`s, and setting all possible parameters.
SearchRequest request = SearchRequest
        .vectorSearch(VectorSearch.create(List.of(
                VectorQuery.create("vector_field", aVector).numCandidates(2).boost(0.3),
                VectorQuery.create("vector_field", anotherVector).numCandidates(1).boost(0.7)),
            vectorSearchOptions().vectorQueryCombination(VectorQueryCombination.AND));

scope.search("search_index_name", request);
cluster.search("search_index_name", request);
```

### VectorSearch
`VectorSearch` construction is platform-idiomatic:

`VectorSearch.create(List<VectorQuery> vectorQueries, [VectorSearchOptions options])`

The first parameter needs to be mandatory.

`VectorSearch.create(VectorQuery vectorQuery)` can be added to make it easier to send a single `VectorQuery`.
This is left optional to better support SDKs without overloads.
The SDK is free to have this overload not take a `VectorSearchOptions`, since the only field currently in there applies to multiple vector queries.
It is left platform-idiomatic as different platforms have different abilities regarding adding overloads later.

`VectorSearchOptions` will currently support just one option:

* `vectorQueryCombination` (`VectorQueryCombination`).  Will be sent in the top-level JSON payload as `knn_operator` (string) as either `"and"` or `"or"`.  We name the field differently in the SDK to a) improve readability and b) be able to better support possible future forms of vector search such as ANN.  If not set by the user, no value is sent.  It controls how elements in the `knn` array are combined.

```
enum VectorQueryCombination {
    AND,
    OR
}
```

`VectorSearch` does not extend the `SearchQuery` interface, which is now reserved for traditional FTS queries.

### VectorQuery
`VectorQuery` creation is platform-idiomatic:

`VectorQuery.create(String vectorFieldName, float[] vector)`

Both parameters need to be mandatory.  They are sent in the JSON as "field" (string) and "vector" (number array) fields respectively. 

The current size limit for the vector is 2048 elements, but this will not be enforced on the SDK side.
The vector must be non-empty though, and if not the SDK will raise `InvalidArgument`. 

It will also support the following parameters, exposed in the same way as the SDK exposes traditional FTS query parameters.
In some SDKs this is as fluent-style methods on `VectorQuery` itself, but a `VectorQueryOptions` block as an optional parameter on `Vector.create()` would be more SDK3-idiomatic.
The SDK should follow its existing convention for FTS parameters.

* `uint32 numCandidates`.  Sent as a `k` number field in the JSON.  We name the field differently in the SDK for the same reason as `knn_operator`.  If not set by the user, FTS PM requests that the SDK send a default value of 3.  It controls how many results are returned and must be >= 1.  If < 1 the SDK will raise `InvalidArgument`.
* `float boost`.  Sent as a `boost` number field in the JSON.  If not set, the field is not sent.

`VectorQuery` does not extend the `SearchQuery` interface, which is now reserved for traditional FTS queries.

### SearchRequest
`SearchRequest` creation is platform-idiomatic.  There need to be two contruction options, allowing it to be created from either a `VectorSearch` or a `SearchQuery`.

```
SearchRequest.vectorSearch(VectorSearch vectorSearch)
SearchRequest.searchQuery(SearchQuery searchQuery)
```

It will also support the following fluent-style methods, allowing one (and only one) `SearchQuery` or `VectorSearch` to be added:

* `searchQuery(SearchQuery searchQuery)`.  Note that this intentionally does not take a SearchOptions - that can only be specified during construction.  If the user has already specified a `SearchQuery`, either at construction time or via another call to `SearchRequest.searchQuery()`, the SDK needs to raise an `InvalidArgumentException`.
* `vectorSearch(VectorSearch vectorSearch)`.  Note that this intentionally does not take a SearchOptions - that can only be specified during construction.  If the user has already specified a `VectorSearch`, either at construction time or via another call to `SearchRequest.vectorSearch()`, the SDK needs to raise an `InvalidArgumentException`.

These fluent-style methods are proposed because, at least in some SDKs, FTS follows a fluent style.
If this is completely unidiomatic to an SDK, this alternative single constructor can be considered:

```
SearchRequest.searchRequest([SearchQuery searchQuery], [VectorSearch vectorSearch], [SearchOptions searchOptions])
```

In this case, at least one of `SearchQuery` and `VectorSearch` must be provided and the SDK must raise `InvalidArgumentException` if not.
Note that this alternative syntax is not hugely desirable, as it does not well support future search features.

`SearchRequest` does not extend the `SearchQuery` interface, which is now reserved for traditional FTS queries.

### showrequest
The SDK will now send `"showrequest": false` in the top-level JSON, for all queries (vector or not) sent via the new `cluster/scope.search()` interface.
Queries sent from the original `cluster/scope.searchQuery()` interface are unaffected.
Older server versions will silently ignore this parameter.

This prevents the server from returning the original request in the response - a request that can be substantial when large vectors are used, and that is not exposed in the SDK.

Note: this `showrequest` approach is being debated by the FTS team and may go through further revision.

## Return Types 

### SearchResult

The `SearchResult` interface provides a means of mapping the results of a Search query into an object.

```
interface SearchResult {
    Stream<SearchRow> Rows();
    Map<String, SearchFacetResult> Facets();
    SearchMetaData MetaData();
}
```

### SearchRow

A single entry of search results. The server calls them "hits", and represents as a JSON object. The following interface describes the
contents of the result row.

```
interface SearchRow {
    String Index();
    String Id();
    double Score();
    JsonObject Explanation();
    Optional<SearchRowLocations> Locations();
    Optional<Map<String, String[]>> Fragments();
    Optional<T> Fields<T>();
}
```

### SearchRowLocations

```
interface SearchRowLocations {
    // list all locations (any field, any term)
    List<SearchRowLocation> GetAll();

    // list all locations for a given field (any term)
    List<SearchRowLocation> Get(String field);

    // list all locations for a given field and term
    List<SearchRowLocation> Get(String field, String term);

    // list the fields in this location
    List<String> Fields();

    // list all terms in this locations, considering all fields (so a set)
    Set<String> Terms();

    // list the terms for a given field
    List<String> TermsFor(String field);
}
```

### SearchRowLocation

```
interface SearchRowLocation {
    String Field();
    String Term();
    uint32 Position;
    uint32 Start;
    uint32 End;
    Optional<uint32[]> ArrayPositions();
}
```

### SearchFacetResult

An individual facet result has both metadata and details, as each facet can define ranges into which results are categorized.

```
interface SearchFacetResult {
    String Name();
    String Field();
    uint64 Total();
    uint64 Missing();
    uint64 Other();
}
```

### SearchMetaData

Represents the meta-data returned along with a search query result.


```
interface SearchMetaData {
    SearchMetrics Metrics();
    Map<String, String> Errors();
}
```

Sample request/response payloads: [sdk-testcases/search](https://github.com/couchbaselabs/sdk-testcases/tree/master/search)

If top-level `"error"` property exists, then SDK should build and throw `CouchbaseException` with its content.

### SearchMetrics

```
interface SearchMetrics {
    Duration Took();
    uint64 TotalRows();
    double MaxScore();
    uint64 TotalPartitionCount(); // SuccessPartitionCount + ErrorPartitionCount
    uint64 SuccessPartitionCount();
    uint64 ErrorPartitionCount();
}
```

# Changelog

* Sept 12, 2019 - Revision #1
    * Initial Draft

* Oct 01, 2019 - Revision #2
    * Extracted options from old `SearchQuery` class
    * Extracted status and metrics into separate interfaces

* Oct 24, 2019 - Revision #3
    * Turned `SearchStatus.Errors` into `Map<String, String>`
    * Locations, Fields and Fragments are exposed as `Optional` now
    * Added note that facet ranges query builder API might be either function with three arguments or with one argument of class `Range`,
      that encapsulates `name`, `min` and `max`

* Oct 31, 2019 - Revision #4
    * added enum for distance units in sorting
    * added link to payloads with various status shapes

* Nov 7, 2019 - Revision #5
    * Expanded `SearchStatus` and embedded it into `SearchMetaData` and `SearchMetrics`
    * Added note about hard failures that must be thrown as exceptions.

* Jan 10, 2020 - Revision #6 (by Sergey Avseyev)
    * Added `SearchOptions#raw()`

* Feb 6, 2020 - Revision #7 (by Brett Lawson)
    * Updated `SearchRowLocation` `Pos` field to `Position` in order to be consistent with RFC guidance which indicates to use full names
      not abbreviations.

* April 30, 2020
    * Moved RFC to ACCEPTED state.

* August 27, 2020
    * Converted to Markdown.

* December 7, 2021 - Revision #8 (by Charles Dixon)
    * Added `includeLocations` to `SearchOptions`.
    * Added `MatchOperator` and added `operator` to `MatchQuery`.

* March 6, 2023 - Revision #9 (by Graham Pople)
    * Added `scope.searchQuery()`.

* December 18, 2023 - Revision #10 (by Graham Pople)
    * Added vector search.

# Signoff

| Language   | Team Member         | Signoff Date   | Revision |
|------------|---------------------|----------------|----------|
| Node.js    | Brett Lawson        | April 16, 2020 | #7       |
| Go         | Charles Dixon       | April 22, 2020 | #7       |
| Connectors | David Nault         | April 29, 2020 | #7       |
| PHP        | Sergey Avseyev      | April 22, 2020 | #7       |
| Python     | Ellis Breen         | April 29, 2020 | #7       |
| Scala      | Graham Pople        | April 30, 2020 | #7       |
| .NET       | Jeffry Morris       | April 22, 2020 | #7       |
| Java       | Michael Nitschinger | April 16, 2020 | #7       |
| C          | Sergey Avseyev      | April 22, 2020 | #7       |
| Ruby       | Sergey Avseyev      | April 22, 2020 | #7       |
