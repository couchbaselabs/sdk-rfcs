# Meta

- RFC Name: SDK3 CRUD API
- RFC ID: 0059-sdk3-crud
- Start Date: September 12, 2019
- Owner: Jeff Morris \<jeffry.morris@couchbase.com\>
- Current Status: ACCEPTED
- [Original Google Drive Doc](https://docs.google.com/document/d/1_fPJn9trqG6e7iTpzqwCnurvxmBlguFVjh00F2Co7Y8)

# Motivation

This document is a sub-RFC for the SDK 3.0 RFC and defines the Key/Value API in detail.

# API Entry Point

This API is part of the ICollection interface and provides basic crud operations among other binary Memcached extended operations. It is accessed via the IBucket interface as the default collection or via the IScope interface as a named collection:

## Key/Value Methods on the ICollection interface

### Get, Insert, Upsert, Replace, etc.

```csharp
interface ICollection{

  ...

  IGetResult Get(string id, GetOptions options = null);
  IMutationResult Upsert(string id, T value, UpsertOptions options = null);
  IMutationResult Insert(string id, T value, InsertOptions options = null);
  IMutationResult Replace(string id, T value, ReplaceOptions options = null);
  IMutationResult Remove(string id, RemoveOptions options = null);

  ...

}
```

#### Get

Fetches a value from the server if it exists.

Signature

```csharp
GetResultIGetResult Get(string id, [options])
```

Parameters

- Required:

  - Id: string - primary key as a string.

- Optional:

  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - WithExpiry(boolean) - if set, transparently switches to subdoc and fetches the expiry as well
  - Project (string[]) - a list or array of fields to project - if called will switch to subdoc and only fetch the fields requested.\
    If the number of fields is > 16, then it will perform a full-doc lookup and extract the requested fields instead. ([Discussion link](https://couchbase.slack.com/archives/CCX40TL9F/p1544705305558800))
  - Transcoder - a custom transcoder for converting the memcached packet to a native type.

- Throws:
  - Documented
    - DocumentNotFoundException (#101)
    - RequestTimeoutException (#1)
    - CouchbaseException (#0)
  - Undocumented
    - RequestCanceledException (#2)
    - InvalidArgumentException (#3)
    - TemporaryFailureException (#7)
    - ServerOutOfMemoryException (#102)
    - DurableWriteReCommitInProgressException (#111)

Returns

- The JSON object or scalar encapsulated in an IGetResult API object.

Throws

- Any exceptions raised by the underlying platform

#### Insert

Insert a JSON document, failing if it already exists. Maps to Memcached Add command.

Signature

```csharp
IMutationResult Insert(string id, T value, [options])
```

Parameters

- Required

  - Id: string - primary key as a string.
  - Value - the document (can be JSON).

- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Expiry - the length of time the  document will be stored in Couchbase before being evicted.
  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority
  - Transcoder - a custom transcoder for converting the memcached packet to a native type.

Returns

- An IMutationResult object if successful otherwise an exception with details for the reason the operation failed.

Throws

- Documented

  - DocumentExistsException (#101)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - ValueTooLargeException (#104)
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)

#### Upsert

Insert a new document or overwrite an existing document in Couchbase server. Maps to Memcached Set command.

Signature

```csharp
IMutationResult Upsert(string id, T value, [options])
```

Parameters

- Required

  - Id: string - primary key as a string.
  - Value - the JSON document

- Optional

  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Expiry - the length of time the  document will be stored in Couchbase before being evicted.

  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority
    - Transcoder - a custom transcoder for converting the memcached packet to a native type.

Returns

- An IMutationResult object if successful otherwise an exception with details for the reason the operation failed.

Throws

- Documented

  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - ValueTooLargeException (#104)
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)

#### Replace

Replaces an existing document in Couchbase server, failing if it does not exist. Maps to Memcached SET command.

Signature

```csharp
IMutationResult Replace(string id, T value, [options])
```

Parameters

- Required

  - Id: string - primary key as a string.
  - Value - the JSON document

- Optional
  - Timeout or timeoutMillis (int/duration)  - the time allowed for the operation to be terminated. This is controlled by the client.
  - Expiry - the length of time the  document will be stored in Couchbase before being evicted.
  - CAS - Compare and Set value for optimistic locking of a document.
  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority
  - Transcoder - a custom transcoder for converting the memcached packet to a native type.

Returns

- An IMutationResult object if successful otherwise an exception with details for the reason the operation failed.

Throws

- Documented

  - DocumentNotFoundException (#101)
  - CasMismatchException (#9)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - ValueTooLargeException (#104)
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)

#### Remove

Removes an existing document in Couchbase server, failing if it does not exist.

Signature

```csharp
IMutationResult Remove(string id,  [options])
```

Parameters

- Required

  - Id: string - primary key as a string.

- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - CAS - Compare and Set value for optimistic locking of a document.
  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority

Returns

- An IMutationResult object if successful otherwise an exception with details for the reason the operation failed.

Throws

- Documented

  - DocumentNotFoundException (#101)
  - CasMismatchException (#9)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)

### Sub-Document Operations

### LookupIn

Allows the chaining of Sub-Document fetch operations like, Get("path") and Exists("path") into a single atomic fetch.

Signature

```csharp
ILookupInResult LookupIn(string id, LookupInSpec[] specs, [LookupInOptions])
```

Parameters

- Required
  - Id: string - primary key as a string.
  - LookupInSpec - an array of fetch operations - requires at least one:
    - Exists - checks for the existence of a field given a path
      - Required
        - String path - path to the element
      - Optional
        - Boolean Xattr - operation is done on an Extended Attribute.
    - Get - fetches an element's value given a path
      - Notes
        - Performing this operation with a blank path will execute an full-document get operation.
      - Required
        - String path - path to the element
      - Optional
        - Boolean Xattr - operation is done on an Extended Attribute.
    - Count - gets the count of a list or dictionary element given a path
      - Required
        - String path - path to the element
      - Optional
        - Boolean Xattr - operation is done on an Extended Attribute.
- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Serializer - a custom serializer for converting the memcached JSON data to a native type.
  - Boolean AccessDeleted maps to 0x04 otherwise omitted - private; used to access XAttrs from a deleted document

Returns

- A ILookupInResult with results of the operations listed in the LookupInSpec object.

Throws

NOTE: The following exceptions are thrown at the "top" level. The errors that are thrown on a per-field basis are covered below.

- Documented

  - DocumentNotFoundException (#101)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - RequestCanceledException (#2)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurableWriteReCommitInProgressException (#111)

Per-Field errors:

- Documented

  - SubdocException (#126)

- Undocumented
  - PathNotFound (#113)
  - PathMismatch (#114)
  - PathInvalid (#115)
  - PathTooDeep (#116)
  - DocumentTooDeep (#117)
  - ValueTooDeep (#118)
  - DocumentNotJson (#120)
  - Xattr (#124)

Notes

- Do not implement the singular Sub-Doc commands as in SDK 2.0, there is a very tiny performance penalty and only adds 3 bytes to the packet size while causing quite a bit of complexity on the client. Instead only implement the "multi" Sub-Doc operations.

- Unlike MutateIn, LookupIn operations can partially fail, every path will receive a result whether or not it failed or succeeded. In the event of a failure, the exception should be propagated when the error is detected on the call to ContentAs(...). In this way, operations that succeed can still be accessed in the event of another operation failing.

- The server requires that all Xattr operations must come before any regular operations when sent to the server; however, when the results are returned from the server the previous order specified by the User must be returned so that ContentAs(int index) can be used to fetch the correct result. A data structure may be used internally to "remember" this ordering.

LookupIn Macro Values

Unlike MutateIn macros, below, these are simple strings and the app still needs to explicitly set the xattrs flag for them.

This isn't the full set of LookupIn macros exposed by the server, just the ones that are going to be most useful to the application.  The excluded macros are: datatype, vbucket_uuid, flags, value_crc32.

| Name           | String Value            |
| -------------- | ----------------------- |
| Document       | $document               |
| ExpiryTime     | $document.exptime       |
| CAS            | $document.CAS           |
| SeqNo          | $document.seqno         |
| LastModified   | $document.last_modified |
| IsDeleted      | $document.deleted       |
| ValueSizeBytes | $document.value_bytes   |
| RevId          | $document.revid         |

### MutateIn

Allows the chaining of Sub-Document mutation operations on a specific document in a single atomic transaction.

Signature

```csharp
IMutateInResult MutateIn(string id, MutateInSpec[] specs, [MutateInOptions])
```

Parameters

- Required
  - Id: string - primary key as a string.
  - MutateInSpec - an array of mutation Sub-Document operations
    - Insert - inserts an element into a document, failing if it exists
      - Notes
        - A blank path specified for an Insert operation must cause an invalid arguments error when attempting to execute the spec.
      - Required
        - String path - path to the elements
        - T value - the value to insert
      - Optional
        - PathFlags enumerations - see [Sub Document Command Spec](https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit#heading=h.ywgf7z5mjdb5)
          - Boolean CreatePath - maps to 0x01 if true, otherwise omitted - create the path if it doesn't exist
          - Boolean Xattr - maps to 0x04 if true, otherwise omitted - operation is done on an extended attribute.
          - Boolean ExpandMacroValues - Indicates that the server should expand any macros before storing the value. Infers that XAttr is also set; for internal diagnostic use only and is an unsupported feature in itself. The SDK should expose this publically, but instead set it when a Mutation Macroe value is detected.
    - Upsert - inserts an element  into a document, overriding the value if it exists
      - Notes
        - A blank path specified for an Upsert operation must cause an invalid arguments error when attempting to execute the spec.
      - Required
        - String path - path to the elements
        - T value - the value to upsert
      - Optional
        - PathFlags - see [Sub Document Command Spec](https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit#heading=h.ywgf7z5mjdb5)
          - Boolean CreatePath - maps to 0x01 if true, otherwise omitted - create the path if it doesn't exist
          - Boolean Xattr - maps to 0x04 if true, otherwise omitted - operation is done on an extended attribute.
    - Replace - replaces an element  in a document, failing if it does not exist
      - Notes
        - A blank path specified for an Upsert operation must trigger an sub-document op-code change to LCB_SET as opposed to LCB_SUBDOC_DICT_ADD.
      - Required
        - String path - path to the elements
        - T value - the value to insert
      - Optional
        - Xattr (0x04)- operation is done on an extended attribute.
    - Remove - removes an element in a document
      - Notes
        - A blank path specified for an Remove operation must trigger an sub-document op-code change to CMD_DELETE as opposed to CMD_SUBDOC_DELETE.
      - Required
        - String path - path to the element to remove
      - Optional
        - Xattr (0x04)- operation is done on an extended attribute.
    - ArrayAppend - Inserts multiple values to the end of an array element in a document.
      - Required
        - String path - path to the elements
        - T[] values - the values to insert
      - Optional
        - PathFlags enumerations - see [Sub Document Command Spec](https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit#heading=h.ywgf7z5mjdb5)
          - Boolean CreatePath - maps to 0x01 if true, otherwise omitted - create the path if it doesn't exist
          - Boolean Xattr - maps to 0x04 if true, otherwise omitted - operation is done on an extended attribute.
    - ArrayPrepend - Inserts multiple values to the beginning of an array element in a document
      - Required
        - String path - path to the elements
        - T[] values - the values to insert
      - Optional
        - PathFlags - see [Sub Document Command Spec](https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit#heading=h.ywgf7z5mjdb5)
          - Boolean CreatePath - maps to 0x01 if true, otherwise omitted - create the path if it doesn't exist
          - Boolean Xattr - maps to 0x04 if true, otherwise omitted - operation is done on an extended attribute.
    - ArrayInsert - Inserts multiple values to an array element in a document given an index
      - Required
        - String path - path to the elements
        - T[] values - the value to insert
      - Optional
        - PathFlags - see [Sub Document Command Spec](https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit#heading=h.ywgf7z5mjdb5)
          - Boolean CreatePath - maps to 0x01 if true, otherwise omitted - create the path if it doesn't exist
          - Boolean Xattr - maps to 0x04 if true, otherwise omitted - operation is done on an extended attribute.
    - ArrayAddUnique - adds a value into an array element if the value does not already exist
      - Required
        - String path - path to the elements
        - T value - the value to insert. The value to be inserted must be a primitive: null, string, boolean (true/false), or a number. Objects are not allowed as there is no way to easily determine the uniqueness of an object. 
      - Optional
        - PathFlags - see [Sub Document Command Spec](https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit#heading=h.ywgf7z5mjdb5)
          - Boolean CreatePath - maps to 0x01 if true, otherwise omitted - create the path if it doesn't exist
          - Boolean Xattr - maps to 0x04 if true, otherwise omitted - operation is done on an extended attribute.
    - Increment - performs an arithmetic increment or decrement on a numeric element within a document.
      - Required
        - String path - path to the elements
        - uint64 delta - the amount to increase the value by.
      - Optional
        - PathFlags - see [Sub Document Command Spec](https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit#heading=h.ywgf7z5mjdb5)
          - Boolean CreatePath - maps to 0x01 if true, otherwise omitted - create the path if it doesn't exist
          - Boolean Xattr - maps to 0x04 if true, otherwise omitted - operation is done on an extended attribute.
    - Decrement - performs an arithmetic increment or decrement on a numeric element within a document.
      - Required
        - String path - path to the elements
        - uint64 delta - the amount to decrease the value by.
      - Optional
        - PathFlags - see [Sub Document Command Spec](https://docs.google.com/document/d/1vaQJxIA5nhWJqji7X2R1xQDZadb5PabfKAid1kVe65o/edit#heading=h.ywgf7z5mjdb5)
          - Boolean CreatePath - maps to 0x01 if true, otherwise omitted - create the path if it doesn't exist
          - Boolean Xattr - maps to 0x04 if true, otherwise omitted - operation is done on an extended attribute.
- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Expiry - the length of time the  document will be stored in Couchbase before being evicted.
  - CAS - Compare and Set value for optimistic locking of a document.
  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority
  - StoreSemantics - the storage action
    - Replace - replace the document, fail if it doesn't exist
    - Upsert - replace the document or create it if it doesn't exist (0x01)
    - Insert - create document, fail if it exists (0x02)
  - Boolean AccessDeleted maps to 0x04 otherwise omitted - private; used to access XAttrs from a deleted document
  - Serializer - a custom serializer for converting the memcached JSON data to a native type.

Returns

- An IMutateInResult object with the results of a Sub-Document operations listed in the MutateInSpec object.

Throws

- Documented

  - DocumentNotFoundException (#105)
  - DocumentExistsException (#101)
  - CasMismatchException (#9)
  - RequestTimeoutException (#1)
  - SubdocException (#126)
  - CouchbaseException (#0)

- Undocumented
  - ValueTooLargeException (#104)
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)
  - PathNotFound (#113)
  - PathMismatch (#114)
  - PathInvalid (#115)
  - PathTooDeep (#116)
  - DocumentTooDeep (#117)
  - ValueTooDeep (#118)
  - CannotInsertValue (#119)
  - DocumentNotJson (#120)
  - NumberTooBig (#121)
  - DeltaRange (#122)
  - PathExists (#123)
  - Xattr (#124)

Notes

- Do not implement the singular Sub-Doc commands as in SDK 2.0, there is a very tiny performance penalty and only adds 3 bytes to the packet size while causing quite a bit of complexity on the client. Instead only implement the "multi" Sub-Doc operations.

- Multi-Mutations are atomic; either all commands succeed or an exception is thrown indicating that the command or one of its children Sub-Document commands failed.

- The server requires that all Xattr operations must come before any regular operations when sent to the server; however, when the results are returned from the server the previous order specified by the User must be returned so that ContentAs(int index) can be used to fetch the correct result. A data structure may be used internally to "remember" this ordering.

- If CAS is non-zero and KEY_EXISTS is returned by the server a CasMismatchException should be propagated.

Mutation Macro Values

| Name        | String Value               |
| ----------- | -------------------------- |
| CAS         | "${Mutation.CAS}"          |
| SeqNo       | "${Mutation.seqno}"        |
| ValueCrc32c | "${Mutation.value_crc32c}" |

There are currently a fixed number of Mutation Macros supported by the server; instead of exposing an open string parameter for the supported Macros, a "typed" value is exposed instead which the SDK can detect and correlate with the actual Macro string.

There are a number of ways to do this, but a very simple way is to define a marker interface and simple implementations for each of the values in the table above where ToString() or its equivalent is called if the interface is detected and then taking the string value and setting the XAttr flag as it is required for macros.

Mutation Macro Example

```csharp
interface IMutationMacro{}
  public class CasMutation implements IMutationMacro {
  public override string toString(){return "${Mutation.CAS}";}

  ...

}

public class MutationMacro {
  public static IMutationMacro Cas => new CasMutationMacro();

  ...

}

//later on

if(value is IMutationMacro){
  XAttr = true;
  value = value.ToString();
}
```

The SDK can then detect the marker interface, fetch the string value, set the Xattr flag and then send the packet to the server.

### Replica Reads

#### GetAnyReplica

Gets a document for a given id, leveraging both the active and all available replicas. This method follows the same semantics of GetAllReplicas (including the fetch from ACTIVE), but returns the first response as opposed to returning all responses.

Signature

```csharp
IGetReplicaResult GetAnyReplica(string id, [options])
```

Parameters

- Required

  - Id: string - the primary key.

- Optional

  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Transcoder - a custom transcoder for converting the memcached packet to a native type.

- Throws
  - Documented
    - DocumentUnretrievableException (#125)
      - The situation where we got responses from all of them (most likely: key not found) but none of them were SUCCESS so we ended up not returning anything (empty stream from get all replicas).
    - RequestTimeoutException (#1)
    - CouchbaseException (#0)
  - Undocumented
    - InvalidArgumentException (#3)

Returns

- The JSON object or scalar encapsulated in a IGetReplicaResult API object.

Throws

- Any exceptions raised by the underlying platform

#### GetAllReplicas

Gets a stream of document data from the server, leveraging both the active and all available replicas; the caller can choose which replica they wish to take and cancel the stream at any time.  This MUST fetch from both the active (using a standard GET opcode) and all replicas (via the usual GET_REPLICA opcode) due to failover situations that can arise.  In addition, the SDK MUST broadcast the requests simultaneously to ensure the application receives data as quickly as possible.

```csharp
Stream<IGetReplicaResult> GetAllReplicas(string id, [options])
```

Parameters

- Required

  - Id: string - the primary key.

- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Transcoder - a custom transcoder for converting the memcached packet to a native type.

Returns

A stream of replicas as a IGetReplicaResult object

Throws

- Documented

  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - InvalidArgumentException (#3)

### Other Key/Value Methods

#### Unlock

Unlocks a document pessimistically locked by a GetAndLock operation.

Signature

```csharp
void  Unlock(string id, uint64 cas, [options])
```

Parameters

- Required

  - Id: string - the document id
  - CAS: uint64 - the CAS from the GetAndLock operation

- Options
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.

Returns

- Nothing

Throws

- Documented
  - DocumentNotFoundException (#101)
  - CasMismatchException (#9)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)
- Undocumented
  - RequestCanceledException (#2)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurableWriteReCommitInProgressException (#111)

#### Touch

Updates the expiration a document given an id, without modifying or returning its value.

Signature

```csharp
IResult Touch(string id, duration expiry, [options])
```

Parameters

- Required

  - Id: string - the primary key
  - Expiry - the time to extend the document life by.

- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.

Returns

- An IResult object indicating the status of the operation.

Throws

- Documented
  - DocumentNotFoundException (#101)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)
- Undocumented
  - DocumentLockedException (#103)
  - RequestCanceledException (#2)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurableWriteReCommitInProgressException (#111)

#### Exists

Returns true if a document exists for a given id, otherwise false.

Signature

```csharp
IExistsResult Exists(string id, [options])
```

Parameters

- Required

  - Id: string - the id of the document

- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.

Returns

- An IExistsResult object with a boolean value indicating the presence of the document.

Throws

- Documented
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)
- Undocumented
  - RequestCanceledException (#2)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)

Notes

- If the server status is success, but the returned deleted flag in extras is true, then Exists should return false as the document has been deleted.

#### GetAndTouch

Gets a document for a given id and extends its expiration.

Signature

```csharp
IGetResult GetAndTouch(string id, Duration expiry, [options])
```

Parameters

- Required

  - Id: string - the primary key
  - Expiry - the length of time the  document will be stored in Couchbase before being evicted.

- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Transcoder - a custom transcoder for converting the memcached packet to a native type.

Returns

- The JSON object or scalar encapsulated in a IGetResult API object.

Throws

- Documented

  - DocumentNotFoundException (#101)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurableWriteReCommitInProgressException (#111)

#### GetAndLock

Gets a document for a given id and places a pessimistic lock on it for mutations

Signature

```csharp
IGetResult GetAndLock(string id, Duration lockTime, [options])
```

Parameters

- Required

  - Id: string - the primary key
  - LockTime: int/Duration - the length of time the lock will be held on the document.

- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Transcoder - a custom transcoder for converting the memcached packet to a native type.

Returns

The JSON object or scalar encapsulated in a [IGetResult](https://docs.google.com/document/d/1nuy8EUvgpZnijxEuItDHLu51tP93JOO6m3EqkIFDdas/edit#heading=h.z2f3vg34lbf2) API object.

Throws

- Documented

  - DocumentNotFoundException (#101)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurableWriteReCommitInProgressException (#111)

## The IBinaryCollection Interface

A special collection for non-JSON operations:

```csharp
interface IBinaryCollection{
  IMutationResult Append(string id, byte[] values, AppendOptions options = null);
  IMutationResult Prepend(string id, byte[] values, PrependOptions options =  null);
  ICounterResult Increment(string id, IncrementOptions options =  null);
  ICounterResult Decrement(string id, DecrementOptions options =  null);
}
```

### Creation

```csharp
var bucket = cluster.Bucket("companies");
var scope = bucket.Scope("ACME");
var collection = scope.Collection("users");

var result = collection.Binary().Increment("thekey");
```

### Methods

#### Increment

Increments a counter by a value, creating the initial value if it doesn't exist.

Signature

```csharp
ICounterResult Increment(string id, [options])
```

Parameters

- Required

  - Id: string - the primary key

- Options
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Expiry - the length of time the  document will be stored in Couchbase before being evicted.
  - Delta - the number to increment the key by. Default to 1.
  - Initial - the initial value to use. If the key doesn't exist, this value will be returned.
  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority

Returns

An ICounterResult object with the status of the operation.

Throws

- Documented

  - DocumentNotFoundException (#101)
  - CasMismatchException (#9)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)

#### Decrement

Decrements a counter by a value, creating the initial value if it doesn't exist.

Signature

```csharp
ICounterResult Decrement(string id, [options])
```

Parameters

- Required

  - Id: string - the primary key

- Optional
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - Expiry - the length of time the  document will be stored in Couchbase before being evicted.
  - Delta - the number to decrement the key by. Default to 1.
  - Initial - the initial value to use. If the key doesn't exist, this value will be returned.
  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority

Returns

- An ICounterResult object with the status of the operation.

Throws

- Documented

  - DocumentNotFoundException (#101)
  - CasMismatchException (#9)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)

#### Append

Appends a value to a given id.

Signature

```csharp
IMutationResult Append(string id, byte[] values, [options])
```

Parameters

- Required

  - Id: string - the primary key
  - Value - a byte array

- Options
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - CAS - Compare and Set value for optimistic locking of a document.
  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority

Returns

- An IMutationResult object if successful otherwise an exception with details for the reason the operation failed.

Throws

- Documented

  - DocumentNotFoundException (#101)
  - CasMismatchException (#9)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - ValueTooLargeException (#104)
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)

#### Prepend

Prepends a value to a given id.

Signature

```csharp
IMutationResult Prepend(string id, byte[] bytes, [options])
```

Parameters

- Required

  - Id: string - the primary key
  - Value - a byte array

- Options
  - Timeout or timeoutMillis (int/duration) - the time allowed for the operation to be terminated. This is controlled by the client.
  - CAS - Compare and Set value for optimistic locking of a document.
  - Durability
    - Either
      - ReplicateTo - the durability requirement for replication.
      - PersistTo - the durability requirement for persistence.
    - Or
      - DurabilityLevel - Majority, MajorityAndPersistToActive or PersistToMajority

Returns

- An IMutationResult object if successful otherwise an exception with details for the reason the operation failed.

Throws

- Documented

  - DocumentNotFoundException (#101)
  - CasMismatchException (#9)
  - RequestTimeoutException (#1)
  - CouchbaseException (#0)

- Undocumented
  - ValueTooLargeException (#104)
  - RequestCanceledException (#2)
  - DocumentLockedException (#103)
  - InvalidArgumentException (#3)
  - TemporaryFailureException (#7)
  - ServerOutOfMemoryException (#102)
  - DurabilityAmbiguousException (#109)
  - DurabilityImpossibleException (#108)
  - DurabilityLevelNotAvailableException (#107)
  - DurableWriteInProgressException (#110)
  - DurableWriteReCommitInProgressException (#111)

## Enums

#### DurabilityLevel

The DurabilityLevel enum allows specification of the level of durability for an operation.

```csharp
public enum DurabilityLevel{
  Majority (0x01),
  MajorityAndPersistToActive (0x02),
  PersistToMajority(0x03)
}
```

## Return Types

### Binary Memcached Return Types

#### IResult Interface

A base interface for return types; Unlock and Lock use as a result type.

```csharp
public interface IResult{
  Uint64 Cas();
}
```

#### IExistsResult

A result for checking the existence of a document in Couchbase Server for a given id.

```csharp
public interface IExistsResult : IResult {
  Boolean Exists();
}
```

#### IGetResult Interface

The IGetResult interface is the return type for full read operations.

```csharp
public interface IGetResult : IResult {
  T ContentAs<T>();
  Optional<PointInTime> ExpiryTime();
}
```

##### ExpiryTime

The ExpiryTime() method returns the point in time when the document expires.

It is up to the SDK implementer to select a semantic type that accurately represents a point in time. In Java, this is an Instant. In other languages, it might be a DateTime or something similar. Be sure to select a type that can be converted back into an epoch second without requiring external knowledge of the correct time zone.

Note that types like Duration or TimeSpan are not appropriate, since they do not represent an absolute point in time. A type like Java's LocalDateTime would also not be appropriate, since it cannot be converted back into an epoch second without knowing which time zone to use.

The value returned by Couchbase Server is the number of seconds between January 1, 1970 (UTC) and the time the document expires. Languages without an appropriate semantic type should return the value from the server as a 64-bit integer.

If the Get request was made without the WithExpiry option, or if the document has no expiry, then ExpiryTime() returns an empty Optional (or zero, if the language does not have an appropriate semantic type).

#### IGetReplicaResult Interface

The IGetReplicaResult interface is the return type for full read operations.

```csharp
public interface IGetReplicaResult : IResult {
  T ContentAs<T>();
  Boolean IsReplica();
}
```

#### ILookupInResult Interface

The ILookupInResult interface is the return type for Sub-Document read operations.

```csharp
public interface ILookupInResult : IResult {
  T ContentAs<T>(int lookupIndex);
  Boolean Exists(int lookupIndex);
}
```

- Exists
  1. Throws InvalidIndexException if the index is < 0 or >= the number of lookups.
  2. Returns true if the status code at the index is `SUCCESS`.
  3. Returns false if the status code at the index is `PATH_NOT_FOUND`.
  4. For all other status codes, throws an exception matching the status code. See [RFC-0058](0058-error-handling.md) for error definitions.

- ContentAs
  1. Throws InvalidIndexException if the index is < 0 or >= the number of lookups.
  2. Gets the value at the index, in one of two ways depending on the lookup type:
     1. If the result came from an `exists` lookup, calls Exists to get a boolean, and uses the JSON representation of that boolean as the "value" in the next step. Propagates any exceptions thrown by `Exists`.
     2. Otherwise, uses the value returned by the server as the "value" in the next step. If the status code at the index is not `SUCCESS`, throws an exception matching the status code. See [RFC-0058](0058-error-handling.md) for error definitions. 
  3. Returns the value at the index, converted to the requested type.

#### IMutationResult Interface

The IMutationResult interface is the return type for K/V write operations from the servers.

```csharp
public interface IMutationResult : IResult {
  Optional<MutationToken> MutationToken();
}
```

#### IMutateInResult Interface

The IMutateInResult is the return type for Sub-Document write operations on a document.

```csharp
public interface IMutateInResult : IMutationResult {
  T ContentAs<T>(int mutateIndex);
}
```

#### ICounterResult Interface

The ICounterResult is the return type for Increment and Decrement.

```csharp
public interface ICounterResult : IMutationResult {
  Uint64 Content();
}
```

# Projections using Get

A new feature of Get is to allow "projections" of document fragments for a given path. Based on Sub-Doc API, the maximum number of fields that can be projected is 16. Once the 16 field limit is reached, the entire document will be fetched using the standard Get command and the requested fields will be extracted.

Given a JSON document with the following structure, the table below it shows the expected inputs/outputs of the API and how an SDK should handle a given path:

```json
{
  "name": "Emmy-lou Dickerson",
  "age": 26,
  "animals": ["cat", "dog", "parrot"],
  "attributes": {
    "hair": "brown",
    "dimensions": {
      "height": 67,
      "weight": 175
    },
    "hobbies": [
      {
        "type": "winter sports",
        "name": "curling"
      },
      {
        "type": "summer sports",
        "name": "water skiing",
        "details": {
          "location": {
            "lat": 49.28273,
            "long": -123.120735
          }
        }
      }
    ]
  }
}
```

Paths that return errors should be silently omitted, in the same way that a standard subdocument lookup operation would behave.

### Examples

#### `"name"`

```json
{ "name": "Emmy-lou Dickerson" }
```

#### "age"

```json
{ "age": 26 }
```

#### "animals"

```json
{ "animals": ["cat", "dog", "parrot"] }
```

#### `"name","age","idontexist"`

```json
{ "name": "Emmy-lou Dickerson", "age": 26 }
```

#### "animals[0]"

```json
{ "animals": ["cat"] }
```

#### "animals[1]"

```json
{ "animals": ["dog"] }
```

#### "animals[2]"

```json
{ "animals": ["parrot"] }
```

#### "attributes"

```json
{
  "attributes": {
    "hair": "brown",
    "dimensions": {
      "height": 67,
      "weight": 175
    },
    "hobbies": [
      {
        "type": "winter sports",
        "name": "curling"
      },
      {
        "type": "summer sports",
        "name": "water skiing",
        "details": {
          "location": {
            "lat": 49.28273,
            "long": -123.120735
          }
        }
      }
    ]
  }
}
```

#### "attributes.hair"

```json
{ "attributes": { "hair": "brown" } }
```

#### "attributes.dimensions"

```json
{
  "attributes": {
    "dimensions": {
      "height": 67,
      "weight": 175
    }
  }
}
```

#### "attributes.dimensions.height"

```json
{
  "attributes": {
    "dimensions": {
      "height": 67
    }
  }
}
```

#### "attributes.dimensions.weight"

```json
{
  "attributes": {
    "dimensions": {
      "weight": 175
    }
  }
}
```

#### "attributes.hobbies"

```json
{
  "attributes": {
    "hobbies": [
      {
        "type": "winter sports",
        "name": "curling"
      },
      {
        "type": "summer sports",
        "name": "water skiing",
        "details": {
          "location": {
            "lat": 49.28273,
            "long": -123.120735
          }
        }
      }
    ]
  }
}
```

#### "attributes.hobbies[0].type"

```json
{
  "attributes": {
    "hobbies": [
      {
        "type": "winter sports"
      }
    ]
  }
}
```

#### "attributes.hobbies[1].type"

```json
{
  "attributes": {
    "hobbies": [
      {
        "type": "summer sports"
      }
    ]
  }
}
```

#### "attributes.hobbies[0].name"

```json
{
  "attributes": {
    "hobbies": [
      {
        "name": "curling"
      }
    ]
  }
}
```

#### "attributes.hobbies[1].name"

```json
{
  "attributes": {
    "hobbies": [
      {
        "name": "water skiing"
      }
    ]
  }
}
```

#### "attributes.hobbies[1].details"

```json
{
  "attributes": {
    "hobbies": [
      {
        "details": {
          "location": {
            "lat": 49.28273,
            "long": -123.120735
          }
        }
      }
    ]
  }
}
```

#### "attributes.hobbies[1].details.location"

```json
{
  "attributes": {
    "hobbies": [
      {
        "details": {
          "location": {
            "lat": 49.28273,
            "long": -123.120735
          }
        }
      }
    ]
  }
}
```

#### "attributes.hobbies[1].details.location.lat"

```json
{
  "attributes": {
    "hobbies": [
      {
        "details": {
          "location": {
            "lat": 49.28273
          }
        }
      }
    ]
  }
}
```

#### "attributes.hobbies[1].details.location.long"

```json
{
  "attributes": {
    "hobbies": [
      {
        "details": {
          "location": {
            "long": -123.120735
          }
        }
      }
    ]
  }
}
```

#### "attributes"

Invalid operation

#### "attributes.hobbies"

Invalid operation

#### "attributes.hobbies[1]"

Invalid operation

#### "attributes.hobbies"

Invalid operation

# Changelog

- Sept 12, 2019 - Revision #1

  - Initial Draft

- Sept 20, 2019 - Revision #2 (by Brett Lawson)

  - Corrected some errors in the RFC meta-data.
  - Added missing change-log and sign-off sections to RFC.
  - Added a common Enumerations section in the document and moved all common enumeration definitions to this singular location..
  - Changed all instances of Expiration to Expiry.
  - Removed Transcoder from all result objects (it is now uniformly specified through options blocks).
  - Corrected LookupIn/MutateIn to use JSONSerializer rather than Transcoder.
  - Removed WithExpiry option from LookupIn which was included in error, also moved expiry from IResult to IGetResult as it is now specific to Get operations.
  - Removed InsertFull and UpsertFull sub-document operations.
  - Added wording to sub-document insert/upsert/replace explaining how blank-paths behave with regard to full-document operations.
  - Removed duplicate options blocks definitions.
  - Modified GetAndLock lockTime to use Duration rather than int for the period of time to make it more consistent with other representations of duration in the SDK.
  - Removed Transcoder option from Unlock and Remove as they do not transcode.

- Sept 23, 2019 - Revision #3 (by Brett Lawson)

  - Removed serializer option which was accidentally still included in a few random mutate-in operations.
  - Removed GetFull from sub-document lookup-in operation.
  - Added wording to sub-document get operation explaining how blank-paths behave with regards to full-document operations.
  - Adjusted mutateIn full-document value operation to be Replace rather than Upsert, as the semantics more closely reflect the true behaviour of the API.

- Sept 27, 2019 - Revision #4 (by Jeff Morris)

  - Replaced boolean flags for doc creation with StoreSemantics enumeration
  - Changed IGetReplicaResult::IsMaster() to IGetReplicaResult::IsReplica().

- Oct 1, 2019 - Revision #5 (by Brett Lawson)

  - Corrected the change above with mutateIn full-document replace which was incorrectly moved from upsert to insert rather than upsert to replace.

- Oct 1, 2019 - Revision #6 (by Jeff Morris)

  - Replace long with duration for expiry parameter of Touch(...)

- Oct 9, 2019 - Revision #7 (by Jeff Morris)

  - Removed CollectionOptions from Collection method
  - Accepted change to MutateIn Insert and Upsert
  - Accepted change to GetAnyReplica intro paragraph
  - Accepted change to GetAllReplicas intro paragraph
  - Changed wording of Id parameter of Unlock

- Oct 10, 2019 - Revision #8 (by Jeff Morris/Sergey

  - Changed byte[] bytes to byte[] values in method signatures
  - Move Unlock from replica sub-section to "Other methods" sub-section.

- Oct 19, 2019 - Revision #9 (by Jeff Morris)

  - Added error handling details for LookupIn and MutateIn.
  - Added error handling details for CAS
  - Re-added Get Projection details and example table.
  - Specify XAttr order/re-ordering wrt to non-XAttr ops

- Nov 8, 2019 - Revision #10 (by Jeff Morris)

  - Added volatile to ICollection/Scope
  - Renamed spec to spec(s)
  - Added detail to xattr macros

- Nov 18, 2019 - Revision #11 (by Michael Nitchinger & Brett Lawson)

  - Added throws information to all methods
  - Added DocumentUnretrievableException to getAnyReplica
  - Unlock documents CasMismatchException
  - Renamed/Clarified MajorityAndPersistToActive in the full document

- Nov 18, 2019 - Revision #12 (by Brett Lawson)

  - Moved RFC to REVIEW state.
  - Added  LookupIn macro section

- Dec 12, 2019 - Revision #13 (by Brett Lawson, Jeff Morris)

  - Rename MajorityAndPersistOnMaster to MajorityAndPersistToActive
  - Rename any occurance of "master" with "active"

- Jan 1, 2020 - Revision #14 (by Jeff Morris)

  - Changed the definition of IMutationResult so that it is clear a status is not included in the result and that an exception or error is returned if a failure happens.

- Feb 12, 2020 - Revision #15 (by Jeff Morris)

  - Updated Exists with details on how to determine deleted status.

- April 30, 2020

  - Moved RFC to ACCEPTED state.

- July 23, 2020 - Revision #16 (by David Nault)

  - Updated IGetResult: Renamed Expiry() to ExpiryTime(). The return type must represent a point in time (not a duration), and must not require external knowledge of the time zone.

- February 10, 2021 - Revision #17 (by Charles Dixon)

  - Updated subdocument remove to allow performing the operation with a blank path.

- June 9, 2021 - Revision #18 (by Jeff Morris)

  - Remove CAS from Increment as it is not supported by the server for that operation.

- September 20, 2021 (by Brett Lawson)

  - Converted to Markdown.
  - Fixed a few typos.
  - Restructured a couple of sections for clarity (no content altered).

- April 4, 2023 - Revision #19 (by David Nault)

  - Clarified behavior of `ILookupInResult.Exists` and `ILookupInResult.ContentAs`.

- June 17, 2025 - Revision #20 (by Matt Wozakowski)

  - Fix Touch signature return type from `IMutationResult` to `IResult`

  # Signoff

| Language   | Team Member         | Signoff Date | Revision |
| ---------- | ------------------- | ------------ | -------- |
| Node.js    | Brett Lawson        | 2020-04-16   | 15       |
| Go         | Charles Dixon       | 2020-04-22   | 15       |
| Connectors | David Nault         | 2020-04-29   | 15       |
| PHP        | Sergey Avseyev      | 2020-04-22   | 15       |
| Python     | Ellis Breen         | 2020-04-29   | 15       |
| Scala      | Graham Pople        | 2020-04-30   | 15       |
| .NET       | Jeffry Morris       | 2020-04-23   | 15       |
| Java       | Michael Nitschinger | 2020-04-16   | 15       |
| C          | Sergey Avseyev      | 2020-04-22   | 15       |
| Ruby       | Sergey Avseyev      | 2020-04-22   | 15       |
