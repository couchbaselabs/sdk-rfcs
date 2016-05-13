# Meta

 - RFC Name: AT_PLUS Consistency for N1QL Queries
 - RFC ID: 0004-at_plus
 - Start Date: 2015-12-16
 - Owner: Michael Nitschinger (@daschl)
 - Current Status: Review

# Summary
This RFC specifies the user API and interaction with the server for enhanced
RYOW query performance (practically known as `AT_PLUS` with "Mutation Tokens").

# Motivation
The current approach to perform RYOW ("Read Your Own Writes") queries is to use the
`REQUEST_PLUS` scan consistency when executing a N1QL query. While this is
semantically correct, it takes the performance hit of having to pick the absolute
latest sequence number in the system and as a result potentially waiting for
a longer time until the indexer has caught up under high mutation rates.

One way to mitigate this is to only wait until specific mutations have
been indexed instead of waiting for all of them up to the latest one. Also, the
likelihood of the indexer already having indexed the mutation is directly
proportional to the time delta between mutation and subsequent query. This also
means that a performance win will be marginal if the `AT_PLUS` query immediately
follows the mutations, in this case the programmatic and runtime overhead of using
the tokens might not be worth the tradeoff.

`AT_PLUS` is treated as a pure optimization over `REQUEST_PLUS` and has no semantic
difference to it in terms of correctness inside the bounds specified through the
mutation tokens. One cannot use `REQUEST_PLUS` and `AT_PLUS` together in the same
query. Note that as described further below, the main tradeoff here is increased
network utilization because the client needs to send more information to the server
as part of the query request.

# General Design
In the big picture of `AT_PLUS` for the developer, there are two components that
must be considered:

 1. Mutations are executed on a bucket. They might or might not include
    `MutationTokens` as part of the response, depending on the SDK configuration
    (at this point, enabling tokens is optional since it leads to larger
      response bodies but is mandatory for this RFC to work).
 2. The application state after the mutation happened needs to be applied at
    N1QL query time.

This RFC does not focus on 1. since it has been implemented already as part of
the enhanced durability requirements initiative in the 4.0 timeframe. This RFC
assumes that the Mutation Token is available as a return value from
a mutation operation, most likely exposed on the `Document` or equivalent or
stored internally (see implementation details). `AT_PLUS` is not possible without
`MutationTokens`.

This RFC proposes an extension to the current `MutationToken` to also
include the bucket name. This is important so that the
token can be properly namespaced at query time to remove ambiguity in
cross-bucket query scenarios and when transported between applications.

Since it is important to this RFC, the properties of a `MutationToken` are as
follows:

 1. A `MutationToken` is opaque to the user and needs to be treated as such.
 2. A `MutationToken` is immutable after creation.
 3. A `MutatonToken`'s cannot be serialized/deserialized directly.
 4. Numerous `MutationToken`'s must be aggregated into a `MutationState` which is
    serializable.

The internal structure of a `MutationToken` looks like this:

```
MutationToken {
    [required, immutable] vbucket_id = long
    [required, immutable] vbucket_uuid = long
    [required, immutable] sequence_number = long
    [required, immutable] bucket_name = string
}
```

The `MutationState` exposes the following methods used to aggragate multiple `MututationTokens` into a single unit.  It can then be serialized/deserialized as well and passed to the `consistent_with` method for a N1Ql query.

```
MutationState {
  static from(docs: Document...) -> MutationState
  static from(fragments: DocumentFragment...) -> MutationState

  add(docs: Document...) -> MutationState
  add(fragments: DocumentFragment...) -> MutationState
  add(state: MutationState) -> MutationState
}
```

The following behavior is defined for the methods based on the contents of the passed MutationState:

 - When called with a list of `Documents`/`Fragments` and the `MutationTokens` are
   present on the `Document`, then they are extracted and used as part of the query.
 - Subsequent calls to either a previously captured `Document`/`Fragment`
   reference must override the old one if the new one has a higher sequence number.
   There must not be duplicates values for any particular bucket and vbid pair.

The N1QL query execution for AT_PLUS consists of the following user level API:

```
consistent_with(mutationState: MutationState)
```

Setting `SCAN_CONSISTENCY` to anything but `AT_PLUS` is a runtime
error (see errors). If the SDK already exposes `AT_PLUS` is must be explicitly set, if
`AT_PLUS` is not yet exposed it should be kept internal since there is no actual benefit
in exposing it. If so, the user only specifies the `consistent_with`
boundaries and the SDK needs to figure out the rest (see implementation).

## Internal implementation
The N1QL Query API specifies the following API to be used for `AT_PLUS`:

```
{"scan_vectors": {
    "bucket1_name": {
      "<vbucket_id>": [<sequence_number>, "<vbucket_uuid>"],
      "<vbucket_id>": [<sequence_number>, "<vbucket_uuid>"]
    },
    "bucket2_name": {
      "<vbucket_id>": [<sequence_number>, "<vbucket_uuid>"]
    }
  },
  "scan_consistency": "at_plus",
  // other optional query options...
}
```

The `scan_vectors` object contains one entry for each bucket that takes part in this query.
Inside the bucket scope all `MutationTokens` that need to be part of the index boundary are put in.
**Be careful** that the `vbucket_uuid` and `vbucket_id` are strings while the `sequence_number` is a JSON number.

Once the query is assembled, the Mutation Tokens are marshaled into their JSON
representation (through the `MutationState`) and passed alongside all the other params to the N1QL query
engine.

The following edge cases need to be considered in the implementation:

 - `Document`/`Fragment`s that are passed in without a token set must fail at runtime immediately.
 - If the user passes in more than one `MutationToken` for a unique vbucket ID, the SDK needs to automatically pick the one with the higher sequence number, making sure that the "lower boundary" is satisfied automatically. This issue can come up in the case where multiple mutation tokens on the same ID are loaded and passed in subsequently to the same query.


## Serialization and Deserialization of MutationState
In order to cover the use case where `MutationToken`s need to be shared across applications, the tokens need to be converted into a common format.

This is done at the `MutationState` level (aggregated MutationToken's). How the marshalling is done depends on the language, but the wire format is defined in this RFC. This format is identical to the expected wire-format as received by the N1QL service as state above in this RFC.

As an example, if the following two `MutationTokens` are stored in the `MutationState` and it is exported:

 - MutationToken(bucket=default, vbid=1, seqno=1, vbuuid=1234)
 - MutationToken(bucket=beer-sample, vbid=25, seqno=10, vbuuid=5678)

```json
{
  "default": {
    "1": [1, "1234"]
  },
  "beer-sample": {
    "25": [10, "5678"]
  }
}
```

This ensures that a `MutationState` can be loaded from this exact JSON back into an object and then used as part of an `AT_PLUS` query.

## Errors
The following user errors need to be covered by the SDKs:

| Error | Behavior |
| ----- | -------- |
| Other `SCAN_CONSISTENCY` than `AT_PLUS` specified in addition to `consistent_with` | IllegalArgumentException or equivalent thrown at runtime |
| `Document`/`Fragment` passed in with no mutation token set | IllegalArgumentException or equivalent thrown at runtime |
| `consistent_with` is used but mutation tokens are not enabled on the config | IllegalArgumentException or equivalent thrown at runtime |

# Language Specifics

## Setting Scan Consistency with Document

 - **Generic:** `consistent_with(mutationState: MutationState)`
 - **C:**

    ```c
    lcb_n1p_setconsistent_handle(lcb_N1QLPARAMS*, const lcb_t);
    lcb_n1p_setconsistent_token(lcb_N1QLPARAMS*, const lcb_MUTATION_TOKEN*);
    token = lcb_resp_get_muation_token(int cbtype, const lcb_RESPBASE* resp);
    ```
 - **Go:** `func (nq *N1qlQuery) ConsistentWith(state *MutationState) *N1qlQuery`
 - **Java:** `N1qlParams consistentWith(MutationState mutationState)`
 - **.NET:** `ConsistentWith(MutationState.From(document1, document2, document3));`
 - **NodeJS:** ``
 - **Python:**

    ```python
    ms = MutationState()
    ms.add_results(*results)  # A `Result` is the Python equivalent of a Document
    q.consistent_with(ms)
    ```
 - **PHP:**

    ```php
    # $source is either CouchbaseMetaDoc or CouchbaseDocumentFragment
    MutationState::from($source, ...) #-> MutationState

    $ms = new MutationState()
    # $source is either CouchbaseMetaDoc or CouchbaseDocumentFragment,
    # or another MutationState
    $ms.add($source, ...) #-> MutationState
    $q.consistent_with($ms)
    ```
 - **Ruby:** *not included*

# Unresolved Questions
None.

# Changelog
 - 2016-05-03: Clarified Serialization wire format with example, polishing and rewording. Weakened the constraint on the enum so that .NET can keep it's AT_PLUS enum variant exposed.
 - 2016-03-25: Moved into REVIEW state as discussed (michael n.)

# Signoff

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| C        | Mark N.| May 3rd, 2016 |
| Go       | Brett L.|  May 3rd, 2016 |
| Java     | Michael N.| May 3rd, 2016 |
| .NET     | Jeff M.| May 3rd, 2016|
| NodeJS   | Brett L.| May 3rd, 2016 |
| Python   | Mark N.| May 3rd, 2016 |
| PHP      | Sergey A. | May 13th, 2016 |
