# Meta

- RFC Name: Datastructure Support
- RFC ID: 0008
- Start Data: 2016-07-15
- Owner: Mark Nunberg
- Current Status: ACCEPTED


# Summary

This RFC provides higher level "datastructure" support based upon the subdocument API. This includes things like language-based support for integration between Couchbase and "lists", "collections", "maps", and other complex data structures for languages which support them.

# Motivation

Data structures might provide a nicer developer experience over the CRUD APIs, and perhaps bypass document construction per-se entirely for some use cases. The trajectory for this project may eventually lead to:

- Lightweight ODMs based on Couchbase/Subdoc
- New features in the subdoc core itself
- Provide better parity with Redis where it currently has more features than Subdoc


# General Design

## High Level Functionality

To provide proper _datastructure support_, several key issues must first be identified with respect to high level functionality. This refers to things like _ordering_, _queuing_ and more.

Functionality may be divided into two categories:

* Object-level functionality. This applies for languages which can implement various _interfaces_ such as a
  generic `List` interface or a generic `Map` interface. Examples of such languages are Java and C#

* Discreet functionality. This is for languages which do not have generic interfaces to implement. In such
  cases, a `List` is a concept which can be manipulated using various discreet functions such as
  `list_append`, `list_prepend` and so on. Examples of such languages are Go and PHP.

The following sections include functionality applicable to one or both of the above categories. See below
(in API reference) for required functionality.


### Lists

Operations when using a list are fairly straightforward. The base representation of a list is a JSON array (`[]`). Items are added to the beginning or end (or at a specific index) of the array.

Here are the operations required:

#### Iteration

Fetch the entire list. When removing an item (while iterating over it), the CAS should also be used, in case the CAS of the item has changed:

```
iteration():
  Items = mc_get("key");
  For item in items: yield item
```

If a 'List' is not a first class object (i.e. datastructures are implemented using discreet functions), then it may suffice to simply document how to access the underlying JSON array.


#### Element Access


Subdoc natively supports element access:

```
__getitem__( index):
  Return subdoc_get("key", "[" + index + "]");
```


#### Size


Spock will contain support for GET_COUNT access:

```
length():
  Return subdoc_get_count("key", "");
```

Until then, SDKs should emulate this on the client side
by fetching the list and counting the elements:

```
length():
  Return len(get("key"))
```


#### Append

Native subdoc `ARRAY_APPEND`

```
append(item Object):
    subdoc_push_last("key", item)
```

#### Prepend

Native subdoc `ARRAY_PREPEND`

```
prepend(item Object):
    subdoc_array_push_first("key", item)
```

#### Index/Contains


> **NOTE**: This overlaps with the functionality of Sets

Spock will contain support for `ARRAY_CONTAINS`, which will return the index of the item

```
contains(item String):
  Rv = subdoc_contains("key", "", item);
  If rv < 0: Return false
  Return true
```


#### Memberwise Remove


> **NOTE**: This overlaps with the functionality of Sets

Use ARRAY_CONTAINS with CAS and then employ the returned index:

```
remove(value):
  While true:
    Rv = subdoc_contains("key", "", item)
    If rv.index < 0: break # missing:
    Cas = rv.cas
    Try:
      subdoc_remove(key, "[" + rv.index + "]")
      return
    Except CasMismatch:
      continue
```


### Maps
The JSON representation of a map is a JSON object (`{}`). Items are added or removed to the object based on their keys.

#### Iteration

Similar to _List_ iteration

#### Element Access

Native subdoc support

#### Contains

Check that a given key exists within the dictionary. This can be done ussing
the `EXISTS` subdoc opcode.

#### Size

#### Keys

Get only the keys of the map. This Should be done by fetching the entire map. Future server versions may offer native support for this

#### Values

Get only the values of the map. This should be done by fetching the entire map.


### Sets

Sets are represented as a JSON list (`[]`). In essence they are derivatives of arrays with specialized add/remove operations.

Because of current implementation limitations, sets are limited to primitives only because uniqueness at the server level is done via a string comparison. Sets will then be used as an array.

Because subdoc does not have sorting functionality, sets will not be sorted.

Note: SDKs implementing a "Set" API **may** use value-types that are not primitives with the caveat that they document the fact that any sort of non-primitive comparison operation will take place on the client. This should not be permissible for "discreet"-style APIs, however.

#### Iteration

See _List_::Iteration

#### Add

Can be done using `ARRAY_ADD_UNIQUE`

##### Values

Fetch the entire set (i.e. array)

##### Contains

See _List_::Contains

##### Size

See _List_::Size


#### Queues

Queues are almost identical to lists, but provide a special `pop()` method which safely returns and removes the item.
In JSON, queues are represented as JSON arrays.


#### Iteration

See _List_::Iteration

#### Add

See _List_::Prepend

#### Contains

See _List_::Contains

#### Size

See _List_::Size

#### Pop


Retrieve `array[-1]` and remove `array[-1]` using CAS:

```
pop() -> item:
  While true:
    Rv = subdoc_get("key", "[-1]")
    Value = rv.value
    Try:
      subdoc_remove("key", "[-1]", cas=rv.cas)
      Return value
    Except CasMismatch:
      continue
```

The pop operation should not participate in the traditional `mutate_in` chain, since it is a compound operation and its execution is not well defined within the context of the execution chain.
Put simply, we can't implement pop with the same atomicity constraints normally expected.


## Nesting Levels

Datstructures supports only a single level of addressable nesting. In a sense, this makes it less powerful than subdocument, since the latter supports accessing up to 32 levels of nesting.

Of course, subdocument and datastructures can be used seamlessly in the same document.

## Errors

Errors should be manifested and wrapped in the native language's errors for various conditions when possible. For example, an ENOENT in Memcached should result in a KeyError (Python) or OutOfBoundsException (Java).
If the error is not a typical data error which cannot be translated natively, then the real Couchbase error should be returned, resulting in an exceptional circumstance. These data structures are ones of convenience, not of altering reality.


### Network/Node failure errors

If these errors occur in the face of a compound operation they may be relayed to the user as normal. Because all the compound operations do not _begin_ as destructive operations, it is always safe to break the execution chain in situ.

### Data Errors

Data errors refer to errors which take place as a result of data not existing or not being within constraints. Within data errors there are fulldoc and subdoc errors.

#### Document Not Found

Handle as normal (i.e. propagate). This would likely result from user intervention (i.e. purposefully deleting the document). This may also be thrown if the document does not exist, and on-demand creation is configurable.


#### Path Mismatch

- If the document is modified outside of datastructures (e.g. used to be a list and is now a dictionary),
- If a wrong input type was supplied

This will happen if the document is modified outside of datastructures, or if there is a bug within the datastructure implementation, or if the wrong type is supplied (e.g. non-primitive for set addition).

#### Document Too Big

If a document becomes too big the current behavior should be to propagate the exception. In the future it may be possible to add BigList or BigMap implementations which would span multiple documents, however such implementations may not be entirely "free".


## Subdocument Method Renaming/Aliasing

For SDKs which do not support the magic wrapping of native data types, data structures can still be implemented at a conceptual level by renaming sub-document API methods. The methods will follow the naming convention of ObjectVerb. This will let users intuitively know which methods are available on which data types.

The return types for all these methods should be of the normal 'Document' return type, or something that functions similarly to it. It would be nice if we had a specific return type called 'Document Fragment'. However that name is already reserved for a collection of 'document fragments'. The different name would just be for clarification, anyway.

Note that in this situation some primitive methods may need to be duplicated multiple times, one for each datatype. For example, Remove may have a ListRemove and DictRemove variant.

Mutator methods may also accept CAS and durability requirements in situations where doing so will not add to much programmatic difficulty. Because the goal of this feature is convenience, it should not sacrifice itself for flexibility.


<table>
<tr><th>Datastructure API</th><th>Subdocument API</th></tr>
<tr>
    <td><code>MapGet(String key, String mapkey)</code></td>
    <td><code>L(key).get(mapkey)</code></td>
</tr>
<tr>
    <td><code>MapRemove(String key, String mapkey)</code></td>
    <td><code>M(key).remove(mapkey)</code></td>
</tr>
<tr>
    <td><code>MapSize(String key) -> Int</code></td>
    <td><code>L(key).get_count</code>. Pre-spock can do <code>len(get(key))</code></td>
</tr>
<tr>
    <td><code>MapAdd(String key, String mapkey, Object value, Bool createDoc)</code></td>
    <td><code>M(key).dict_upsert(mapkey, value, create_doc=createMap)</code>.
    Note that pre-spock does not have support for <code>FLAG_MKDOC</code>, and therefore the implementation
    may look something like this:
    <code><pre>
while True:
    try:
        bucket.mutate_in(key, SD.upsert(mapkey, value))
    except NotFoundError:
        if not createMap:
            ## Missing, and doc creation not requested. Simply bail
            raise
        try:
            bucket.insert(key, {mapkey: value})
        except KeyExistsError:
            # Race condition. We'll just do a normal mutate_in in the next iteration
            continue

</pre></code></td>
</tr>

<tr>
    <td><code>ListGet(String key, Int index)</code></td>
    <td><code>L(key).get("", '[' + index + ']')</code></td>
</tr>
<tr>
    <td><code>ListAppend(String key, Object value, Bool createList)</code></td>
    <td><code>M(key).array_push_last("", value)</code>. See notes on <code>MapAdd</code> for how to handle
    the <code>create</code> option</td>
</tr>
<tr>
    <td><code>ListPrepend(String key, Object value, Bool createList)</code></td>
    <td><code>M(key).array_push_first("", value)</code>. See notes on <code>MapAdd</code> for how to handle
    the <code>create</code> option</td>
</tr>
<tr>
    <td><code>ListRemove(String key, Int index)</code></td>
    <td><code>M(key).remove('[' + index + ']')</code></td>
</tr>
<tr>
    <td><code>ListSet(String key, Int index, Object value)</code></td>
    <td><code>M(key).replace('[' + index + ']', value)</code></td>
</tr>
<tr>
    <td><code>ListSize(String key) -> Int</code></td>
    <td>See <code>MapSize</code></td>
</tr>

<tr>
    <td><code>SetAdd(String key, String value, boolean createSet</code></td>

    <td><code>M(key).add_unique(key, value)</code>. See <code>MapAdd</code> for
    how to handle the <code>create</code> argument.</td>
</tr>
<tr>
    <td><code>SetContains(String key, String value) -> Bool</code></td>
    <td>Currently not implemented on the server side. This will need to be
    emulated on the client like so:
    <code><pre>
list = get(key)
if value in list:
    return true
else:
    return false
</pre></code></td>
</tr>
<tr>
    <td><code>SetSize(String key) -> Int</code></td>
    <td>See <code>MapSize</code></td>
</tr>
<tr>
    <td><code>SetRemove(String key, String value)</code></td>
    <td>
    <code><pre>
while True:
    rv = bucket.get(key)
    if rv.value not in list:
        return False
    index = list.index(rv.value)
    try:
        bucket.mutate_in(SD.remove('[' + index + ']'), cas=rv.cas))
        return True
    except PathNotFoundError, KeyExistsError:
        # The document has been modified since last accessing it. Fetch it again
        continue
</pre></code></td>
</tr>

<tr>
    <td><code>QueuePush(String key, Object value, Bool createQueue)</code></td>
    <td>See <code>ListPrepend</code></td>
</tr>
<tr>
    <td><code>QueuePop(String key) -> Object</code></td>
    <td>See Queue::Pop implementation above. Note that an empty queue should raise an exception
    of the type appropriate when native language-level queues are empty</td>
</tr>
<tr>
    <td><code>QueueSize(String key) -> Int</code></td>
    <td><code>ListSize(key)</code></td>
</tr>
</table>

Note that operations whose server-side commands have not yet been implemented (NYI) are expected to implement this client-side.

Clients may also implement backwards-compatible functionality for features which are dependent on a specific server version.

## Language Specifics

### Java

The datastructure implementations will conform to the JDK's Collection API. Each collection will at least need the document's id and a reference to the Bucket object through which to access the subdoc API and in which to store the underlying document.Note the types are for now restricted to JSON simple types (String, boolean, Number, JsonArray, JsonObject...).

Below are quick examples that showcase this:

#### For lists:

```java
//we provide initial values for the list, so any pre-existing doc is overridden
List<String> names = new CouchbaseArrayList<String>("key", bucket, "foo", "bar");
if (names.contains("bar")) {
    names.add(0, "baz");
}
for (String name : names) { //the collection has an iterator
    System.out.println(name);
}
//prints baz, foo, bar

//this will rebuild the collection from existing document
List<String> namesFromDoc = new CouchbaseArrayList<String>("key", bucket);
for (String name : names) {
    System.out.println(name);
}
//prints baz, foo, bar as well
```

#### For maps

```java
Map<String,String> map = new CouchbaseMap<String>("key", bucket);
map.put("subkey", "subvalue");
```

### Python

Python does have a collections API. Two options are offered below, one uses the 'collections' API and the other uses an explicit function-based API. Unlike Java, python uses duck-typing APIs, so there is no predetermined set of operations which the SDK `must` support. Generally however, index/key addressing and iteration support is desired.

#### Collections-based API:

```python
dd = CouchbaseMap('a_map', cb)
dd['foo'] = 'bar'
    print(cb.get('a_map'))
    len(dd)

    aa = CouchbaseList('a_list', cb)
    aa.append('Hello')
    aa.append('World')
    aa.extend(['foo', 'bar', 'baz'])
    len(aa)

    ss = CouchbaseSet('a_set', cb)
    ss.add('hello')
    ss.add('hello')
    len(ss)
    ss.remove('hello')

    qq = CouchbaseQueue('a_queue', cb)
    qq.push(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    print(qq.pop())
    print(qq.pop())
```


#### Function-based API

```
cb.map_set('docid', 'email', 'm@n.com', create=True)
cb.list_append('docid', 1, 2, 3, 4, 5, 6, create=True)
cb.queue_push('docid', 'job1234')
job = cb.queue_pop('docid').value
cb.set_remove('docid', value)
```

**NOTE**: Currently, only the functional API has been implemented

### Go

```go
bucket.MapAdd("a_map", "mapkey", "stuff", false)
bucket.MapGet("a_map", "mapkey", &valOut)
bucket.ListGet("a_list", 0, &valOut)
bucket.SetAdd("a_set", "set_member", true)
bucket.SetRemove("a_list", "set_member");
```

### .NET

```csharp
 var dictionary = new CouchbaseDictionary<string, Poco>(_bucket, key);
 dictionary.Add("somekey1", new Poco { Name = "poco1" });
 dictionary.Add("somekey2", new Poco { Name = "poco2" });
 var result = dictionary.Remove("somekey2");

 var collection = new CouchbaseList<Poco>(_bucket, key);
 collection.Add(new Poco { Key = "poco1", Name = "Poco-pica" });
 var item = collection[0];
 var item = queue.Peek();
```

## Questions

### Do sets have any ordering?

No, the ordering of sets is undefined. We only make a guarantee about membership.

### Can sets contain non-string members?

The server-side subdoc implementation can only handle primitive containing arrays
for set operations (currently, only `ADD_UNIQUE`). So technically speaking, you can
also add support for sets containing integers, booleans, natural numbers, and null
values.

If you are implementing a generic API (such that you cannot make a distinction at the API level between different set value types), you *may* implement handling for non-primitive
set types, so long as you *document* that any operations on this set will need to go
through a full-document fetch.

### What if a document grows beyond 20MB

This is a situation which shouldn't happen in normal use, but in such a case the standard
`E2BIG` error should just be bubbled up.

In the future, we may add more data types which are specifically suited to grow beyond 20MB

## Other Thoughts

These ideas were floated while discussing datastructures, but were deemed out of scope for this feature. They are given a mention here:

* **Bitwise operations**
  The ability to use integers rather than explicit JSON for boolean values.
  this may save storage and bandwidth. _Requires server support_.

* **Large Queues**
  Many people require large scalable job queues. The queue currently implemented
  in datastructures/subdoc is simply an array with pop/push functionality and is not designed to
  contain more than several items, but rather on the order of tens or hundreds.

* **Large Data Structures**
  As a superset of large queues, sometimes a large map would be nice.

* **Higher level set operations**.
  Currently we only support things like adding and membership checking.
  A "proper" set API usually contains things like intersection, union,
  and so on.
