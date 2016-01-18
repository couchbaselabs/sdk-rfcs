# Meta

 - RFC Name: SubDocument API for clients
 - RFC ID: 0002
 - Start Date: 2015-11-11
 - Owner: Mark Nunberg
 - Current Status: Draft

# Summary

This RFC proposes a client-side API for the Sub-Document operations which will
be released in a future server version.

# Motivation

Expose the high performance subdocument API to developers

# General Design

**PLEASE READ THIS FIRST**
https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#

The link above contains the full commands specification at the behavior and
protocol layer. The specification is considered authoritative, and this RFC
merely adapts the SDKs to the aforementioned specification.

## Terminology

* Document: A document or item in the Memcached sense.
* Path: A specific location in a JSON document. A path uses the path syntax
  described in the server-side API.
* Sub-Document: A section of a document, specified by the path.
* Top-level: Document-level operations, for example `upsert`.

## Overview

The Sub-Document API exposes CRUD-like methods which operate on paths _within_
documents. In these respects it is identical to CRUD methods except that it
treats each path as its own "sub-document" (hence the name).

Subdocument operations come in two flavors. One is the simple single-operation
form which takes a single document ID and a single path. Another is a
_multi-path_ form which retrieves or mutates _several_ paths within a document.

Single-path methods are slightly more efficient and require less processing
overhead than their multi-path equivalents (that is, even if you pass a single
path to a multipath method, some additional processing overhead may be involved).

Note that both 'multi-path' and 'single-path' APIs operate on the same
document, and correspond to a single binary operation. This should not be
confused with 'multi' operations in some SDKs which pipeline multiple binary
requests in the network.

## Objects

### Document/Item

This class is used in top-level KV operations. The `Document` object itself is the return value from normal KV operations
which contain the value, cas, mutation token, etc.

### DocumentFragment
The `Document` _may_ be replaced by a `DocumentFragment` to signal that the
value contained therein is only _part_ of the document (a fragment), and that top-level operations can't apply to a fragment.

This subclass contains the same fields as the `Document`, except the `value`
(or `contents`) is replaced by the `fragment` field.

It also additionally contains the `path` considered in subdoc operations.

Thus `DocumentFragment` can be used both as a return type (eg. when attempting to get a fragment of a document) and an input (for mutations, `DocumentFragment` contains everything needed: id of the document, path in the document, fragment value to be applied, cas and expiry...).
In some languages this can replace longer method signatures for mutations.

This object is used as an example, some SDKs may wish to either use the
'value' or 'document' field of the existing `Document` return value, or
subclass it, replacing value with 'fragment'.

## Path

From the perspective of the SDK, the path is a simple string used to specify
a location within the document. The path uses N1QL syntax to specify locations
within a document, thus a valid path may be:

    foo.bar.baz

or

    `foo`.`bar`.`baz`

or

    `foo.some_array[0]`

or

    `foo.some_array[-1]`

As in N1QL, special path characters may be escaped via a backtick (`` ` ``).

Some operations (notably, those taking a path to an array) may accept the
_empty path_ (i.e. an empty string)
which refers to the entire document. This is valid if the document is a bare
JSON array.

Each path component must be valid JSON (without the enclosing quotes). This
means that things like tabs, quotes, and backslashes must be escaped. To think
of it another way, the path matching algorithm in the server works via simple
string comparison, so a JSON key `"bs\\ and \t tabs are here"`` would be matched
by a path component of `"bs\\ and \t tabs are here"`.

The SDK MUST verify that a path is non-empty for operations which do not
support empty paths. All other forms of path verification is done on the server
(but an empty path for unsupported operations will return a memcached `EINVAL`
which isn't really helpful).

## Single-path Methods

Methods are named exactly like their top-level counterparts, and should also
exist at the top level as well.

Error codes are common to most operations.

### `get_in(docid, path)`
Gets a single path from the document. Returns a `Document`
(or `DocumentFragment`) containing the path value.

* `docid`: Document ID
* `path`: The sub-document path

### `exists_in(docid, path)`

Like `get_in`, but does not return the value (only checks if it exists)`. If the
item does not exist, the error code is suitably set in the result (or whatever
the appropriate error delivery mechanism is for the SDK).

### `upsert_in(docid, path, value, create_parents, cas, ttl, persist_to, replicate_to)`
Unconditionally places the specified value in the provided path.

* `docid`: Document ID
* `path`: the sub-document path
* `value`: the sub-document value. This should be anything JSON-serializable.
  JSON primitives such as true, false, null, strings and numbers are accepted
  too.
* `create_parents`: Boolean parameter indicating whether the full parent tree
  should be created. By default, only the _last_ path element is created if
  it does not exist.
* `cas`: Exactly the same meaning as it has in top-level CRUD operations.
* `ttl`: Expiration time. This has the same meaning as it does in the CRUD
  operations. The TTL can be set for _all_ mutation methods.
* `persist_to`, `replicate_to`: Durability requirements. Functions exactly as in CRUD

> **NOTE**: This (And the other sub-document APIs) will never create or remove documents
> on the server. This means that if the document does not exist, a `KEY_ENOENT`
> is returned from the server. The document is never created, even if `create_parents`
> is true

> **ALTERNATIVE SIGNATURES**: Having a `DocumentFragment` class with all the fields necessary for a mutation opens the option of using the `DocumentFragment` as an input parameter for all mutation methods in place of `docid`, `path`, `value`, `cas` and `ttl`.

### `insert_in(docid, path, value, create_parents, cas, ttl, persist_to, replicate_to)`
Exactly like `upsert_in`, except the path must not already exist.
If it does, `PathExists` is thrown.

### `replace_in(docid, path, value, cas, ttl, persist_to, replicate_to)`
Exactly like `upsert_in`, except the penultimate path must exist. If it
not exist, `PathNotFound` is thrown. Unlike `insert_in` and `upsert_in`,
`replace_in` can work on both dictionary and array values

### `extend_in(docid, path, value_or_values, direction, create_parents, cas, ttl, persist_to, replicate_to)`
Add an element to the back or front of an array.

* `value_or_values`: Can be a single JSON value, or a list of multiple
  JSON values. The list should be of a special type (MultiValue) so the
  SDK should not serialize the list itself as JSON.
* `direction`: `FRONT` or `BACK` depending on whether the item is to be
  added to the beginning or end of the array
* `create_parents`: Create the enclosing array and any intermediate elements.
  If this is not set, then the immediate array must exist.

### `arrayinsert_in(docid, path, value_or_values, cas, ttl, persist_to, replicate_to)`
Insert an element(s) at a given position in the array. The path should indicate
the desired position at which to insert the new element(s).

**FIXME:** This should really be `insert_in`, but we are already using that for
the object-based insert.

### `addunique_in(docid, path, value, create_parents, cas, ttl, persist_to, replicate_to)`
Adds a 'unique' element into an array (at which end is not defined or specified).
Functions exactly like `extend_in`. There are some server-side constraints into
which values can be used for unique operations - currently it is restricted to
JSON primitives (so no compound items).

* Throws `PathExists` if the unique element already exists
* Throws `CantInsertValue` if the value is not a JSON primitive.
* Throws `PathMismatch` if the array already contains non-primitive elements

> Because there is too much variation in semantics between `add_unique_in` and
> `extend_in`, they are separate methods rather than a single unified one.

### `counter_in(docid, path, delta, create_parents, cas, ttl, persist_to, replicate_to)`
Performs an atomic counter operation on a given path.

* `delta`: The numeric delta, this is simply added (`+`) to the existing value.
  This value may be signed to 'decrement' or unsigned to 'increment' the existing
  value.

See "Numeric Errors" below for a list of errors thrown by this operation.
This method also returns the current value as a `DocumentFragment`

### `remove_in(docid, path, cas, ttl, persist_to, replicate_to)`
Removes a path from the document

## Multi-Path methods

These methods take a document ID and a list of _specifications_ to perform. There are
only two methods (`mutate_in` and `lookup_in`).

The specification of each method is highly language specific, to use pseudo code, we will
pretend there are two static helper classes which generate appropriate specs:

### `LookupSpec`
Creates lookup specifications. There are two 'static methods'

* `LookupSpec.get(path)` (takes the same arguments as `get_in`
* `LookupSpec.exists(path)` (takes the same arguments as `exists_in`)

Both these objects should return a spec suitable for consumption by the `lookup_in`
method.

### `MutationSpec`
Creates mutation specifications. There are several 'static methods' which may be used.
These all function exactly like their single-path counterparts except that the document
ID and cas are not provided (these are passed directly into `mutate_in`).

* `MutationSpec.replace(path, value)`: See `replace_in`
* `MutationSpec.upsert(path, value, create_parents)`. See `upsert_in`
* `MutationSpec.insert(path, value, create_parents)`. See `insert_in`
* `MutationSpec.extend(path, value_or_values, direction, create_parents)`. See `extend_in`
* `MutationSpec.arrayinsert(path, value_or_values)`. See `arrayinsert_in`
* `MutationSpec.addunique(path, value)`. See `addunique_in`
* `MutationSpec.counter(path, delta)`. See `counter_in`
* `MutationSpec.remove(path)`. See `remove_in`

### `LookupResult`
This object represents a result for a single path when using `lookup_in`.
It contains at least two fields:

* `status`: The error code for the operation. If the path exists, this should
  contain an object or value which indicates "Success" (this can be a NULL or
  empty value to indicate no error)
* `value`: The value for the path. This is always empty when using `exists`.

Optionally and for developers convenience, implementations can also add two fields:

* `path`: the path that was looked up for this result.
* `operation`: a representation of the actual lookup operation that was attempted by the associated spec (can be the memcached opcode or preferably an enum or user-friendlier representation).

The implementation of this as a concrete type is _optional_. Python for example
simply exposes this as a tuple of `(error, value)`.

Implementing as a concrete type can open the possibility to offer convenience methods such as `exists()` or `valueOrThrow()` (see Java specifics).

### `lookup_in(docid, lookup_specs...)`

Retrieve (or check the existence of) multiple paths.

Since this operation can have mixed results, exceptions should not be raised (or
if they are, ensure they are extremely well documented).

This should return a `Document` containing a list of `LookupResultSpec` objects in
its  `value` field. If a multi-variant `value` field is not suitable, a separate
return type (for example, `MultiLookupResult`) can be offered for this purpose (not necessarily extending `Document`).

### `mutate_in(docid, cas, ttl, persist_to, replicate_to, mutation_specs...)`
Mutate paths in a document. Either all or none of the mutations will succeed.

Upon success, a success return value is provided (i.e. a new `Document` object, or a dedicated `MultiMutationResult` containing the updated cas and `MutationToken`).
Upon failure, the Memcached protocol will inform the client of the index of the
failure (i.e. the 0-base _nth_ spec which caused the error).
This may be provided to the user as part of the exception structure; via
a log message; or simply suppressed.

## Errors

In addition to the traditional document-level errors (which are received
when there is a problem with the document - for example, is missing or
has a different CAS), there are additional sub-document specific error codes.

### Generic Errors
* PathNotFound: Received when a path does not exist in the document. The exact meaning of
  path existence depends on the operation and inputs.
* PathExists: Received if a path already exists and it shouldnt't (for example, `replace_in`)
* PathMismatch: Received if the path structure conflicts with the document structure (for
  example, if a path mentions `foo.bar[0].baz`, but `foo.bar` is actually a JSON object.
* PathInvalid: Path has a syntax error, or path syntax is incorrect for operation (for
  example, if operation requires an array index)
* CantInsertValue: If the provided value cannot be inserted at the given path. This is
  _usually_ because the value is invalid JSON and should generally not be visible in
  SDKs which serialize their input beforehand.
* DocumentNotJson: Received if document itself is not JSON
* PathTooDeep: Path is too deep to parse. Depth of a path is determined by how many
  components (or levels) it contains. The current limitation is 32 levels, though
  that number is arbitrary. It is there to ensure a single parse does not consume
  too much memory (overloading the server).
  This error is similar to other `TooDeep` errors, which all relate to various
  validation stages to ensure the server does not consume too much memory when
  parsing a single document.
* DocumentTooDeep: Document is too deep to parse.
* ValueTooDeep: Proposed value would make the document too deep to parse

### Numeric Errors
These errors are only received for `counter_in` operations:

* NumberTooBig: Existing number value in document is too big (out of range between `INT64_MIN` and `INT64_MAX`
* ZeroDelta: Got a zero value for a delta
* BadNumber: Value provided as delta cannot be parsed as an integer. This error should
  usually not be seen with high level SDKs
* DeltaTooBig: The delta itself is too big, or the result of the operation would yield
  a number too big.


# Language Specifics
In this section, the abstract design parts need to be broken down on the SDK level by each maintainer and signed off eventually.

## C

The C SDK will not use these verbs, and will instead model the subdoc operations
after the existing `lcb_store3()` API - using `lcb_sdstore3`:

    // Single command
    lcb_CMDSDSTORE cmd = { 0 };
    LCB_CMD_SET_KEY(&cmd, "key", 3);
    LCB_CMD_SET_VALUE(&cmd, "\"value\"", strlen("\"value\""));
    LCB_SDCMD_SET_PATH(&cmd, "path", strlen("path"));
    cmd.mode = LCB_SUBDOC_DICT_UPSERT;
    lcb_sched_enter(instance);
    lcb_sdstore3(instance, NULL, &cmd);
    lcb_sched_leave(instance);

    // Multi Commands
    lcb_error_t rc = LCB_SUCCESS;
    lcb_CMDSDMULTI cmd = { 0 };
    LCB_CMD_SET_KEY(&cmd, "key", 3);
    cmd.multimode = LCB_SDMULTI_MODE_LOOKUP;
    lcb_SDMULTICTX *ctx = lcb_sdmultictx_new(instance, NULL, &cmd, &err);
    // Check error...
    lcb_CMDSDGET gcmd = { 0 };
    LCB_SDCMD_SET_PATH(&gcmd, "path1", strlen("path1"));
    err = lcb_sdmultictx_addcmd(ctx, LCB_SUBDOC_GET, (const lcb_SDCMDBASE*)&gcmd);
    LCB_SDCMD_SET_PATH(&gcmd, "path2", strlen("path2"));
    err = lcb_sdmultictx_addcmd(ctx, LCB_SUBDOC_EXISTS, (const lcb_SDCMDBASE*)&gcmd);
    lcb_sched_enter(instance);
    lcb_sdmultictx_done(ctx);
    lcb_sched_leave(instance);
    lcb_wait(instance);

## Python

    from subdocument import get, exists, upsert, counter

    cb.get_upsert_in('user:mnunberg', 'address', ['123 Main St', 'Reno', 'NV', 'USA'])

    result = cb.get_in('user:mnunberg', 'address[0]')
    result.value
    # '123 Main St'

    # Demonstrate lookup specs
    from couchbase.subdocument import get, exists
    results = cb.lookup_in('user:mnunberg',
                           get('email'),
                           get('address'),
                           exists('couchbase_id'))
    email_ok, email = results[0]
    addr_ok, addr = results[1]
    couchbase_exists, _ = results[2]

    # Demonstrate mutation specs
    from couchbase.subdocument import extend, replace, arrayinsert, addunique
    cb.mutate_in('user:mnunberg',
                 extend('messages', {'from': 'management', 'body': 'Hello!'}, direction=FRONT),
                 replace('email', 'mnunberg2000@juno.com'),
                 arrayinsert('interests[4]', 'sitting'),
                 addunique('likes', 'running'))

## Java
 * `DocumentFragment` will be used as input parameter for single mutations as well as return type for single lookups and mutations.
 * `DocumentFragment` will NOT implementing the `Document` interface (so it's impossible to do `bucket.upser(someDocumentFragment)`).
 * method naming follow the Java camelCase convention so eg. `arrayinsert_in` becomes `arrayInsertIn`.
 * multi lookup result is a `MultiLookupResult` with a list of `LookupResult` and convenience methods to detect if it was a total success, partial success or total failure (eg. every LookupSpec was on a path not found).
 * `LookupResult` also have convenience methods:
   - `valueOrThrow()` to go from an error code to throwing the same exception that would have been thrown by `getIn` or `existIn`
   - `exists()`: returns true if the status is `SUCCESS`, which makes sense in the Lookup.EXIST case as well as the Lookup.GET case (where `exists() == true` means you can safely look at the `value()`).
 * multi mutation result is a `MultiMutationResult` that only contains the document's id, the updated cas and optional MutationToken.
 * **TODO** `extend_in` and `arrayinsert_in` core-level implementations don't yet support multiple values.

# Unresolved Questions

* Array method naming: `append` and `prepend` vs `extend`?
* Array prefix: `arrayXXXX`?
* How should multi mutation errors be reported?
* For languages supporting overloads, should one simply overload `get()`, `upsert()`, etc.

# Signoff
If signed off, each representative agrees both the API and the behavior will be implemented as specified.

| Language | Representative | Date       |
| -------- | -------------- | ---------- |
| Java     | Michael N.     | |
| C        | Mark N.        | |
| Python   | Mark N.        | |
| .NET     | Jeff M.        | |
| node.js  | Brett L.       | |
| PHP      | Brett L.       | |
| Go       | Brett L.       | |
