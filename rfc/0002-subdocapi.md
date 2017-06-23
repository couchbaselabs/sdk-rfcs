# **Meta**

* RFC Name: SubDocument API for clients

* RFC ID: 0002

* Start Date: 2015-11-11

* Owner: Mark Nunberg

* Current Status: Review

## Summary

This RFC proposes a client-side API for the Sub-Document operations which will be released in a future server version.

## Motivation

Expose the high performance subdocument API to developers

[[TOC]]

# General Design

PLEASE READ THIS FIRST [https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#)

The link above contains the full commands specification at the behavior and protocol layer. The specification is considered authoritative, and this RFC merely adapts the SDKs to the aforementioned specification.

## Terminology

* Document: A document or item in the Memcached sense.

* Path: A specific location in a JSON document. A path uses the path syntax described in the server-side API.

* Sub-Document: A section of a document, specified by the path.

* Top-level: Document-level operations, for example upsert.

* DocumentFragment: A container of a list of sub-documents.

## Overview

The Sub-Document API exposes CRUD-like methods which operate on paths within documents. In these respects it is identical to CRUD methods except that it treats each path as its own "sub-document" (hence the name).

While internally our Subdocument protocol supports both single and multi operations, we only support a single builder API which permits you to do both single and multi operations.

## Objects

### Document/Item

This class is used in top-level KV operations. The **Document** object itself is the return value from normal KV operations which contain the value, cas, mutation token, etc.

### DocumentFragment

This class is used as the resulting value from all subdocument operations. It represents a fragment of a full Document and may contain numerous paths within the document.

## Path

From the perspective of the SDK, the path is a simple string used to specify a location within the document. The path uses N1QL syntax to specify locations within a document, thus a valid path may be:

    `foo.bar.baz`

Or

    `foo`.`bar`.`baz`


Or `foo.some_array[0]` or `foo.some_array[-1]`

As in N1QL, special path characters may be escaped via a backtick (`).

Some operations (notably, those taking a path to an array) may accept the empty path (i.e. an empty string) which refers to the entire document. This is valid if the document is a bare JSON array.

Each path component must be valid JSON (without the enclosing quotes). This means that things like tabs, quotes, and backslashes must be escaped. To think of it another way, the path matching algorithm in the server works via simple string comparison, so a JSON key `"bs\\ and \t tabs are here"` would be matched by a path component of `"bs\ and \t tabs are here"`.

The SDK MUST verify that a path is non-empty for operations which do not support empty paths. All other forms of path verification is done on the server (but an empty path for unsupported operations will return a memcached **EINVAL** which isn't really helpful).

# API Methods

The Subdocument APIs will be exposed through two separate builders which are available through two new top-level Bucket methods **LookupIn** and **MutateIn**. Additionally, both **MutateIn** and **LookupIn** when executed will result in a **DocumentFragment** object, which represents the top-level results (Status, CAS, MutationToken) and also contains the result of all subdocument operations which were executed.

### Opcodes

All data access methods below have corresponding server-level opcodes which may be found here: [https://github.com/couchbase/memcached/blob/master/include/memcached/protocol_binary.h](https://github.com/couchbase/memcached/blob/master/include/memcached/protocol_binary.h)

## Bucket Methods

#### Bucket.LookupIn(doc_id) -> LookupBuilder

Creates a **LookupInBuilder** object which you can then use method-chaining to populate with lookup operations and then later execute.

#### Bucket.MutateIn(doc_id, cas, expiry) -> MutateBuilder

Creates a **MutateInBuilder** object which you can then use method-chaining to populate with mutation operations and then later execute.

#### Bucket.RetrieveIn(doc_id, paths...) -> DocumentFragment [OPTIONAL]

This is a shorthand alternative to constructing a builder, for allowing simple, idiomatic retrieval of items.

    RetrieveIn(docid, path1, path2, path3)

is equivalent to:

    LookupIn(docid).Get(path1).Get(path2).Get(path3).Do()

In some languages `RetrieveIn` may offer significant syntactic simplicity, and is therefore optional

### Encoding

Details on how operations should be encoded may be found within the protocol. Note that at the protocol level there are *two* ways to encode mutation and lookup commands. One is [single command](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.96ul8ovnrqhr)[ encoding ](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.96ul8ovnrqhr)and the other is [multi command](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.iohx97ltyes)[ encoding](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.iohx97ltyes). While the encoding details are abstracted from the user, the SDK **must** ensure that *single* *command* encoding is used when there is only a single operation within the builder.

*single command* encoding saves 7 bytes

> NOTE:  For the request: `{npath(2)+flag(1)}` in extras for single, vs `{opcode(1),npath(2),flag(1)}` in multi, saving one byte. For the response: `{status(2)+nvalue(4)}` vs `{...implicit...}` saving 6 bytes, saving a total of 7 bytes overall.) per operation for lookups and 5-12 bytes

> NOTE:  For the request: `{npath(2)+flag(1)}` in extras for single, vs `{opcode(1),npath(2),flag(1),nvalue(4)}` in multi, saving 5 bytes. For non-counter responses, the payload is empty; for counter responses the response contains `{index(1),status(2),nvalue(4)}`, which is 7 bytes, vs empty (excluding the actual data) for single).) per operation for mutations. Because subdoc`s primary goal is to save bandwidth, the SDK should ensure to maximize its encoding features.

#### Expiration

For mutations, subdoc allows to specify the expiration. Refer to the spec for encoding details. The SDK should not encode expiration (i.e. specify 0) if the user did not explicitly request an expiration time.

### LookupInBuilder Class

This builder exposes the creation of a set of lookup operations to be performed. Note that empty paths passed to either of the following methods must be considered a programming error.

#### L.Get(path) -> L

Retrieves a path's value from a document. Refer to [GET](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.hfbqcirlfdhy) in the spec. In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_GET`

#### L.Exists(path) -> L

Check if a path exists in the document. Refer to [EXISTS](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.6kssvsak0ue0) in the spec. In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_EXISTS`

#### L.Do() -> DocumentFragment

Note: for some languages, "do" can be a reserved word (eg. Java).

### MutateInBuilder Class

This builder exposes the creation of a set of mutation operations to be performed. Note that empty paths passed to the **Insert,** **Upsert,** **Replace,** **Remove,** **ArrayInsert** or Counter methods must be considered a programming error.

#### M.Insert(path, value, createParents) -> M

Inserts a new value into a dictionary. Refer to [DICT_ADD](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.sn5908a1dlo4) (in the spec). In Memcached, this is `PROTOCOL_BINARY_CMD_SUBDOC_DICT_ADD`.

The *createParents* option is mapped to the `SUBDOC_FLAG_MKDIR_P` and has a value of 0x01

#### M.Upsert(path, value, createParents) -> M

Inserts or replaces a path into a dictionary. Refer to [DICT_UPSERT](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.2247ha5v2ku0) (in the spec). In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_DICT_UPSERT`.

#### M.Replace(path, value) -> M

Replaces an existing path (the path can either be an array index or a dictionary). Refer to [REPLACE](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.pzmstailha6l) in the spec. In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_REPLACE`

#### M.Remove(path) -> M

Removes a path. Refer to [DELETE](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.66doo0bfbiyw) in the spec. In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_DELETE`

#### M.ArrayPrepend(path, value, createParents) -> M

Adds a value to the beginning (left) of an array. Refer to [PUSH_FIRST](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.cw9augia9ok7) in the spec. In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_ARRAY_PUSH_FIRST`.

#### M.ArrayAppend(path, value, createParents) -> M

Adds a value to the end (right) of an array. Refer to [PUSH_LAST](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.kao18yxnao3j) in the spec. In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_ARRAY_PUSH_LAST`

#### M.ArrayInsert(path, value) -> M

Inserts a value at a given position within an array. The position is indicated as part of the path. Refer to [ARRAY_INSERT](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.no2iyqi2qfcz). In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_ARRAY_INSERT`

#### M.ArrayAddUnique(path, value, createParents) -> M

Adds a value to an array if the value does not already exist in the array. Refer to [`ARRAY_ADD_UNIQUE`](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.fcuii6b8s2hu) in the spec. In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_ARRAY_ADD_UNIQUE`.

#### M.Counter(path, delta, createParents) -> M

Perform an arithmetic operation on a numeric value in a document. Refer to [COUNTER](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.oehj6fynxzhm) in the spec. In Memcached this is `PROTOCOL_BINARY_CMD_SUBDOC_COUNTER`. SDKs Should throw a BadDelta exception when the given delta is 0 *before* the command is sent to the server.

#### M.Do() -> DocumentFragment

### Multi-Value Array Operations

The protocol allows to specify *[multiple](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.r9wqti7ot83n)[ values](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit#heading=h.r9wqti7ot83n)* for the `ArrayAppend`, `ArrayPrepend`, and `ArrayInsert`. Instead of specifying _n_ `Append` (or similar) operations to add _n_ items, one can encode the value as a series of comma-separated values. In essence specifying a JSON array without the enclosing brackets.

SDKs **must** implement a way to submit multiple values so that they are encoded in such a manner. This may be done in the following ways (among others):

* Variable argument method overload: e.g. `ArrayAppend(path, value1, value2, value3)`

* Explicit *multi* argument w/variable arguments: e.g. `ArrayAppendAll(path, value1, value2, value3)`

* Explicit *multi* argument w/container: e.g. `ArrayAppendAll(path, []int{1,2,3,4})`

* Container object: e.g. `ArrayAppend(path, MultiValue(1,2,3,4))`

The list above just gives some ideas and should not be thought of as limiting the way multiple values may be specified. It is recommended that multiple values be passed in a manner idiomatic for the platform. Implementations and designs must take care to disambiguate between inserting a collection as a single element (i.e. to add a nested array to an existing array) and inserting multiple elements into an array.

In python this can be expressed as:

Assuming the existing path contains: `["Hello", "World", null]`

* `array_append(path, 1, 2, 3, 4)` => results in `["Hello", "World", null, 1, 2, 3, 4]`

* `array_append(path, [1,2,3,4])` => results in `["Hello", "World", [1,2,3,4]]`

### DocumentFragment

This object represents a result of any sub-document operation.

It should expose the CAS for the document in all cases and additionally a mutation token should be available when a mutation operations is performed.

In addition to the document meta-data that is available through this class, there should also be methods which allow access to the contents of any lookups or content-returning mutations that occurred.

#### DocumentFragment.Cas() -> CAS

This method returns the CAS of the document which was operated on. This may be exposed as a property rather than method in languages where this makes sense.

#### DocumentFragment.MutationToken() -> MutationToken

This method returns a _MutationToken_ of the document which was operated on when returned by MutateIn and if Mutation Tokens are enabled. This may be exposed as a property rather than method in languages where this may make sense.

#### DocumentFragment.Content(string path) -> VALUE

This method returns the value of the operation that was performed on the specified path. In the case that two operations are performed on the same path, the result that is returned is undefined, but should be consistent within the same SDK. This method should return/throw exceptions representing any errors that were returned from the server for the returned operation. Additionally, this method should return the language-specific non-value for valueless mutations (ie: null in Javascript, nil in Go).

#### DocumentFragment.Content(int index) -> VALUE

This method returns the value of the operation at the specified index. This method should return/throw exceptions representing any errors that were returned from the server for the returned operation. Additionally, this method should return the language-specific non-value for valueless mutations (ie: null in Javascript, nil in Go).

#### DocumentFragment.Exists(string path) -> bool

This method returns a boolean value indicating whether the specified path exists within the result set. This should only return true if the indicated path was a path which was executed as part of the source operation set and the server returned a successful status code for that operation.

## Errors

In addition to the traditional document-level errors (which are received when there is a problem with the document - for example, is missing or has a different CAS), there are additional sub-document specific error codes.

Because sending *n* operations may result in receiving 0..*n* errors, the library should signal somehow, as the top-level error code, a MultiError error type. This would signal to the user that one or more sub-commands failed.

Note that the behavior for mutation and lookup failures may differ in this case. Because mutations do not (currently) allow for partial failure, and because the *nature *of mutations is essentially fire-and-forget, mutation errors should immediately trigger an exception informing the user of the specific error (or; a multi-failure with a nested "cause" containing the first error).

For multi lookups where the application is expected to check the value of each individual result (as the operation itself is not of much use without the return value), the exception may be raised when the individual result itself is accessed IF the error was not-found.

In substituting the two paragraphs above, SDKs may treat multi mutations with the same semantics as general mutations, and multi lookups with the same semantics as general lookups

Errors can be thrown for these reasons:

* Path-based: Bad input provided to the command (for example, an invalid path for an operation)

* Document-based: Bad document provided to command (for example, not json, or too deep)

* Execution-based: The document and path are ok, but an execution error resulted when trying to apply a given operation to a given document (e.g. path does not exist).

Some error codes straddle the line between execution and path-based, most notably **PathMismatch.**

### Generic Errors

* **PathNotFound (0xC0, `SUBDOC_PATH_ENOENT`)**: Received when a path does not exist in the document. The exact meaning of path existence depends on the operation and inputs.

* **PathExists (0xC9, `SUBDOC_PATH_EEXISTS`)**: Received if a path already exists and it shouldnt't (for example, replace_in).

* **PathMismatch (0xCa, `SUBDOC_PATH_MISMATCH`)**: Received if the path structure conflicts with the document structure (for example, if a path mentionsfoo.bar[0].baz, but foo.bar is actually a JSON object.

* **PathInvalid (0xC2,  SUBDOC_PATH_EINVAL`)**: Path has a syntax error, or path syntax is incorrect for operation (for example, if operation requires an array index).

* **CantInsertValue (0xC5, `SUBDOC_VALUE_CANTINSERT`)**: If the provided value cannot be inserted at the given path. This is usually because the value is invalid JSON and should generally not be visible in SDKs which serialize their input beforehand. This may also be received for *COUNTER* operations, in which case it means that performing the operation would result in an overflow/underflow.

* **DocumentNotJson (0xC6, `SUBDOC_DOC_NOTJSON`)**: Received if document itself is not JSON.

* **PathTooDeep (0xC3, `SUBDOC_PATH_E2BIG`)**: Path is too deep to parse. Depth of a path is determined by how many components (or levels) it contains. The current limitation is 32 levels, though that number is arbitrary. It is there to ensure a single parse does not consume too much memory (overloading the server). This error is similar to other TooDeep errors, which all relate to various validation stages to ensure the server does not consume too much memory when parsing a single document.

* **DocumentTooDeep (0xC4, `SUBDOC_DOC_E2DEEP`)**: Document is too deep to parse.

* **ValueTooDeep (0xCA, `SUBDOC_VALUE_ETOODEEP`)**: Proposed value would make the document too deep to parse. `PROTOCOL_BINARY_RESPONSE_SUBDOC_VALUE_ETOODEEP` = 0xca

* **MultiFailure (0xCC, `SUBDOC_MULTI_PATH_FAILURE`)**. Received when one or more sub-command in a multi operation have failed. The actual reason for failure is found in the response payload (see spec for how to decode this).

### Numeric Errors

These errors are only received for COUNTER operations:

* **NumberTooBig (0xC7, `SUBDOC_NUM_ERANGE`):** Existing number value in document is too big (out of range between `INT64_MIN` and `INT64_MAX`).

* **CantInsertValue (synthesized, see above):** Combining delta and existing number would result in overflow/underflow. (NOTE:  Note that `CANT_INSERT` would normally mean that the value is not JSON; however when a non-JSON value is passed to `COUNTER`, the `DELTA_EINVAL` error is returned.)

* **BadDelta (0xC8, SUBDOC_DELTA_EINVAL):** Delta is 0, not a number, or beyond range of `int64_t`. Note that the 0 delta case can be detected early by the SDK and directly thrown by the builder method.

### Internal Errors

These errors are returned by Memcached and should not be exposed to users. They are indicative of insufficient error checking/validation at the SDK level:

* `EINVAL` (0x04). This is received if the command was not properly encoded. Notably this also includes specifying an empty path for commands which do not allow it.

* `SUBDOC_INVALID_COMBO` (0xCB). This is received if an invalid opcode was specified in a multi operation. This usually means that the SDK didn't do proper input validation to ensure that mutation and lookup commands weren't mixed in the same builder.

### Error Propagation

Internally, multi subdoc commands will fail with **`MULTI_FAILURE`** if one of the sub-specifications failed. However with single commands (at the protocol layer) the **`MULTI_FAILURE`**error code is absent, and instead, the error code is encoded as the top-level error code.

SDKs should ensure that error behavior between multi and single commands is transparent, which according to the API above, means that a **MULTI_FAILURE** error code should be returned per above. SDKs should ensure that **`MULTI_FAILURE`** behavior is only emulated when the top-level code is indeed a subdoc error code and not a general document-access error code.

**A subdoc error code may be defined as:**

*An error code which, if received in the context of a multi-operation, would not prevent the successful execution of other operations in that packet.*

Thus, **`PATH_NOT_FOUND`** is a subdoc error code and would not prevent the execution of other operations; whereas **`KEY_EEXISTS`** is a document access error code and would inherently invalidate any other operation within the theoretical packet.

An exception to that wrapping in a MultiMutation exception is for the case where the delta passed to counter is 0. This case can easily be detected early in the builder, and a BadDelta exception can then be directly thrown by the SDK.

## Language Specifics

In this section, the abstract design parts need to be broken down on the SDK level by each maintainer and signed off eventually.

### C

The C SDK will not use these verbs, and will instead model the subdoc operations after the existing lcb_store3() API - using lcb_sdstore3:

### Go

```code golang
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

### Java

* The *Do()* method for execution of builders will be implemented in Java as *execute()*.

* The *Bucket.retrieveIn()* method won't be implemented in Java, as it doesn't offer that much value with Java verbosity. If users keep wondering why it is not there, then we'll come back to it.

### .NET

* The *Do()* method for execution of builders will be implemented in .NET as Execute()

### Python

* Does not use a "builder pattern" but rather a list of "command specs" which are constructed in a similar fashion.

# **Unresolved Questions**

None.

# **Signoff**

If signed off, each representative agrees both the API and the behavior will be implemented as specified.

<table>
  <tr>
    <td>Language</td>
    <td>Representative</td>
    <td>Date</td>
  </tr>
  <tr>
    <td>Java</td>
    <td>Michael N., Simon B.</td>
    <td>SB April 2 2016, (+ Michael N., 2 May 2016)</td>
  </tr>
  <tr>
    <td>C</td>
    <td>Mark N.</td>
    <td>MN April 27 2016</td>
  </tr>
  <tr>
    <td>Python</td>
    <td>Mark N.</td>
    <td>MN April 27 2016</td>
  </tr>
  <tr>
    <td>.NET</td>
    <td>Jeff M.</td>
    <td>JM April 26 2016</td>
  </tr>
  <tr>
    <td>node.js</td>
    <td>Brett L.</td>
    <td>BL May 6, 2016</td>
  </tr>
  <tr>
    <td>PHP</td>
    <td>Brett L., Sergey A.</td>
    <td>SA May 4 2016</td>
  </tr>
  <tr>
    <td>Go</td>
    <td>Brett L.</td>
    <td>BL May 6, 2016</td>
  </tr>
</table>


# Changelog

* Moved to google docs

* Removed WouldOverflow error (this is actually ValueCantInsert)

* Clarify that 0 delta can be caught at the SDK level for the most part and should trigger an invalid input exception

* Add mention of MultiValue

* Rename ArrayXXX operations: ArrayAppend, ArrayPrepend, ArrayInsert, ArrayAddUnique

* Add RetrieveIn (Optional)

* Specify that either *Do* or *Execute* actually sends the commands to the server

* Clarify common semantics with error handling: Both single and multi results should always return a top-level MULTI_FAILURE (or equivalent) error code.
