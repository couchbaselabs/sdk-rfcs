# Meta

- RFC Name: SDK3 Datastructures
- RFC ID: 0050-sdk3-datastructures
- Start Date: July 28, 2019
- Owner: Brett Lawson \<brett@couchbase.com\>
- Current Status: ACCEPTED
- Relates To
  - [SDK-RFC#8](rfc/0008-datastructures.md) (Datastructures)
- [Original Google Drive Doc](https://docs.google.com/document/d/1mKk20ScVE8ssF2DvqZTe9xIUvOUanJ7LOARFiCJPkQ0)

# Motivation

A subset of Couchbase's user-base desires easy drop-in replacement for various other technologies or drop-in persistent replacements for various concurrent data-structures available in the SDKs.  In order to enable competitive representation, and continue with the feature-set already enabled by SDK 2, the decision was made to persist our data-structure support into SDK 3.

# Summary

This RFC describes information about higher level data-structures which should be implemented in all RFCs.  As opposed to the typical SDK 3 RFC which provides a specific interface which should be implemented, this RFC specifically targets providing a specific feature set with the implementation being driven by what is idiomatic to the specific SDK platform.

# Technical Details

To provide data-structures within the framework of SDK 3.0, methods should be included within the Collections interface which enable you to instantiate a concurrent data-structure object according to what is available in each language.  In Java you might expect this to be interfaces from the java.util.concurrent namespace, in .NET this might be interfaces from the System.Collections.Concurrent namespace, etc...  The remainder of this document shall be dedicated to describing the functionality which must be provided by each SDK.  Note that SDKs are only responsible for implementing (and should only implement) interfaces which are designed for concurrent use and must not block by default.  Additionally, data structures are only supported at the document level, and cannot be nested.  Lastly, all data-structures should be created lazily (only when they are first used) and all retrieval methods should make the assumption that a missing document simply indicates an empty structure.

### Error Handling

Exceptions for datastructures fall into two categories:

- Exceptions raised by the SDK that cannot be mapped into the language specific exception types based on the languages data-structures implementation.
- Exceptions which are covered by the language's data-structures implementation.

For Couchbase-Specific exceptions these errors need to be documented:

- TimeoutException (#1)
- CouchbaseException (#0)

All other documented or undocumented errors are implementation specific because they heavily depend on the language-specific interface/type they implement.

### Lists

The server-side representation of a list is a JSON array.  An instance of this type is returned via a non-blocking, no-network call to ICollection::list(key).  The following is a list of operations which must be supported upon lists.

#### Iteration

Implemented by fetching the entire list and yielding a result per value in the list:

```typescript
items = mc_get($doc)
for item in items:
  yield item
```

#### Indexed Access

Implemented via a subdocument element access:

```typescript
return mc_lookupin_get($doc, "[" + $index + "]");
```

#### Indexed Removal

Implemented via a subdocument element removal:

```typescript
mc_mutatein_remove($doc, "[" + $index + "]");
```

#### Append

Implemented via a subdocument array append:

```typescript
mc_mutatein_array_append($doc, $value);
```

#### Prepend

Implemented via a subdocument array prepend:

```typescript
mc_mutatein_array_prepend($doc, $value);
```

#### IndexOf

Implemented by fetching the whole document and finding the appropriate item (note that depending on language specifics, this should support specifying a starting index):

```typescript
items = mc_get($doc)

for i, item in items:\
  if i < $startingIndex: continue
  if item === $value:
    return i
return -1
```

#### Size

Implemented via a subdocument get count:

```typescript
mc_lookupin_get_count($doc);
```

#### Clear

Implemented via a document remove:

```typescript
mc_remove($doc);
```

### Maps

The server-side representation of a map is a JSON object.  An instance of this type is returned via a non-blocking, no-network call to ICollection::map(key).  The following is a list of operations which must be supported for maps.

#### Iteration

Implemented by fetching the entire list and yielding a compound object per value in the map:

```typescript
items = mc_get($key)
for k, v in items:
  yield {k, v}
```

#### Keyed Access

Implemented using a subdocument get operation:

```typescript
return mc_lookupin_get($doc, $key);
```

#### Keyed Insertion

Implemented using a subdocument set operation:

```typescript
mc_mutatein_mapset($doc, $key, $value);
```

#### Keyed Removal

Implemented using a subdocument removal operation:

```typescript
mc_mutatein_mapremove($doc, $key);
```

#### Exists

Implemented by using a subdocument exists operation:

```typescript
return mc_lookupin_exists($doc, $key);
```

#### Size

Implemented via a subdocument get count operation:

```typescript
mc_lookupin_get_count($doc);
```

#### Keys

Implemented by fetching the entire document and extracting the keys:

```typescript
items = mc_get($key)\
return KEYS_OF(items)
```

#### Values

Implemented by fetching the entire document and extracting the values:

```typescript
items = mc_get($key)\
return VALUES_OF(items)
```

#### Clear

Implemented via a document remove:

```typescript
mc_remove($doc);
```

### Sets

The server-side representation of a map is an unsorted JSON list.  An instance of this type is returned via a non-blocking, no-network call to ICollection::set(key).  The following is a list of operations which must be supported for sets.

Note: Sets are restricted to containing primitive types only due to server-side comparison restrictions.

#### Iteration

See implementation of List::Iteration

#### Add

Implemented using a subdocument array add unique operation:

```typescript
mc_mutatein_array_add_unique($doc, $value);
```

#### Remove

Implemented by fetching the entire document, identifying the item to remove, and then performing a CAS based removal of the singular entry, retrying on CAS failures:

```typescript
while !timedOut():
  items = mc_get($doc)
  for i, item in items:
    if item === $value:
      try:
        mc_mutatein_remove($doc, "[" + $i + "]", cas: cas)
      catch CasMismatch:
        continue
      catch ...:
        throw
      return
throw TimeoutException()
```

#### Values

Implemented by fetching the entire document and returning it (as it should already be an array):

```typescript
return mc_get($key);
```

#### Contains

Implemented by fetching the entire document and searching for the requested value:

```typescript
items = mc_get($doc)
for i, item in items:
  if item === $value:
    return true
return false
```

#### Size

See implementation of List::Size

#### Clear

Implemented via a document remove:

```typescript
mc_remove($doc);
```

### Queues

The server-side representation of a queue is a JSON list.  An instance of this type is returned via a non-blocking, no-network call to ICollection::queue(key).  Queues are implemented similarly to lists, but include a pop method for removing an item from the queue. Queues are a FIFO structure.  The following is a list of operations which must be supported for queues.

#### Iteration

See implementation of List::Iteration

#### Push

See implementation of List::Prepend

#### Pop

Implemented by performing a subdocument get on "[-1]", and then performing a CAS-based subdocument removal with a path of "[-1]", retrying on CAS failures:

```typescript
while !timedOut():
  value, cas = mc_lookupin_get($doc, "[-1]")
  try:
    mc_mutatein_remove($doc, "[-1]", cas: cas)
  catch CasMismatch:
    continue
  catch ...:
    throw
  return
throw TimeoutException()
```

#### Size

See implementation of List::Size

#### Clear

Implemented via a document remove:

```typescript
mc_remove($doc);
```

# Changelog

- July 28, 2019 - Revision #1 (by Brett Lawson)

  - Initial Draft

- Sept 11, 2019 - Revision #2 (by Brett Lawson)

  - Added wording to describe how instances of the interfaces described in this document are accessed from the SDK API.
  - Clarified the continue/break levels for some of the retry loops.
  - Corrected an error in the example implementation for Set::values()
  - Added missing Keyed Access and Keyed Removal to Maps.
  - Updated example code for operations which must do CAS-retries to include a 16 attempt maximum rather than indefinite attempts.
  - Corrected some minor typos.
  - Removed useless questions section.

- Sept 19, 2019 - Revision #3 (by Brett Lawson)

  - Add wording to clarify that data-structures should be created lazily and that a missing document should not error, but rather indicates an empty structure.
  - Added Clear() method to all data-structures which removes the underlying document which effectively clears the structure.

- Nov 18, 2019 - Revision #4 (by Brett Lawson)

  - Added correct pseudo-code to demonstrate the appropriate retry behaviour during CAS operations.
  - Added wording to clarify that the Queue data-structure is a FIFO structure.
  - Added more specific error handling details.
  - Added support for specifying a starting index to the indexOf method of List.

- Nov 18, 2019 (by Brett Lawson)

  - Moved RFC to REVIEW state.

- April 30, 2020 (by Brett Lawson)

  - Moved RFC to ACCEPTED state.

- Sept 17, 2021 (by Brett Lawson)

  - Converted to Markdown

# Signoff

| Language   | Team Member         | Signoff Date | Revision |
| ---------- | ------------------- | ------------ | -------- |
| Node.js    | Brett Lawson        | 2020-04-16   | 4        |
| Go         | Charles Dixon       | 2020-04-22   | 4        |
| Connectors | David Nault         | 2020-04-29   | 4        |
| PHP        | Sergey Avseyev      | 2020-04-22   | 4        |
| Python     | Ellis Breen         | 2020-04-29   | 4        |
| Scala      | Graham Pople        | 2020-04-30   | 4        |
| .NET       | Jeffry Morris       | 2020-04-22   | 4        |
| Java       | Michael Nitschinger | 2020-04-16   | 4        |
| C          | Sergey Avseyev      | 2020-04-22   | 4        |
| Ruby       | Sergey Avseyev      | 2020-04-22   | 4        |
