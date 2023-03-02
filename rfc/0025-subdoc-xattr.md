# Meta

 * RFC Name: SubDocument additions for Couchbase 5.0 including XATTRs
 * RFC ID: 25
 * Start Date: 2017-01-12
 * Owner: Brett Lawson (previously Mark Nunberg)
 * Current Status: ACCEPTED

# Summary

API to access extended (hidden) document attributes (XATTRS).

This RFC depends on [0002: Subdocument API](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/0002-subdocapi.md)

# Motivation

To facilitate development of frameworks and storage of data which is otherwise not part of the document body.  This may include items such as the documents class, referential information between documents, or document schema meta-information. Spring-Data-Couchbase stores meta type information of an entity in `_class` attribute. It needs this info for deserialization from JSON to the respective object through data mapper. Type information is always stored with the object on save and fetched on retrieval.

# Changelog

 * January 18 2017:
   * EXPAND_MACROS is something else, and is intended to be used for values that are dynamic depending on the state of the document.
 * April 26 2017:
   * Moved doc flags to their own section to reflect protocol changes: CreateDocument, AccessDeleted
 * May 1 2017:
   * Xattr option name is *xattr* from *attribute_access*
 * May 3 2017:
   * Add section on subdoc-encapsulated fulldoc access.
 * May 16 2017:
   * CreateDocument to UpsertDocument
   * Add GET_COUNT opcode (MutateInBuilder)
 * August 11 2017:
   * Cleaned up sections
 * September 6 2017:
   * The section on subdocument flags still had CreateDocument instead of UpsertDocument and it was in an email on review of this document.  That was later determined to be a typo, now fixed. The pair is **UpsertDocument/InsertDocument**.

# General Design

Accessing XATTRs is done through the subdocument API. The Memcached protocol will feature new flags, `SUBDOC_FLAG_XATTR_PATH` and `SUBDOC_FLAG_EXPAND_MACROS`. These flags can be OR'd with the existing flags (`FLAG_MKDIR_P` and `FLAG_MKDOC`) to access XATTRs.

On the SDK side, XATTR support is added through exposing a new "xattr access" feature to the existing subdocument API.

## Modifications to current implementations

### HELLO

A new `HELLO` flag has been introduced to indicate if the cluster supports XATTR operations. This has been introduced to safeguard against a partially-upgraded clusters where some servers could handle XATTR operations and some cannot.

Each SDK should take note if the cluster allows XATTR operations and return appropriate errors if an XATTR operation is attempted against a cluster that does not support them.

### API

#### Enum `SubdocPathFlags`

The following table lists the various subdoc options. These options must be able to exist together (e.g. in a bitmask or other manner). None of these options are mutually exclusive of another


| Name | Description | Protocol Flag |
| ---- | ----------- | ------------- |
| CreatePath        | Create path if it does not exist (For use in mutations)| `SUBDOC_FLAG_MKDIR_P` = `0x01` |
| Xattr       | Current path refers to a location within the document's hidden/extended attributes, not the document body |  `SUBDOC_FLAG_XATTR_PATH` = `0x04` |
| ExpandMacroValues [Private]     | Value contains references to macros (such as `${Mutation.CAS}`, etc.. Implies Xattr. | `SUBDOC_FLAG_EXPAND_MACROS` = `0x010` |


#### Enum `SubdocDocFlags`

These flags control how subdoc behaves in respect to the entire document. If they are specified, they are to be encoded into a new protocol field. This field is 1 byte wide and goes *after* any other command extras, such that the last byte of extras is always the doc flags.

If there are no doc flags set by the user, then this field **should** be omitted. The subdoc command specification contains the low level details with respect to encoding these flags.

These flags are described in detail within the [subdoc spec](https://docs.google.com/document/d/1hOQIPMTEFTmdNgWftSZddMtoc57Zi_yuKJ8BW-QJSiQ/edit).

| Name | Description | Protocol Flag |
| ---- | ----------- | ------------- |
| UpsertDocument | Create document if it does not exist (For use in mutations. Implies CreatePath path flag for each path) | `0x01` |
| InsertDocument | Like UpsertDocument, except that the document is only created if it does not exist. This option only makes sense in the context of wishing to create a new document together with xattrs. | `0x02` |
| AccessDeleted[Private] | Access documents which have already been deleted | `0x04` |

### Response Codes and Errors

There are also some additional response status codes related to XATTR operations. Note that other subdocument errors (e.g. `PATH_ENOENT`, `NOT_FOUND`) etc. may also be returned.


| Name | Description | Protocol Value |
| ---- | ----------- | -------------- |
| XattrInvalidFlagCombo | The combination of the subdoc flags for the xattrs doesn't make any sense | `SUBDOC_XATTR_INVALID_FLAG_COMBO` = `0xce` |
| XattrInvalidKeyCombo | Only a single xattr key may be accessed at the same time | `SUBDOC_XATTR_INVALID_KEY_COMBO` = `0xcf` |
| XattrUnknownMacro | The server has no knowledge of the requested macro | `SUBDOC_XATTR_UNKNOWN_MACRO` = `0xd0` |

### MutateInBuilder

Pattern: `<Method>(String path, Object value, SubdocDocFlags docFlags)` -> `MutateInBuilder`

This new overload (or 'Ex') variant will allow to set document-specific flags for entire operations.

 * `Upsert(String path, Object value, SubdocPathFlags options)`
 * `Insert(String path, Object value, SubdocPathFlags options)`
 * `Replace(String path, Object value, SubdocPathFlags options)`
 * `Remove(String path, Object value, SubdocPathFlags options)`
 * `ArrayAppend(String path, Object value, SubdocPathFlags options)`
 * `ArrayAppend(String path, SubdocPathFlags options, object[] values)`
 * `ArrayPrepend(String path, Object value, SubdocPathFlags options)`
 * `ArrayPrepend(String path, SubdocPathFlags options, object[] values)`
 * `ArrayInsert(String path, Object value, SubdocPathFlags options)`
 * `ArrayInsert(String path, Object value, SubdocPathFlags options)`
 * `ArrayAddUnique(String path, Object value, SubdocPathFlags options)`
 * `Counter(String path, Long delta, SubdocPathFlags options)`

Original methods that contain `bool createPath` should be deprecated.

**NOTE**: If bitflags are not idiomatic to your SDK, you may provide these options in the form of a DSL, for example:

```
lookupIn().get(path, SubdocOptions.Xattr()());
```

Or

```
lookup_in(SD.get(path, xattr=True))
```

### LookupInBuilder

 * `Get(String path, SubdocPathFlags options)`
 * `Exists(String path, SubdocPathFlags options)`
 * `GetCount(String path, SubdocPathFlags options)`

## Fulldoc access via Subdoc

While not strictly related to extended attributes, the server also now allows sending full-doc commands such as `CMD_GET (0x00)` and `CMD_SET (0x01)` encapsulated within a subdoc "multi" command. `CMD_SET` can be used in conjunction with xattr commands to upsert a document together with its xattrs atomically. Likewise `CMD_GET` may be used to retrieve a document alongside its xattrs atomically.

In the SDKs the proposed way to expose this is by allowing an empty path to the `get()` and `upsert()` sub-functions of *LookupInBuilder*. *MutateInBuilder*.

**Note:** You must encapsulate a `CMD_SET` or `CMD_GET` in a `MULTI_MUTATION/MULTI_LOOKUP` command, as these opcodes have other meanings in their standalone context. It actually doesn't make sense to have a standalone `CMD_SET` in a subdoc context anyway, as it's then equivalent to a normal CMD_SET.

### LookupInBuilder

#### `Get()`

Allow for an empty path. Optionally provide a `SUBDOC_ROOT` constant. Internally this should map to encapsulating a `CMD_GET` rather than a `CMD_SUBDOC_GET`.

Optionally, languages may also implement an overload without arguments.

### MutateInBuilder

#### `Upsert(Object value)`

See description for `get()`.

## `GET_COUNT` Operation

 - this belongs on the `LookupInBuilder class`

#### `L.GetCount(path) -> L`

Retrieve the number of elements in the path. The path should point to an array or dictionary. The result will be the number of elements in JSON format. (i.e. a JSON number).

See the [Subdocument commands specification # GET_COUNT](https://github.com/couchbase/memcached/blob/master/include/memcached/protocol_binary.h#L510) for more information.

## Materialized (Virtual) Attributes

Materialized attributes are implemented in the server using the *$document* xattr, e.g. `lookup_in(sd.get('$document', xattr=True))`. These attributes are generated on-demand to expose storage-level document metadata and are not actually part of the document.

At this point, no additional API changes beyond xattr support is required to support virtual attributes.

# Language Specifics

## Python

Python uses keyword arguments rather than an enum, naturally:

```python
import couchbase.subdocument as SD

cb.mutate_in(docid, SD.upsert('my.xattr', 'foo', create_parents=True, xattr=True))
print cb.lookup_in(docid, SD.get('my.xattr', xattr=True))
cb.mutate_in(docid, SD.upsert('my.macrod', '${Mutation.seqno}', xattr=True, _expand_macros=True))
print cb.lookup_in(docid, SD.get('my.macrod', xattr=True))
cb.mutate_in('mattup', SD.upsert('foo.bar.baz', 'foo'), upsert_document=True) # PYCBC-424: upsert_document fixing
# cb.mutate_in(docid, SD.upsert(...), insert_document=True)

cb.mutate_in(docid, SD.upsert('my.xattr', xattr=True, _expand_macros=True))

cb.lookup_in(docid, SD.get('my.xattr', xattr=True))

cb.mutate_in(docid, SD.upsert('foo.bar.baz'), upsert_document=True)

cb.mutate_in(docid, SD.upsert(...), insert_document=True)

cb.lookup_in(docid, SD.get_doc(), SD.get('my.xattr', xattr=True))

cb.mutate_in(docid, SD.upsert_doc(docid), SD.upsert('my.xattr', xattr=True))
```

## Java

LookupIn and MutateIn builders accept subdoc options. Deprecated the old overload with createParents

```java
bucket.mutateIn(key).insert("spring.class", "SomeClass", new SubdocOptionsBuilder().createParents(true).xattr(true)).execute()

bucket.lookupIn(key).exists("spring.class", new SubdocOptionsBuilder().xattr(true)).execute()

bucket.lookupIn(key).get("spring.class", new SubdocOptionsBuilder().xattr(true)).execute()

bucket.mutateIn(key).remove("spring.class", new SubdocOptionsBuilder().xattr(true)).execute()
```

## .NET

.NET client uses bit flags to indicate additional behaviour for SubDoc operations:

```csharp
var getResult = _bucket.LookupIn<dynamic>(key)
  .Get("_data.created_by", SubdocPathFlags.Xattr)
  .Execute();

bucket.LookupIn<dynamic>("key")
  .Get("app.created_by", SubdocLookupFlags.AttributePath)
  .Execute();

var mutateResult = _bucket.MutateIn<dynamic>(key)
  .Upsert("_data.created_by", username, SubdocPathFlags.CreatePath | SubdocPathFlags.Xattr).Execute();

bucket.MutateIn<dynamic>("key")
  .Upsert("app.created_by", "mike", SubdocMutateFlags.AttributePath | SubdocMutateFlags.CreatePath)
  .Execute();
```

### C

New subdoc commands.

 - `LCB_SDCMD_GET_FULLDOC`
 - `LCB_SDCMD_SET_FULLDOC`

In C, the following new flags are available for lcb_SDSPEC::options:

New per-spec flags (lcb_SDSPEC::flags)

 - `LCB_SDSPEC_F_XATTRPATH`
 - `LCB_SDSPEC_F_XATTR_MACROVALUES`

New command flags (lcb_CMDSUBDOC::cmdflags)

 - `LCB_CMDSUBDOC_F_UPSERT_DOC`
 - `LCB_CMDSUBDOC_F_INSERT_DOC`
 - `LCB_CMSUBDOC_F_ACCESS_DELETED`

Note that specifying any of these attributes implies `F_XATTTRPATH`

### Node.js

 - `bucket.lookupIn('key').get('app.created_by', {xattr:true}).execute()`
 - `bucket.mutateIn('key').upsert('app.created_by', 'mike', {xattr:true, createParents:true}).execute()`

### Go

 - `bucket.LookupIn("key").GetEx("app.created_by", SubdocFlagXattr).Execute()`
 - `bucket.MutateIn("key").UpsertEx("app.created_by", "mike", SubdocFlagXattr | SubDocFlagMkDir).Execute()`

### PHP

 - `$bucket->lookupIn("key")->get("app.created_by", ["xattr" => true])->execute();`
 - `$bucket->mutateIn("key")->upsert("app.created_by", "mike", ["xattr" => true, "createPath" => true])->execute();`

# Questions

**(Matt) How would I retrieve all XATTRS?**
You don't.

**(Unknown) How would one access the TTL on the XATTR?**

At this time, access virtual attributes of a document will be performed through the same pathway used to access normal extended-attributes, virtual attributes exist as a virtual extended-attribute called `$document`.  For instance, `lookup_in(key).get('$document.exptime', xattr=True)`.  No specialized API will be provided for this.

**(Unknown) Can I store arbitrary byte[]?**

Extended attributes are JSON-specific and are not able to store binary data directly (unless the data is first encoded into something JSON-like (for instance an escaped string).

**(Brett) Are XATTRs be visible over DCP?**

Extended attributes can be streamed via DCP with special flags being enabled.  These values are streamed in a specialized value format.

**(Unknown) Are materialized xattributes (ie: virtual attributes) going to handled specially?
**No.  They will be handled as any other extended-attribute would be.

**(Brett) Should we expose some method of 'generic application meta-data' (ie: a default xattribute where you can store meta-data for your application)?**

At this point in time, we do not which to expose xattrs for generic application meta-data, but instead limit it to framework-oriented use-cases.  For this reason, there will be no specific application xattrs.

**(Unknown) Is there any use in providing access to the 'expand macros' API? - note, this is probably still needed for Python and Go, but the question is whether it's also needed for other SDKs or common use cases.**

ExpandMacros will be an available operation for sub-document operations, enabling the use of them.  We will not provide an explicit API at this time.  The python and go usage is for internal to Couchbase only.  Macros are further defined in the [Mobile Convergence KV-Engine Specification](https://docs.google.com/document/d/18UVa5j8KyufnLLy29VObbWRtoBn9vs8pcxttuMt6rz8/edit#heading=h.t0wlj1ke9naq).

**(Unknown) How should we call AttributeAccess? Python uses 'xattr', other languages might do something different.**

We will use the term "XATTR" to refer to extended-attribute access within the SDKs.

**(Unknown) Is KV Error Map required to use extended attributes?**

No. Extended attributes are completely independent from any of these other specifications.

**(Matt) Should I use long or short names for the document flags?**

You should follow the existing naming patterns from the specific SDK which is being implemented.

# Signoff

<table>
  <tr>
    <td>Language</td>
    <td>Representative</td>
    <td>Date</td>
  </tr>
  <tr>
    <td>C</td>
    <td>Sergey Avseyev</td>
    <td>2017-08-23</td>
  </tr>
  <tr>
    <td>Go</td>
    <td>Brett Lawson</td>
    <td>2017-08-11</td>
  </tr>
  <tr>
    <td>Java</td>
    <td>Michael Nitchinger</td>
    <td>2017-08-23</td>
  </tr>
  <tr>
    <td>.NET</td>
    <td>Jeff Morris</td>
    <td>2017-09-05</td>
  </tr>
  <tr>
    <td>NodeJS</td>
    <td>Brett Lawson</td>
    <td>2017-08-11</td>
  </tr>
  <tr>
    <td>PHP</td>
    <td>Sergey Avseyev</td>
    <td>2017-08-23</td>
  </tr>
  <tr>
    <td>Python</td>
    <td>Matt Ingenthron</td>
    <td>2017-09-06</td>
  </tr>
</table>
