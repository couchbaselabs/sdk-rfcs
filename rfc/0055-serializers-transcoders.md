# Meta

* RFC Name: SDK3 Transcoders & Serializers
* RFC ID: 0055-serializers-transcoders
* Start Date: June 13, 2019
* Owner(s): Brett Lawson, Jeffry Morris
* Current Status: ACCEPTED

## Motivation

The SDK team has provided a common system for converting SDK data-types to storable formats for Couchbase Server which is common across all SDKs and provides the ability for the customer to seamlessly transition their data between numerous languages.

In certain cases, the transcoder or serializer behaviours may be altered by the user providing a custom implementation of `ITranscoder` and/or `IJsonSerializer`. For example:

A user has stored its data in Couchbase in a binary format and wants it to remain binary as a storage format.
A user has stored its data in Couchbase in a binary format and wants to read it as binary and write it as JSON.
A user wishes to use a serializer other than the default serializer (Jackson or NewtonSoft.NET for example) and provide a custom implementation of serializer.
etc...

The first two cases are generally legacy/upgrade scenarios from customers or users who started with very old Couchbase servers before JSON was a storage type. The last is done to satisfy the needs of a customer or user for whatever reason does not want to use the default serializer (usually for performance reasons or because they wish to reuse the custom configuration which they've already implemented with an alternative JSON serialization library).

## Summary

Most SDKs use Transcoders and Serializers when reading or writing data to and from Couchbase using the various Services (Search, KV, Views, etc). 

Transcoders handle the encoding and decoding of the documents when stored and retrieved from Couchbase Server using the [Common Flags Specification][SDK-RFC#20]. This is done so that documents can be written in one language's SDK and read universally in another and vice versa. The RFC defines the format types of JSON, non-JSON string, and raw binary. Transcoders also delegate the reading or writing of the data using serializers or native conversions from bytes to concrete types and vice versa.

Serializers are used by Transcoders to handle converting the actual JSON bytes to and from native language objects (native data structures that mirror the JSON documents elements). These native objects are frequently very specific to a platform. 

## Technical Details

The class diagram below shows the two major interfaces `ITranscoder` and `IJsonSerializer`. In addition to these interfaces, specific core libraries for converting to & from bytes and primitive types may be used within an implementation. 


![figure 1: general design](https://github.com/couchbaselabs/sdk-rfcs/blob/master/rfc/figures/rfc55-uml1.png)

The serializer is used by the various non-binary operations within Couchbase to perform parsing of JSON data.  This is used for example when parsing rows from N1QL, fields from Search or values from Views.

The transcoder types are used by various CRUD operations within Couchbase to perform transcoding of the raw bytes stored within Couchbase to and from the native types of the specific SDK language.  The transcoder is responsible for accepting a native type and generating the raw bytes for storage along with a set of flags used to specify the format of those bytes for other SDKs.  Note that transcoders which perform serialization/deserialization of JSON types should have a IJsonSerializer property used to configure the JSON serialization behaviour of that Transcoder.

### Datatype Field Handling

The SDK is responsible for parsing the flags returned by the transcoder and detecting the presence of the Common Flags JSON bit.  If this bit is set, the SDK should also specify the JSON bit within the datatype field sent to the server.  During the deserialization phase, the common flags are used exclusively, and the datatype flag is ignored.

## Default JsonSerializer

The default JsonSerializer implementation within the SDK should handle passing types of T, all numeric types and strings through the standard JSON library included with the SDK.  In the case of a byte array, the Serializer should instead pass the data through unchanged, enabling query service results to be fetched without performing serialization by passing a byte array as the output type T.

## Default Transcoders

The SDK must implement a number of specific implementations of the `ITranscoder` interface, along with a default implementation of the IJsonSerializer interface.  The JsonTranscoder implementation should be the default transcoder utilized by the SDK should no other option be specified by the user, the default serializer should take advantage of whatever JSON serialization already exists within the SDK.  The following is a list of specific transcoder implementations which MUST be implemented by the SDK, along with information about their specific behaviours.  Note that some knowledge of the specifics of [Common Flags][SDK-RFC#20] is assumed in the explanation of the transcoders below.

The provided transcoders should not perform any validation on the format bits of the common flags. They should attempt to deserialize the bytes regardless of the flags' value.

### LegacyTranscoder

The LegacyTranscoder is intended to be a backwards-compatibility transcoder.  This transcoder implements behaviour which matches the SDK 2.0 behaviours and allows customers migrating from these older SDKs who did not make use of strictly JSON data to correctly cooperate with the newer SDK.  Note that this transcoder should have a IJsonSerializer property used to customize the JSON serialization behaviour of the transcoder.

* T -> JSON Value [Flags=JSON]
* String -> String Bytes [Flags=String]
* Number -> JSON Number [Flags=JSON]
* Buffer -> Binary Bytes [Flags=Binary]

### JsonTranscoder (Default)

The JsonTranscoder is intended to be the new default for SDK 3.0, this transcoder implements JSON transcoding alone in order to guarantee compatibility with the services integrated in Couchbase Server.  In addition, the specifics of this implementation resolve a number of ambiguous scenarios with regards to handling pre-transcoded types, forcing the user to explicitly specify their meaning through (potentially) the selection of another transcoder type.  Note that this transcoder should have a IJsonSerializer property used to custom the JSON serialization behaviour of the transcoder.

* T -> JSON Value [Flags=JSON]
* String -> JSON String [Flags=JSON]
* Number -> JSON Number [Flags=JSON]
* Binary -> Error

#### RawJsonTranscoder

The RawJsonTranscoder provides the ability for the user to explicitly specify that the data they are storing or retrieving is meant to be strictly JSON bytes.  This enables the user to retrieve data from Couchbase and forward it across the network without incurring unnecessary parsing costs along with enabling the user to take JSON data which was received across the network and insert it into Couchbase without incurring parsing costs.  Note that this transcoder does not accept an IJsonSerializer, and always performs straight pass through of the data to the server.

* T -> Error
* String -> JSON Bytes [Flags=JSON]
* Number -> Error
* Buffer -> JSON Bytes [Flags=JSON]

#### RawStringTranscoder

The RawStringTranscoder provides the ability for the user to explicitly store and retrieve raw string data to Couchbase.  This transcoder is responsible for ensuring that the data is encoded as UTF-8 and masking the appropriate string flag.

* T -> Error
* String -> String Bytes [Flags=String]
* Number -> Error
* Buffer -> Error

#### RawBinaryTranscoder

The RawBinaryTranscoder provides the ability for the user to explicitly store and retrieve raw byte data to Couchbase.  The transcoder does not perform any form of real transcoding, but rather passes the data through and assigns the appropriate binary flags.

* T -> Error
* String -> Error
* Number -> Error
* Buffer -> Binary Bytes [Flags=Binary]

#### SDK Specific Transcoder

A number of languages in the wild have custom serialization formats implicitly provided by the language.  For instance, in the case of Java there is Java Serialized Object Format, in the case of PHP there is the PHP Serialized Object Format.  In the case that a custom language specific serialization format exists for the implementing language, a custom transcoder which specifically acts against this format should be implemented.  The SDK MUST ensure that this custom format encodes the flags data according to the appropriate rules specified by [Common Flags][SDK-RFC#20].

### Language Specifics
In languages such as Go, where there is a common language-level method of specifying an array of bytes that is specifically of JSON type (in Go this is the json.RawMessage type), the SDK is responsible for extending the LegacyTranscoder, JsonTranscoder and RawJsonTranscoder to support these types implicitly.  Additionally, depending on the context of any SDK Specific Transcoder, these too should support these specialized types.

####  Languages With Type Trait Capabilities
(I'll use Scala terms here, but some languages have similar and can make use of this.)
Scala gets a small tweak to permit it to take advantage of its advanced type capabilities.  When app does:

```scala
val result = collection.get("id", transcoder = JsonTranscoder())
result.contentAs[JsonObject]()
```

The contentAs method takes a Scala implicit parameter, a `JsonSerializer[T]` (technically a Type Trait).  The 'implicit' part means the compiler is asked to find a `JsonSerializer[JsonObject]`, in various scopes.  There's a number of these `JsonSerializable[T]` provided out of the box, providing support for several popular JSON libraries, plus Strings and byte arrays.  For instance, a `JsonSerializer[io.circe.Json]` would use the Circe library to turn a byte array to and from the `io.circe.Json` type.  The user is also free to add their own to support e.g. another JSON library.  

That compiler-discovered `JsonSerializer` is then provided to the Transcoder previously specified in the GetOptions block, if a) it's a `JsonTranscoder` (the only one that can accept a `JsonSerializer`) and b) the app didn't specify a serializer for the transcoder already.  This is the only real change from the RFC - instead of the app having to choose and specify a serializer, it's compiler provided.

#### Other examples:

```scala
// Save byte array, unserialized, as Binary
val imageData: Array[Byte] = // … 
coll.replace("id", imageData, transcoder = RawBinaryTranscoder())

// saves string, unserialized, with Json dataformat
coll.replace("id", """{"hello":"world"}""", transcoder = RawJsonTranscoder())

// saves string, unserialized, with String dataformat
coll.replace("id", """{"hello":"world"}""", transcoder = RawStringTranscoder())

// pick up JsonSerializer[String], which is a pass-through serializer, and default JsonTranscoder will save with Json dataformat
coll.replace("id", """{"hello":"world"}""")
```

### Custom Transcoders and Serializers

The SDKs MUST ensure that the `ITranscoder` and `IJsonSerializer` types are able to be implemented by the user outside the scope of the default transcoders which are provided by the SDK.

Note that while utilizing a custom Transcoder may affect the ability for a customer to share data among various SDKs, a correctly implemented custom transcoder will still specify flags data conforming to [Common Flags][SDK-RFC#20], and this case should be detectable by other SDKs to avoid attempting to parse potentially unparsable data and return an error instead.

## Changelog
* June 13, 2019 - Revision #1 (by Jeffry Morris)
    * Initial Draft
* Sept 26, 2019 - Revision #2 (by Brett Lawson)
    * Rewrote the RFC according to the newly specified transcoding behaviours from Sept 21, 2019 SDK3 Core meeting.  Specifically, changing the transcoder behaviour to utilize individualized transcoders to explicitly specify the desired transcoding behaviour as opposed to utilizing a DataFormat abstraction.
    * Added a number of clarifications with regards to the implementation of transcoders and their interaction with Common Flags and the Datatype field supported by the server.
Clarified the behaviour of the JsonSerializer.
* April 30, 2020
    * Moved RFC to ACCEPTED state.
* August 26, 2025 - Revision #3 (by Dimitris Christodoulou)
    * Clarify that the provided transcoders should not validate the format bits of the common flags when decoding.

| Language | Representative | Date | Revision |
|:--- |:--- |:--- |:--- |
| Node.js    | Jared Casey         | Aug 27, 2025   | #3 |
| Go         | Charles Dixon       | April 22, 2020 | #2 |
| Connectors | David Nault         | Aug 26, 2025   | #3 |
| PHP        | Sergey Avseyev      | Sept 27, 2019  | #2 |
| Python     | Jared Casey         | Aug 27, 2020   | #3 |
| Scala      | Graham Pople        | Oct 24, 2019   | #2 |
| Java       | David Nault         | Aug 26, 2025   | #3 |
| Kotlin     | David Nault         | Aug 26, 2025   | #3 |
| C          | Sergey Avseyev      | Sept 27, 2019  | #2 |
| Ruby       | Sergey Avseyev      | Sept 27, 2019  | #2 |
| .NET       | Jeffry Morris       | Sept 8, 2025   | #3 |


[SDK-RFC#20]: /rfc/0020-common-flags.md
