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
* DocumentFragment: A container of a list of sub-documents.

## Overview

The Sub-Document API exposes CRUD-like methods which operate on paths _within_
documents. In these respects it is identical to CRUD methods except that it
treats each path as its own "sub-document" (hence the name).

While internally our Subdocument protocol supports both single and multi operations, we only support a single builder API which permits you to do both single and multi operations.

## Objects

### Document/Item

This class is used in top-level KV operations. The `Document` object itself is the return value from normal KV operations which contain the value, cas, mutation token, etc.

### DocumentFragment

This class is used as the resulting value from all subdocument operations.  It represents a fragment of a full Document and may contain numerous paths within the document.

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

## API Methods

The Subdocument APIs will be exposed through two separate builders which are available through two new top-level Bucket methods `LookupIn` and `MutateIn`.  Additionally, both `MutateIn` and `LookupIn` when executed will result in a `DocumentFragment` object, which represents the top-level results (Status, CAS, MutationToken) and also contains the result of all subdocument operations which were executed.

### Bucket Methods
`Bucket.LookupIn(doc_id) -> LookupBuilder`
Creates a `LookupInBuilder` object which you can then use method-chaining to populate with lookup operations and then later execute.

`Bucket.MutateIn(doc_id, cas, expiry) -> MutateBuilder`
Creates a `MutateInBuilder` object which you can then use method-chaining to populate with mutation operations and then later execute.

### LookupInBuilder Class
This builder exposes the creation of a set of lookup operations to be performed.  Note that empty paths passed to either of the following methods must be considered a programming error.

`LookupInBuilder.Get(path) -> LookupInBuilder`
`LookupInBuilder.Exists(path) -> LookupInBuilder`

`LookupInBuilder.Do() -> DocumentFragment`

### MutateInBuilder Class
This builder exposes the creation of a set of mutation operations to be performed.  Note that empty paths passed to the `Insert`, `Upsert`, `Replace`, `Remove`, `ArrayInsert` or `Counter` methods must be considered a programming error.

`MutateInBuilder.Insert(path, value, createParents) -> MutateInBuilder`
`MutateInBuilder.Upsert(path, value, createParents) -> MutateInBuilder`
`MutateInBuilder.Replace(path, value) -> MutateInBuilder`
`MutateInBuilder.Remove(path) -> MutateInBuilder`
`MutateInBuilder.PushFront(path, value, createParents) -> MutateInBuilder`
`MutateInBuilder.PushBack(path, value, createParents) -> MutateInBuilder`
`MutateInBuilder.ArrayInsert(path, value) -> MutateInBuilder`
`MutateInBuilder.AddUnique(path, value, createParents) -> MutateInBuilder`
`MutateInBuilder.Counter(path, delta, createParents) -> MutateInBuilder`

`MutateInBuilder.Do() -> DocumentFragment`

### `DocumentFragment`
This object represents a result of any sub-document operation.

It should expose the CAS for the document in all cases and additionally a mutation token should be available when a mutation operations is performed.

In addition to the document meta-data that is available through this class, there should also be methods which allow access to the contents of any lookups or content-returning mutations that occured.

#### `DocumentFragment.Cas() -> CAS`
This method returns the CAS of the document which was operated on.  This may be exposed as a property rather than method in languages where this makes sense.

#### `DocumentFragment.MutationToken() -> MutationToken`
This method returns a MutationToken of the document which was operated on when returned by MutateIn and if Mutation Tokens are enabled.  This may be exposed as a property rather than method in languages where this may make sense.

#### `DocumentFragment.Content(string path) -> VALUE`
This method returns the value of the operation that was performed on the specified path.  In the case that two operations are performed on the same path, the result that is returned is undefined, but should be consistent within the same SDK.  This method should return/throw exceptions representing any errors that were returned from the server for the returned operation.  Additionally, this method should return the language-specific non-value for valueless mutations (ie: null in Javascript, nil in Go).

#### `DocumentFragment.Content(int index) -> VALUE`
This method returns the value of the operation at the specified index.  This method should return/throw exceptions representing any errors that were returned from the server for the returned operation.  Additionally, this method should return the language-specific non-value for valueless mutations (ie: null in Javascript, nil in Go).

#### `DocumentFragment.Exists(string path) -> bool`
This method returns a boolean value indicating whether the specified path exists within the result set.  This should only return true if the indicated path was a path which was executed as part of the source operation set and the server returned a successful status code for that operation.

## Errors

In addition to the traditional document-level errors (which are received
when there is a problem with the document - for example, is missing or
has a different CAS), there are additional sub-document specific error codes.

Be aware that for lookup operations which fail partially in languages which can return both an error and a value, the error returned should be MULTI_ERROR.  In languages where this is not possible, no high-level MULTI_ERROR error should be returned.

Errors can be thrown for these reasons:

* Path-based: Bad input provided to the command (for example, an invalid path for an operation)
* Document-based: Bad document provided to command (for example, not json, or too deep)
* Execution-based: The document and path are ok, but an execution error resulted when trying to apply a given operation to a given document (e.g. path does not exist).

Some error codes straddle the line between execution and path-based, most notably `PathMismatch`.

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

* NumberTooBig: Existing number value in document is too big (out of range between `INT64_MIN` and `INT64_MAX`)
* WouldOverflow: Combining delta and existing number would result in overflow/underflow
* BadDelta: Delta is 0, not a number, or beyond range of `int64_t`

## Error Propagation
Internally, multi subdoc commands will fail with `MULTI_FAILURE` if one of the
sub-specifications failed. However with single commands (at the protocol layer) the
`MULTI_FAILURE` error code is absent, and instead, the error code is encoded as the
top-level error code.

SDKs should ensure that error behavior between multi and single commands
is transparent, which according to the API above, means that a `MULTI_FAILURE`
error code should be returned per above. SDKs should ensure that `MULTI_FAILURE`
behavior is _only_ emulated when the top-level code is indeed a subdoc error
code and not a general document-access error code. A subdoc error code may be
defined as:

> An error code which, if received in the context of a multi-operation, would
> not prevent the successful execution of other operations in that packet.

Thus, `PATH_NOT_FOUND` is a subdoc error code and would not prevent the execution
of other operations; whereas `KEY_EEXISTS` is a document access error code and
would inherently invalidate any other operation within the theoretical packet.

# Language Specifics
In this section, the abstract design parts need to be broken down on the SDK level by each maintainer and signed off eventually.

## C

The C SDK will not use these verbs, and will instead model the subdoc operations
after the existing `lcb_store3()` API - using `lcb_sdstore3`:

```c
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
```

## Go
```go
// -- MUTATE IN EXAMPLE --
res, err := bucket.MutateIn('testjson', 0, 0).
    Replace('invalid', 'Frederick').
    Do()
//err: Subdocument mutation 0 failed (Sub-document path does not exist)
//res==nil: true

// -- LOOKUP IN EXAMPLE --
res, err := bucket.LookupIn('testjson').
    Get('name').
    Get('invalid').
    Do()
//err: Could not execute one or more multi lookups or mutations.
//res==nil: false

err = res.Content('name', &value)
//err: <nil>
//value: Frederick

err = res.Content('invalid', &value)
//err: Sub-document path does not exist
//value:
```

# Unresolved Questions

* Array method naming: `append` and `prepend` vs `extend`?

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
