# Meta

- RFC Name: SDK 3 Field-Level Encryption
- RFC ID: 64
- Start Date: April 16, 2020
- Owner: David Nault <david.nault@couchbase.com>
- Current Status: Accepted
- Relates To:
    - [RFC 32 - Field Level Encryption (SDK 2)](0032-field-level-encryption.md)
    - [RFC 58 - SDK 3: Error Handling](0058-sdk3-error-handling.md)

# Summary

Field-Level Encryption (FLE) is a feature of the Couchbase SDKs that
provides a framework for client-side encryption of sensitive fields in
JSON documents. This RFC defines an encryption algorithm and the basic
functionality all Couchbase SDKs should implement.

For a summary of the differences from FLE in SDK 2, see [Appendix
A](#appendix-a-changes-from-rfc-32).

# Motivation

Developers working in certain industries have a requirement to store
sensitive data in an encrypted format. As an alternative to
full-document encryption, FLE lets a developer retain the ability to
perform queries on the unencrypted fields of a document.

Couchbase Server is compatible with third party tools that provide full
disk or file-level encryption for data at rest. Client-side encryption
has the advantage of not requiring third party software, and also
provides additional isolation in multi-tenant environments.

Encryption is difficult to implement correctly. Providing out-of-the-box
support across the Couchbase SDKs relieves developers of this
challenging and error-prone task.

Field-Level encryption may be used to meet security requirements such as
the Payment Card Industry Data Security Standard (PCI DSS). PCI DSS
section 3.5.3 says:

> Store secret and private keys used to encrypt/decrypt cardholder data
> in one (or more) of the following forms at all times:
>
> - Encrypted with a key-encrypting key that is at least as strong as
    the data encrypting key, and that is stored separately from the
    data-encrypting key
> - Within a secure cryptographic device (such as a hardware (host)
    security module (HSM) or PTS-approved point-of-interaction device)
> - As at least two full-length key components or key shares, in
    accordance with an industry accepted method

# Limitations

- The FLE feature applies only to fields of JSON objects.

- This iteration of FLE is limited to the K/V service. Querying on
  encrypted fields is out of scope. Subdocument operations are out
  of scope.

- This RFC does not address how encrypted fields should be handled by
  connectors and other "up the stack" software components.

- This RFC is silent on the issue of key management. It defines a
  [Keyring](#keyring) interface for obtaining data encryption
  keys, but it is up to a developer to implement this interface in a
  way that follows best practices like key rotation and envelope
  encryption (encrypting data encryption keys with a master key).

- This RFC does not mandate all FLE implementations be FIPS compliant.
  The encryption algorithm specified in this RFC uses only FIPS
  140-2 "Approved Security Functions", but FIPS compliance is a high
  bar; it depends on the specific implementation of the security
  functions, as well as the runtime environment.

# JSON Format

When a field is encrypted, its value is replaced with a JSON Object that
holds the encrypted value. This object must have an "alg" field whose
value is the name of the algorithm used to encrypt the field. The object
also contains whatever other information is required to decrypt the
value.

Here’s what an encrypted value looks like. Note that in this example,
the "kid" (key ID) and "ciphertext" fields are specific to the standard
"AEAD\_AES\_256\_CBC\_HMAC\_SHA512" algorithm. Custom algorithms may
define different fields.

```json
{
  "alg": "AEAD_AES_256_CBC_HMAC_SHA512",
  "kid": "my-secret-key",
  "ciphertext": "&lt;base64-encoded-ciphertext&gt;"
}
```

When decrypting a value, the FLE framework uses the algorithm name to
look up the associated [Decrypter](#decrypter). The decrypter
then reads the algorithm-specific fields.

## Field Name Mangling

When a field is encrypted, the SDK mangles the field’s name by prefixing
it with a marker. The default prefix is "encrypted$". Implementations
should allow the developer to specify a different prefix at a global
level when configuring FLE.

Developers upgrading from SDK 2 will need to configure the prefix to be
"\_\_crypt\_". The default prefix was changed in SDK 3 to avoid
conflicts with Sync Gateway, which does not permit field names to start
with an underscore. Alternatively, the legacy field names can be updated
using a N1QL query as described in this forum post:
https://forums.couchbase.com/t/replacing-field-name-prefix/28786

Here’s a document with a normal field called "foo" and an encrypted
field called "bar". Note that the name of the "bar" field has been
mangled to indicate it holds an encrypted value:

```json
{
  "foo": "I am not a secret",
  "encrypted$bar": {
    "alg": "AEAD_AES_256_CBC_HMAC_SHA512",
    "kid": "my-secret-key",
    "ciphertext": "&lt;base64-encoded-ciphertext&gt;"
  }
}
```

## Supported Value Types

Any JSON value may be encrypted, regardless of type. FLE operates on the
textual representation of the JSON value.

To encrypt a value, first convert it to its textual representation
(serialize it as JSON). Then convert the text to a byte array using
UTF-8 encoding. The resulting byte array is the plaintext to encrypt.

| JSON Value             | Encrypt these bytes (hex)
|------------------------|-----------------
| "xyzzy"                | `22 78 79 7a 7a 79 22`
| {"dance":10,"looks":3} | `7b 22 64 61 6e 63 65 22 3a 31 30 2c 22 6c 6f 6f 6b 73 22 3a 33 7d`
| [1,1,2,3,5]            | `5b 31 2c 31 2c 32 2c 33 2c 35 5d`
| 10                     | `31 30`
| null                   | `6e 75 6c 6c`

## Test Case

Here are some test values for verifying the correctness of an FLE
implementation using the
[AEAD\_AES\_256\_CBC\_HMAC\_SHA512](#appendix-b-aead_aes_256_cbc_hmac_sha512)
algorithm which all SDKs must support.

### Key

This static key is provided only for testing. Outside of testing, the
key is obtained from a keyring.

```
00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f
10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f
20 21 22 23 24 25 26 27 28 29 2a 2b 2c 2d 2e 2f
30 31 32 33 34 35 36 37 38 39 3a 3b 3c 3d 3e 3f
```

### Initialization Vector (IV)

This static IV is provided only for testing. Outside of testing, the IV
MUST be randomly generated using a cryptographically secure algorithm.

```
1a f3 8c 2d c2 b9 6f fd d8 66 94 09 23 41 bc 04
```

### Document with Plaintext Field

```json
{"maxim":"The enemy knows the system."}
```

### Plaintext Field Value (encrypt these bytes)

```
22 54 68 65 20 65 6e 65 6d 79 20 6b 6e 6f 77 73
20 74 68 65 20 73 79 73 74 65 6d 2e 22
```

### Associated Data

In this iteration of FLE, there is no associated data. In other words,
the associated data is a byte array of length zero.

### Document with Encrypted Field

```json
{
  "encrypted$maxim": {
    "alg": "AEAD_AES_256_CBC_HMAC_SHA512",
    "kid": "test-key",
    "ciphertext": "GvOMLcK5b/3YZpQJI0G8BLm98oj20ZLdqKDV3MfTuGlWL4R5p5Deykuv2XLW4LcDvnOkmhuUSRbQ8QVEmbjq43XHdOm3ColJ6LzoaAtJihk="
  }
}
```

# Reference Design

This section offers recommendations for implementing FLE. Implementers
should adhere to the reference design where it makes sense, and diverge
where the idioms of a specific language or SDK provide a more natural
solution.

## Separate Library

The FLE feature should be packaged as a pluggable extension of the SDK.
Interfaces to support FLE may be present in the SDK code, but
implementing classes should live in a separate library. This allows the
FLE code to be distributed under a separate license.

## Manual Encryption and Decryption

An FLE implementation must allow a developer to manually encrypt and
decrypt individual fields of a JSON object.

In the Java SDK, manual encryption is done using a class called
JsonObjectCrypto. An instance of this class is obtained by calling
JsonObject.crypto() and passing in the cluster environment, a
collection, or a CryptoManager. A JsonObjectCrypto instance is bound to
the JsonObject it was created from. Fields accessed via the
JsonObjectCrypto instance are decrypted on read and encrypted on write.
Attempting to read an unencrypted field via the crypto view has the same
result as if the field does not exist.

Here’s an example of how a developer might use JsonObjectCrypto to
perform manual encryption and decryption:

```java
Collection col = cluster.bucket("b").defaultCollection();

// Write encrypted value
JsonObject document = JsonObject.create();
JsonObjectCrypto crypto = document.crypto(collection);
crypto.put("location", "Between palm trees");
collection.upsert("treasureMap", document);

// Read encrypted value
JsonObject readItBack = collection.get("treasureMap")
    .contentAsObject();
JsonObjectCrypto readItBackCrypto = readItBack.crypto(collection);
System.out.println(readItBackCrypto.getString("location"));
```

## Encrypted annotation

If an SDK language supports annotations and JSON data binding, an
implementation may support annotating specific fields of a class with an
annotation called "Encrypted".

This annotation has an optional "encrypter" attribute whose value is the
alias of an [Encrypter](#encrypter) to use when serializing the
field value. If this attribute is absent, the default encrypter is used.

Annotated fields are encrypted during serialization and decrypted during
deserialization. If any field cannot be encrypted or decrypted, the
entire data binding operation must fail in order to prevent clobbering
encrypted values.

## Framework Components

This section describes the components of the FLE framework as
implemented in the Java SDK. Implementations in other languages should
use native idioms where appropriate.

### Key

Immutable value object. An encryption key.

Properties:

- String **id** - The name of the key.

- byte\[\] **bytes** - The raw bytes of the key.

### Keyring

Interface. Provides access to encryption keys.

Methods:

- Key **get**(String keyId) - Returns the requested key. If the
  keyring supports key rotation and the key ID does not include a
  version, returns the latest version of the key.

SDKs should provide an Keyring implementation that gets keys from the
platform’s native key store, if applicable. For example, the Java SDK
has a KeyStoreKeyring that gets keys from a Java KeyStore supplied by
the developer.

### EncryptionResult

Mutable value object. The encrypted form of a message. It’s a wrapper
around a Map&lt;String, Object&gt; that corresponds directly to the JSON
format of an encrypted value.

The map must have an "alg" entry whose value is the name of the
encryption algorithm. It typically also has a "kid" entry identifying an
encryption key, and a "ciphertext" entry holding the Base64-encoded form
of the ciphertext (although these attributes are not required by the
framework).

In Java, EncryptionResult has methods for getting and putting values of
different types. For example, it has a "put" method that takes an
attribute name and a byte array; this method handles converting the byte
array to a Base64-encoded string, and a \`getBytes\` method that takes
an attribute name and returns the Base64-decoded version of the string
value. These methods are called by Encrypter and Decrypter
implementations when building or reading the encrypted form of the
message.

### Decrypter

Interface. An implementation is immutable. Knows how to decrypt using a
specific algorithm.

Methods:

- String **algorithm**() - Returns the name of the algorithm handled
  by this decrypter.

- byte\[\] **decrypt**(EncryptionResult encrypted) - Decrypts the
  given EncryptionResult. If the algorithm uses a key, the ID of the
  key is obtained from the "kid" property of the given
  EncryptionResult. Note that It is an error to pass a result whose
  algorithm does not match the decrypter.

The algorithm() method is called by the
[DefaultCryptoManager](#defaultcryptomanager) when the decrypter
instance is registered. Only one decrypter may be registered for a given
algorithm.

A decrypter gets keys from a [Keyring](#keyring). The Keyring is
passed as an argument to the decrypter’s constructor.

### Encrypter

Interface. An implementation is immutable. Knows how to encrypt using a
specific algorithm and encryption key.

Methods:

- EncryptionResult **encrypt**(byte\[\] plaintext) - The "alg" of the
  returned EncryptionResult must match the algorithm of a registered
  decrypter. The result also contains all the information required
  for decryption (this is typically a key ID and ciphertext, but can
  be any attributes required by the algorithm).

An encrypter gets keys from a [Keyring](#keyring). This keyring
is passed as an argument to the encrypter’s constructor.

The name of the encryption key is passed as a constructor argument. The
encrypter must ask the keyring for the key every time a message is
encrypted. The encrypter must inspect the ID of the returned key and use
that value when populating the EncryptionResult. This is necessary
because a keyring that supports [key rotation](#key-rotation) may
return a key whose ID differs from the requested ID.

An encrypter is registered with the
[DefaultCryptoManager](#defaultcryptomanager) under an alias
chosen by the developer. Multiple encrypter instances for the same
algorithm may be registered under different aliases (typically with
different algorithm parameters, like different encryption keys). When
the developer encrypts a field, they refer to the encrypter by this
alias.

One encrypter may be registered as the "default" encrypter. The
encrypter alias "\_\_DEFAULT\_\_" is reserved for this purpose. The
default encrypter is used in contexts where the developer does not
specify an encrypter alias.

### CryptoManager

Interface. This is the integration point between the core SDK and the
encryption library. In Java this interface lives in the core SDK and is
implemented by a class in the external encryption library.

Methods:

- Map&lt;String, Object&gt; **encrypt**(byte\[\] plaintext, String
  encrypterAlias) - Encrypts the given plaintext using the named
  encrypter. Returns the EncryptionResult as a Map. Raises
  [EncryptionFailure](#EncryptionFailure) if encryption fails
  for any reason.

- byte\[\] **decrypt**(Map&lt;String, Object&gt; encryptedNode) -
  Selects an appropriate decrypter based on the "alg" key of the
  given map. Converts the map to an EncryptionResult and asks the
  decrypter to decrypt it. Raises
  [DecryptionFailure](#DecryptionFailure) if decryption fails
  for any reason.

- String **mangle**(String fieldName) - Transforms the given field
  JSON field name to indicate its value has been encrypted (prefixes
  it with "encrypted$" or whatever prefix the developer has
  configured).

- String **demangle**(String fieldName) - Reverses the transformation
  performed by the mangle method (strips the prefix).

- boolean **isMangled**(String fieldName) - Returns true if the field
  name has been mangled (starts with the prefix).

This is the broadest abstraction in the FLE framework. A developer who
wants to radically alter how the framework behaves can implement their
own CryptoManager and handle encryption however they want.

In Java, the CryptoManager is injected into the ClusterEnvironment as
part of the global configuration. This makes it accessible to the parts
of the SDK that drive FLE.

### Provider

A provider is a factory for Encrypters and their associated Decrypter.
It does not implement any interface, and is not strictly a part of the
FLE framework. Its role is to simplify the configuration of related
components. We’ll see an example in the [Sample
Configuration](#sample-configuration) section.

### DefaultCryptoManager

Immutable registry of encrypters and decrypters. This is the standard
implementation of the CryptoManager interface.

Decrypters are indexed by algorithm. There can be at most one decrypter
per algorithm. Encrypters are indexed by alias. There can be several
encrypters for an algorithm; this is how different encryption keys are
specified. One encrypter may be registered as the default encrypter.

### Legacy Decrypters

The encryption algorithms defined in Couchbase RFC 32 are deprecated.
FLE implementations should provide decrypters (but not encrypters) for
the legacy algorithms.

The JSON format of the legacy symmetric algorithms includes the ID of
the encryption key, but not the authentication key. Likewise the JSON
form of the legacy RSA algorithm includes the ID of the public key, but
not the private key. This omission is hostile to key rotation, and is
the primary reason these algorithms are deprecated.

In order to decrypt these fields, a mapping from encryption key name to
signing key name is required. (For RSA this mapping is from public key
name to private key name.) The key name mapping is provided by the
developer when registering the legacy decrypters.

## Sample Configuration

A developer who wants to use Field-Level Encryption supplies a
CryptoManager instance when configuring the SDK. The easiest way to do
this is to build a DefaultCryptoManager. Here’s what that might look
like in Java:

```java
// Create a keyring backed by a Java KeyStore
KeyStore javaKeyStore = loadJavaKeyStore();
Keyring keyring = new KeyStoreKeyring(javaKeyStore,
    keyName -> "password");

// Create a provider for the standard encryption algorithm
var standardProvider = AeadAes256CbcHmacSha512Provider.builder()
    .keyring(keyring)
    .build();

// Create a provider for a user-defined algorithm
var customProvider = new MyCustomProvider(keyring);

// Create a new CryptoManager and register encrypters and decrypters
CryptoManager cryptoManager = DefaultCryptoManager.builder()
    .legacyAesDecrypters(keyring, s -> s + "_signing")
    .decrypter(standardProvider.decrypter())
    .decrypter(customProvider.decrypter())
    .encrypter("foo", standardProvider.encrypterForKey("key-a"))
    .encrypter("bar", customProvider.encrypterForKey("key-b"))
    .defaultEncrypter(standardProvider.encrypterForKey("key-c"))
    .build();
```

## Error Definitions

Field-Level Encryption errors are assigned IDs in the range 700-799. The
following errors have been added to SDK-RFC 58 (Error Handling) as an
addendum:

### \# 700 CryptoException

- Generic cryptography failure.

- Inherits from CouchbaseException (\# 0)

- Parent Type for all other Field-Level Encryption errors

<a name="EncryptionFailure"></a>
### \# 701 EncryptionFailure

- Raised by CryptoManager.encrypt() when encryption fails for any
  reason.

- Should have one of the other Field-Level Encryption errors as a
  cause.

<a name="DecryptionFailure"></a>
### \# 702 DecryptionFailure

- Raised by CryptoManager.decrypt() when decryption fails for any
  reason.

- Should have one of the other Field-Level Encryption errors as a
  cause.

<a name="CryptoKeyNotFound"></a>
### \# 703 CryptoKeyNotFound

- Raised when a crypto operation fails because a required key is
  missing.

<a name="InvalidCryptoKey"></a>
### \# 704 InvalidCryptoKey

- Raised by an encrypter or decrypter when the key does not meet
  expectations (for example, if the key is the wrong size).

<a name="DecrypterNotFound"></a>
### \# 705 DecrypterNotFound

- Raised when a message cannot be decrypted because there is no
  decrypter registered for the algorithm.

<a name="EncrypterNotFound"></a>
### \# 706 EncrypterNotFound

- Raised when a message cannot be encrypted because there is no
  encrypter registered under the requested alias.

<a name="InvalidCiphertext"></a>
### \# 707 InvalidCiphertext

- Raised when decryption fails due to malformed input, integrity check
  failure, etc.

## Key Rotation

Key rotation involves periodically generating new versions of data
encryption keys. The most recent version of a key is used when
encrypting. The ciphertext is annotated to indicate which key should be
used to decrypt the message. Old keys are retained so old messages can
be decrypted.

This limits the number of messages encrypted with any one key, and
mitigates the damage if a key is compromised. A more subtle (but
arguably more important) goal is to prevent key exhaustion. If an
attacker has access to a great many ciphertexts encrypted with the same
key, they may be able to learn things about the plaintexts or the key.

Key rotation does not require re-encrypting old messages with the new
key, although that is something you might want to do if a key has been
compromised. Re-encryption is out of scope for this RFC, but nothing in
this design prevents a developer from doing it themselves.

### Simple Key Rotation Example

SDKs are not required to provide out-of-the-box support for key
rotation; the only requirement in this area is that SDKs be implemented
in such a way that that key rotation can be added by a developer. The
[Keyring](#keyring) interface in the reference design is an
extension point where developers can implement the key rotation scheme
of their choice.

As a simple example, consider a key naming convention where a key’s name
consists of a base name followed by a delimiter and then a version
identifier. If we use "--" as a delimiter and ISO 8601 dates for the
version, a full key name might look like "myKey--2020-04-29".

An [Encrypter](#encrypter) is then configured to refer to the key
by its base name, "myKey". When the encrypter encrypts a message, it
asks the keyring for "myKey". The keyring notices this name does not
include a delimiter, so it returns the latest version of the key, which
happens to be called "myKey--2020-04-29". This fully-qualified name is
part of the [Key](#key-1) object returned to the encrypter.

When the encrypter writes the "kid" property into the encryption result,
it uses the full key name: "myKey--2020-04-29". When it’s time to
decrypt the message, the [Decrypter](#decrypter) gets the key
name from the "kid" property and asks the keyring for
"myKey--2020-04-29". Since this is a fully-qualified name, the keyring
returns the requested version (even though it might not be the most
recent).

In this way the encrypter always uses the latest version of the key, and
the decrypter can decrypt any message so long as the key exists in the
keyring.

# Appendix A: Changes from RFC 32

Field-Level Encryption (FLE) was originally specified in Couchbase RFC
32 and implemented in SDK 2. At the time, [key
rotation](#key-rotation) was out of scope. With SDK 3 we have an
opportunity to reshape the API to facilitate key rotation.

Another goal is to make the API less opinionated about how to do
encryption and how to manage keys, so developers can more easily
customize the behavior of the FLE components.

At the same time, it’s important to retain the ability to decrypt
messages encrypted by SDK 2.

## Universal Decryption

An important part of key rotation is being able to decrypt any message
so long as the key is available. In SDK 2 the developer had to specify
the crypto provider when decrypting. In SDK 3 the algorithm and key are
determined solely by the "alg" and "kid" fields of the JSON.

To support universal decryption, the developer now registers a single
[Decrypter](#decrypter) for each algorithm. For each encryption
key the developer wants to use, they create an
[Encrypter](#encrypter) for that key and register it under an
alias. The developer later refers to this encrypter by its alias when
encrypting fields.

## Default Encrypter

When registering encryptors, a developer may designate one as the
default. Anywhere an encryptor alias is allowed (for example, the
"encrypter" attribute of the Encrypted annotation), the developer may
omit it to use this default encrypter.

## Deprecated Algorithms

The encryption algorithms defined in RFC 32 are deprecated. SDKs must
retain the ability to decrypt messages using these algorithms, but
should not use them to encrypt new messages.

### RSA Deprecation

The RSA-2048 algorithm defined in RFC 32 is limited to encrypting
plaintext messages of [no more than 245
bytes](https://security.stackexchange.com/questions/33434/rsa-maximum-bytes-to-encrypt-comparison-to-aes-in-terms-of-security).
It is quite easy to exceed this limit, leading to errors at runtime.

Nobody has complained about the RSA implementation in SDK 2; this may
indicate it is not widely used. It is likely that asymmetric encryption,
while interesting, is the solution to a problem our customers do not
have.

The RSA-2048 algorithm is deprecated with no replacement.

### AES Deprecation

The AES algorithms defined in RFC 32 require separate keys for
encryption and signing, but the name of the signing key is not present
in the JSON. This is an obstacle to key rotation, since it's impossible
to authenticate old messages without knowing which signing key to use.

Additionally, the implementations of these algorithms were inconsistent
across SDKs. Due to confusion around signature calculation, the Java and
.NET SDKs are unable to read each others’ encrypted messages.

These algorithms are deprecated in favor of
AEAD\_AES\_256\_CBC\_HMAC\_SHA\_512.

## New Algorithm

SDK 3 uses a new algorithm,
[AEAD\_AES\_256\_CBC\_HMAC\_SHA\_512](#appendix-b-aead_aes_256_cbc_hmac_sha512),
which has two advantages over the deprecated AES algorithms:

- It has an unambiguous specification in the form of an Internet
  Draft: [Authenticated Encryption with AES-CBC and
  HMAC-SHA](https://tools.ietf.org/html/draft-mcgrew-aead-aes-cbc-hmac-sha2-05).

- It bundles the auth key and the encryption key into a single 64-byte
  "fat key." The key materials used by the AES and HMAC algorithms
  are distinct, but the keys can be managed as a single unit.

## Miscellaneous Changes

In SDK 2, encrypted fields could only appear at the root level of a
document. In SDK 3 they may be present at any depth and may be nested
within one another.

In SDK 2, the key name was included in signature calculations. In SDK 3
it is not; if the user wants to rename their keys, that’s totally fine.

The EncryptedField annotation is renamed to "Encrypted", and its
"provider" attribute is renamed to "encrypter". The attribute is
optional; if not specified, the default encrypter is used.

# Appendix B: AEAD\_AES\_256\_CBC\_HMAC\_SHA512

All Couchbase SDKs must support the "AEAD\_AES\_256\_CBC\_HMAC\_SHA512"
algorithm which is formally described by an Internet Draft:
[Authenticated Encryption with AES-CBC and
HMAC-SHA](https://tools.ietf.org/html/draft-mcgrew-aead-aes-cbc-hmac-sha2-05).

It’s worth reading at least the
"[Rationale](https://tools.ietf.org/html/draft-mcgrew-aead-aes-cbc-hmac-sha2-05#section-4)"
and "[Security
Considerations](https://tools.ietf.org/html/draft-mcgrew-aead-aes-cbc-hmac-sha2-05#section-6)"
sections.

An informal description of the algorithm is provided below. Note that in
this iteration of FLE, the "additional data" will always have zero
length.

## Encryption

Encryption takes as input a 64-byte key, the plaintext to encrypt, and
some associated data to authenticate but not encrypt.

- Split the 64-byte key in half. The first 32 bytes are the HMAC key,
  and the second 32 bytes are the AES key.

- Generate a random 16-byte initialization vector (IV) using a
  cryptographically secure algorithm.

- Using this IV, encrypt the plaintext with AES in CBC mode with PKCS7
  padding. Prepend the IV to the result to get the "AES ciphertext."

- Calculate an HMAC SHA-512 digest of the following bytes, in order:

    - The associated data. This is anything you want to have
      authenticated but not encrypted. It's optional and may have
      zero length. Note that the associated data cannot be
      reconstructed from the output of this method.

    - The AES ciphertext.

    - The length in bits of the associated data, represented as an
      unsigned 64-bit big-endian integer. For example, if there are
      42 bytes of associated data, that's 336 bits, or 0x150 in hex.
      The value to pass to the message digest are these 8 bytes:
      `0x00 0x00 0x00 0x00 0x00 0x00 0x01 0x50`.

- The digest generated in the previous step should be 64 bytes long.
  Truncate it to 32 bytes to get the "auth tag" (or signature, if
  you prefer).

- Append the auth tag to the AES ciphertext to get the "authenticated
  ciphertext". Return the authenticated ciphertext.

## Decryption

Decryption takes as input a 64-byte key, the ciphertext to decrypt, and
some associated data to authenticate.

- Split the key as for encryption.

- Split the authenticated ciphertext into the AES ciphertext and the
  auth tag (the auth tag is the last 32 bytes of the authenticated
  ciphertext).

- Calculate the HMAC SHA-512 digest as for encryption, then truncate
  it to 32 bytes. Compare the result to the auth tag using a
  time-constant comparison (compare every byte; **don't
  short-circuit at the first mismatch**).

- If there's a mismatch,throw an exception or return a special value
  to signal a failure. Otherwise, proceed to the next step.

- Decrypt the AES ciphertext (remember, the first 16 bytes are the
  IV). Return the plaintext.

## Test Case

Here are some test values from the Internet Draft for verifying the
correctness of your implementation.

### Key

```
00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f
10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f
20 21 22 23 24 25 26 27 28 29 2a 2b 2c 2d 2e 2f
30 31 32 33 34 35 36 37 38 39 3a 3b 3c 3d 3e 3f
```

### Initialization Vector (IV)

This static IV is provided only for testing. Outside of testing, the IV
MUST be randomly generated using a cryptographically secure algorithm.

```
1a f3 8c 2d c2 b9 6f fd d8 66 94 09 23 41 bc 04
```

### Plaintext

```
41 20 63 69 70 68 65 72 20 73 79 73 74 65 6d 20
6d 75 73 74 20 6e 6f 74 20 62 65 20 72 65 71 75
69 72 65 64 20 74 6f 20 62 65 20 73 65 63 72 65
74 2c 20 61 6e 64 20 69 74 20 6d 75 73 74 20 62
65 20 61 62 6c 65 20 74 6f 20 66 61 6c 6c 20 69
6e 74 6f 20 74 68 65 20 68 61 6e 64 73 20 6f 66
20 74 68 65 20 65 6e 65 6d 79 20 77 69 74 68 6f
75 74 20 69 6e 63 6f 6e 76 65 6e 69 65 6e 63 65
```

### Associated Data

```
54 68 65 20 73 65 63 6f 6e 64 20 70 72 69 6e 63
69 70 6c 65 20 6f 66 20 41 75 67 75 73 74 65 20
4b 65 72 63 6b 68 6f 66 66 73
```

### Ciphertext

```
1a f3 8c 2d c2 b9 6f fd d8 66 94 09 23 41 bc 04
4a ff aa ad b7 8c 31 c5 da 4b 1b 59 0d 10 ff bd
3d d8 d5 d3 02 42 35 26 91 2d a0 37 ec bc c7 bd
82 2c 30 1d d6 7c 37 3b cc b5 84 ad 3e 92 79 c2
e6 d1 2a 13 74 b7 7f 07 75 53 df 82 94 10 44 6b
36 eb d9 70 66 29 6a e6 42 7e a7 5c 2e 08 46 a1
1a 09 cc f5 37 0d c8 0b fe cb ad 28 c7 3f 09 b3
a3 b7 5e 66 2a 25 94 41 0a e4 96 b2 e2 e6 60 9e
31 e6 e0 2c c8 37 f0 53 d2 1f 37 ff 4f 51 95 0b
be 26 38 d0 9d d7 a4 93 09 30 80 6d 07 03 b1 f6
4d d3 b4 c0 88 a7 f4 5c 21 68 39 64 5b 20 12 bf
2e 62 69 a8 c5 6a 81 6d bc 1b 26 77 61 95 5b c5
```

# Appendix C: HASHICORP\_VAULT\_TRANSIT

This section describes an optional integration with the HashiCorp Vault
Transit secrets engine.

Transit provides "encryption as a service". According to the HashiCorp
web site, "The primary use case for transit is to encrypt data from
applications while still storing that encrypted data in some primary
data store. This relieves the burden of proper encryption/decryption
from application developers and pushes the burden onto the operators of
Vault."

If an SDK provides a Transit integration, it must use the algorithm name
"HASHICORP\_VAULT\_TRANSIT". It must store the name of the encryption
key in the "kid" attribute, and store the encrypted data in the
"ciphertext" attribute.

Here’s an example of a field encrypted with Transit:

```json
{
  "alg": "HASHICORP_VAULT_TRANSIT",
  "kid": "myKey",
  "ciphertext": "vault:v1:8SDd3WHDOjf7mq6..."
}
```

The HASHICORP\_VAULT\_TRANSIT algorithm does not require a Keyring,
since the encryption keys are managed by Vault and never leave the Vault
server.

# **Changelog**

- Apr 29, 2020 - Revision \#1 (by David Nault)

    - Initial Draft

- May 12, 2020 - Revision \#2 (by David Nault)

    - Updated motivation to indicate Couchbase Server’s support for
      encryption at rest is provided by third party software.

    - CryptoException must be a subclass of CouchbaseException.

    - Added description of HASHICORP\_VAULT\_TRANSIT algorithm.

- May 20, 2020 - Revision \#3 (by David Nault)

    - Changed default field name prefix from "\_\_crypt\_" to
      "encrypted$" to avoid conflict with Sync Gateway which does
      not allow field names to start with an underscore.

    - Updated JsonObjectCrypto example.

    - Simplified Keyring interface to have a single "get" method.
      Implementers are free to elaborate as necessary.

- May 27, 2020 - Revision \#4 (by David Nault)

    - The "EncryptedField" annotation is renamed to "Encrypted". (More
      concise, and one day we might want to let users apply it to a
      whole object.)

    - The "InvalidKeySize" error is renamed to "InvalidCryptoKey".

    - Added "EncryptionFailure" and "DecryptionFailure" errors, raised
      by CryptoManager.encrypt() and decrypt() respectively.

    - Formalized the error definitions and added them to SDK RFC 58.

- December 15, 2020 - Revision \#5 (David Nault)

    - Fix typo: a JsonCryptoObject is obtained by calling
      JsonObject.**crypto**() -- not JsonObject.**create**().

    - Added link to forum post describing how to upgrade legacy field
      names from "\_\_crypt\_" to "encrypted$"

    - Clarified that EncryptionResult.put/get methods are intended to
      be called by Encrypter and Decrypter implementations.

- January 27, 2021 - Revision \#6 (David Nault)

    - Added info about Payment Card Industry Data Security Standard
      (PCI DSS) to the Motivation section.

# **Signoff**

| Language     | Team Member     | Signoff Date     | Revision     |
|--------------|-----------------|------------------|--------------|
| Node.js      | Brett Lawson    | 2021-03-03       | 6            |
| Go           | Charles Dixon   | 2021-02-15       | 6            |
| Connectors   | David Nault     | 2020-02-11       | 6            |
| PHP          | Sergey Avseyev  | 2020-03-03       | 6            |
| Python       | Jared Casey     | 2021-02-24       | 6            |
| Scala        | Graham Pople    | 2021-03-12       | 6            |
| .NET         | Jeffry Morris   | 2021-03-03       | 6            |
| Java         | David Nault     | 2020-02-11       | 6            |
| C            | Sergey Avseyev  | 2020-03-03       | 6            |
| Ruby         | Sergey Avseyev  | 2020-03-03       | 6            |
