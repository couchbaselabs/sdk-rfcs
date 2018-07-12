# Meta

* RFC Name: Field Encryption
* RFC ID: 0032
* Start date: 12/6/2017
* Owner: Jeff Morris
* Current Status: REVIEW

## Summary

As Couchbase Server continues to add more and more security features, one feature that customers continuously ask for is document and/or field encryption within the SDK. This RFC defines the motivation, general design and language specifics with respect to field encryption providing a blueprint for the implementation across all Couchbase SDK’s and documents any idiomatic differences between the SDK implementations. 

## Motivation

Customers needing to support [FIPS-140-2](https://en.wikipedia.org/wiki/FIPS_140-2) compliance must at a minimum encrypt data using a FIPS-140-2 compliant cryptography algorithm. Currently, the go to solution has been to use Vormetric. The goal is to provide a built in, field level solution in addition to the parter solution with Vormetric for Couchbase customers.  This would still allow FIPS-140-2 compliance to be met. Additionally, the SDK feature should be extensible so that additional features can be added in the future as customer demands dictate.

### Functional Requirements

*  The encryption and decryption must happen entirely on the client; the server is completely passive.
*  The supported algorithms must be FIPS-140-2 compliant; AES-256 and RSA-2048 are supported.
*  The feature should be packaged as a pluggable extension of the SDK.
*  The SDK must check the size of the document after encryption and fail if it exceeds to the maximum Couchbase document size of 20MB; this must be done on the client and not rely on the server to reject the request with 0x0003 (Value too large).
*  Documents with encrypted fields must be discoverable for auditing purposes. The simplest example of auditing is using a N1QL query to select documents with encryption.
*  K/V only is in-scope for field encryption in this iteration.
*  The KeyStore must be abstract and pluggable; Java should support the Java Key Store and .NET should support the Windows Certificate Store at a minimum. Future implementations should include Vault and perhaps KMIP.

### What are some approaches to Field Encryption?

There are a couple of different ways of doing this, for example (but not exclusively):

*  Declaratively using annotations or attributes in meta-languages such as Java and .NET based languages on the object model representing the JSON document
*  Streaming and using expressions to select specific fields to encrypt/decrypt
*  As an up-the-stack feature for example in Spring or Ottoman (see Mongoose field encryption and Mongoid for reference)
*  As an API built into or extending the Document object introduced and supported by all Couchbase SDKs that are SDK 2.0 API compliant
*  As an extension of the Sub-Document client API where a Sub-document operation can contain a parameter or action indicating that the field being operated on should be encrypted or decrypted before being sent to the database or after it has been received by the SDK
*  As part of event API within the SDK where users can inject behaviour at key points of the couchbase operations request/response cycle: beforeSave, afterSave, beforeSend, afterSend, etc.
*  There is also JOSE which is an IETF spec https://tools.ietf.org/html/rfc7520 . There is a Javascript implementation here: https://github.com/anvilresearch/jose#usage. This may be useful in terms of being a mature proposal, albeit attached to schema-based documents. Perhaps any non-schema based approach could at least align with its general modus operandi/configuration parameters, etc. 

## General Design

There are three major components which make up the entirety of the solution:
  1. **Key Management** - how keys are created, where they are stored, how they are protected and how they are distributed.
  2. **Data Encryption** - which algorithms to support and what/how to encrypt/decrypt
  3. **API Design** - the actual means of encrypting/decrypting data via an API or sub-process within the SDK. Note that the initial implementation is based solely on the K/V API - N1QL, FTS and Analytics are OUT OF SCOPE

![figure 1: general design][figures/rfc32-fig1.png]

Above is a high-level architectural model explaining the various actors and their responsibilities. It’s important to note that the current spec is for field encryption is to be client-side only, however, a future enhancement may include server-side collaboration.

It’s important to note that Key Management and initialization of Data Encryption are done during configuration of the SDK and the Data Encryption API is done at runtime while the application is doing its normal read/write operations.

### Key Management

Nearly every platform has its own standard key and/or certificate store. For instance, Java (and appears GO, Node, Python, etc as well) uses the [Java Keystore](https://en.wikipedia.org/wiki/Keystore) and Windows uses the Windows Registry via The [Certificate Store](https://superuser.com/questions/217719/what-are-the-windows-system-certificate-stores). Other solutions include 3rd party API’s like [Hashicorp Vault](https://www.vaultproject.io/?_ga=2.157641401.2008868641.1512607701-1757330291.1509988984) which are platform agnostic relying on HTTP based API’s.

We will support a pluggable “provider” model for the keystore so that via configuration the SDK user can chose the key management infrastructure or solution of their choosing. The default implementation should be the platform idiomatic keystore for a given SDK. 

The keystore component should be pluggable and extensible to support at the least, the following providers:

  * An in-memory keystore for development and testing
  * A native key store for the SDK’s platform (Java Keystore, etc)
  * A Hashicorp Vault keystore

Here is an example of the simplest interface for the key management API:

```C#
    public interface IKeyProvider
    {
        byte[] GetKey(string keyname);
    }

```

| Method      | Optional           | Description  |
| ------------- |:-------------:| :-----|
| GetKey     | false | Returns an encryption key from a keystore given a unique key-name or path as a Base64 encoded byte array. |

The implementation of the keystore should use any necessary libraries or 3rd party API’s to enable the retrieval and storage of signing keys. The simplest implementation should be an in-memory solution for development and testing where the keys are simply stored as fields and be retrievable by key-name or path. The path would likely be the location of the key(s) on disk or perhaps URI for some remote host if a 3rd party key-management solution is implemented, like Hashicorp Vault. 

Note that the IKeyProvider is specifically terse so that it's easier to support a large number of disparate implementations.  It’s implied that concrete implementations will contain specialized constructors, properties and/or fields so that a wide-range of key-store options (JKS, DATAPI, Hashicorp Vault, etc) can be supported.

Here is an example of an in-memory store for development and testing called the “InsecureKeyStore”, because it is, well, not very secure...:

```C#
    public class InsecureKeyProvider : IKeyProvider
    {
        private readonly Dictionary<string, string> _keys = new Dictionary<string, string>();

        public string GetKey(string keyname)
        {
            return _keys[keyname];
        }

        public void StoreKey(string keyname, string key)
        {
            _keys[keyname] = key;
        }

        public string ProviderName { get; set; }

        public string PublicKeyName { get; set; }

        public string PrivateKeyName { get; set; }

        public string SigningKeyName { get; set; }

        public bool RequiresAuthentication { get; }
    }
```

The ICryptoProvider should have fields equivalent to:

| Field     | Description          | Type  |
| ------------- |:-------------:| :-----|
|ProviderName|The name or alias of the configured ICryptoProvider instance - for example 'MyProvider'.|string|
|PublicKeyName|The name of the encryption key.|string|
|PrivateKeyName|The name of the private if required for an asymmetric algorithm|string|
|SigningKeyName|The name of the password or key used for signing if required|string|

Simple and straightforward, the keys are stored by name in a dictionary and retrieved using the same name. You can imagine a similar key-store which uses a file on disk to store keys as well.

### Data Encryption

The encryption algorithms to support should be the standard asymmetric and symmetric supported by most programming languages and platforms and MUST be a [FIPS 140-2](http://csrc.nist.gov/publications/fips/fips140-2/fips1402annexa.pdf) compliant algorithm. Examples include:


Symmetric key:
  * AES-256-CBC-HMAC-SHA256
  * AES-256 (Alias for AES-256-CBC-HMAC-SHA256)

Asymmetric key:
  * RSA-2048-OEP
  * RSA-2048 (Alias for RSA-2048-OEP)

Since new algorithms are introduced and because old ones are deprecated, the data encryption algorithm implementation should be _pluggable_ and _extensible_. *A possible interface might look like this:

```C#
    interface ICryptoProvider
    {
        IKeyProvider KeyProvider { get; set; }

        byte[] Decrypt(byte[] value);

        byte[] Encrypt(byte[] value);
    }
```

And then algorithm implementations would include support for asymmetric and symmetric supported algorithms. For example:

```C#
    public class RsaCryptoProvider : ICryptoProvider
    {
        // ...
    }

    public class AesCryptoProvider : ICryptoProvider
    {
        // ...
    }
```

Since this component is largely internal, the underlying interface definition should be tailored to the platform or language for each SDK. They main point to illustrate is that there is interaction between the KeyStore and the crypto-provider and that the crypto-provider will be handling the encryption and decryption of data using keys stored securely in a key-store. It is up to the SDK author to design a structure that enables this interaction and allows for multiple algorithms to be supported.

Note, whatever the internal implementation, SDKs MUST be able to encrypt/decrypt fields given the same keys, data, salt, IV, etc. across all SDKs. See section “Example Test Data” below.

#### Failure Scenarios and Exceptions

If during a read there is an error while decrypting the field, the entire read should fail to minimize the risk of an SDK writing back a document with “garbled” data. The same goes for a write; the SDK should fail-fast and return an error back to the application without saving the document.

The following are common configuration and runtime exceptions that should be thrown when the state is invalid:

| Exception Name  | Description          | Message  |
| ------------- |:-------------:| :-----|
|CryptoProviderNotFoundException|No crypto provider can be found for a given alias|The cryptographic provider could not be found for the alias: [ALIAS]|
|CryptoProviderAliasNullException|The annotation has no associated alias or is null or and empty string.|Cryptographic providers require a non-null, empty alias be configured.|
|CryptoProviderMissingPublicKeyException|The PublicKeyName field has not been set in the crypto provider configuration or is null or and empty string|Cryptographic providers require a non-null, empty public and key identifier (kid) be configured for the alias: [ALIAS]|
|CryptoProviderMissingSigningKeyException|The SigningKeyName field has not been set in the crypto provider configuration or is null or and empty string. Required for symmetric algos.|Symmetric key cryptographic providers require a non-null, empty signing key be configured for the alias: [ALIAS]|
|CryptoProviderMissingPrivateKeyException|The PrivateKeyName field has not been set in the crypto provider configuration or is null or and empty string. Required for asymmetric algos.|Asymmetric key cryptographic providers require a non-null, empty private key be configured for the alias: [ALIAS]|
|CryptoProviderSigningFailedException|Thrown if the authentication check fails on the decryption side.|The authentication failed while checking the signature of the message payload for the alias: [ALIAS]|
|CryptoProviderEncryptFailedException|Thrown if an error occurs during encryption.|The encryption of the field failed for the alias: [ALIAS]|
|CryptoProviderDecryptFailedException|Thrown if an error occurs during decryption.|The decryption of the field failed for the alias: [ALIAS]|
|CryptoProviderKeySizeException|Thrown if key size does not match the size of the key that the algorithm expects.|The key found does not match the size of the key that the algorithm expects for the alias: [ALIAS]. Expected key size was [EXPECTE_KEYSIZE] and configured key is [CONFIGURED_KEYSIZE]|

### Field Encryption API

The public API differs slightly between SDK’s, however parameter and method naming must be consistent and follow this RFC.

#### Annotation API

For Java, .NET and other SDKs which use annotations to define the fields to be encrypted, the annotation must be named EncryptedField and take a single string parameter named “provider”. The provider name is an alias which maps to the crypto-provider instance that has been configured. Here are examples for .NET and Java:

**Java**
```Java
public class Person
{
	@EncryptedField(provide=”MyProvider”)
	public String password;
}
```

**C#**
```C#
public class Person
{
	[EncryptedField(provider=”MyProvider”)]
	public string Password {get;set;
}

```

For SDK languages that do not support annotations in this manner, an equivalent marker can be used, but must be named consistent with EncryptField. If a namespace prefix is required by the platform, it is acceptable to add one. For example in GO the prefix is “cb”:


**Go**
```Go
type Person struct {
  	Password string `cbcrypt:myProvider`
}

```

#### Implicit vs Explicit Encryption/Decryption

.NET uses the JSON.NET API to hook into the encryption/decryption process when performing serialization or deserialization a document. There is no explicit requirement for the calling application call any special methods for encryption or decryption. As long as the annotation is in place the encryption or decryption process will happen on the annotated field. For languages or frameworks lacking the sufficient hooks do to this, then the application must explicitly do this by calling special methods and passing in the document to encrypt and decrypt. In this case the method should have the following name and parameters:

| Method      | Optional           | Description  |
| ------------- |:-------------| :-----|
|Encrypt_Fields| document: a json document  fields: An object that describing the fields to be encrypted|A json document with encrypted fields|
|Decrypt_Fields|document: a json document with fields to be decrypted fields: an object describing the fields to be decrypted |A json document with decrypted fields|

**Signature:** 

```
Encrypt_Fields(Json document,  Object fields) : Json

Decrypt_Fields(Json document,  Object fields) : Json
```

**Example:**

```C#
CryptoManager cryptoMgr = new CryptoManager();
cryptoMgr.Register(“MyProvider”, new AesCryptoProvider(new KeyStore(“mykey”)));

var doc = {
	String password;
}

var fields = [{“MyProvider”, “password”}];

var encryptedDoc = cryptoMgr.Encrypt_Fields(doc, fields);
var decryptedDoc = cryptoMgr.Decrypt_Fields(encryptedDoc, fields);
```

Note that the CryptoManager object is an optional, global store for ICryptoProviders and may actually be the configuration object itself. 

#### Field Encryption Format

For consistent encryption/decryption of fields on documents between SDKs, a distinct format has been proposed. The idea is that given a format, the SDK can offer a platform idiomatic public API and implementation; the various SDKs must define their own method handling the encryption/decryption of fields.

The format includes both the temporary field name used to hold the encrypted value, plus additional metadata associated with the algorithm used and the public key:

```
__[PREFIX]_[FIELDNAME]” : {
	“kid” : “[KEY_IDENTIFIER]”,
	“alg”: “[ALGORITHM”],
	“ciphertext”: “[BASE64_ENCRYPTED_DATA]”,
	“sig”: “[BASE64_HMAC_SIGNATURE]”,
	“Iv” : [“INITIALIZATION_VECTOR”]
}
```

The value of sig can be computed as follows:

```
sig = BASE64(HMAC256(kid + alg + BASE64(iv) + BASE64(ciphertext))))
```

Where:

| Name       | Description          |
| ------------- |:-------------:|
|PREFIX | Prefix for temporary field that holds content of encrypted field. This should be configurable so that naming conflicts can be easily resolved at the application level. The default is “crypt”.|
|FIELDNAME |The original name of the field that contained the data to be encrypted |
|“kid”| The “key-identifier” for resolving the key used to decrypt/encrypt from the KeyStore. See section on Key Management above.|
|“alg” |The algorithm used to encrypt/decrypt. |
|“ciphertext” | A Base64 encoded string that is the value from the field that has been encrypted.|
|“sig”|Optional, required for AES. The HMAC signature of the following fields concatenating and then base64 encoding them: “alg”, “iv”, and “ciphertext”. Order is important and must be respected across all implementations |
|“iv”|Optional, required by AES. The Base64 encoded initialization vector used for the cipher-text. |

The purpose for the temporary field for storing the encrypted data is so that we store the encrypted data as a string; if we chose to store in the original field, there may be a type-mismatch if the field type was say a integer. For strongly typed languages this may cause an exception or error when deserializing and the string value is assigned to its original property.

By making the temporary field name a composite of the field name and the string literal “__crypt”, fields that have been encrypted can be easily identified for auditing purposes (see OBJECT_NAMES N1QL function) and the original field can be derived for decrypting when the document is read from the database. 

For example, given a field element on a json document called “foo” with a value of 2:

```JSON
{
	"foo": 2
}
```

The document post-encryption would be stored as follows in Couchbase server:

```JSON
{
	"__crypt_foo": {
		"kid": "thekeyidentifier",
		"alg": "AES-256-HMAC-SHA1",
		"ciphertext": "7TDz8VnL3l7sOSoYvEKYDALOA6+4VPnLsjgapj1JPGS0ADrF3DdPJ7/x5T1OE4813IFhJQjwZw3/vpzdWoE0gw==",
		"sig":"AZjwq/MJ2JfpYkOORG3OT2wV9pu+Wwsdjm9BiAjkXvk=",
		"iv" : "dosi5HmEpoM5LP0Huk55jQ=="
	}
}
```

Upon retrieval from Couchbase, the document should resemble its initial, un-encrypted state with the temporary field removed and the original field populated with the decrypted value. Once again it is up to the SDK to implement a platform idiomatic solution. In the next section, we will go over the .NET implementation of field encryption as an example implementation.

#### Supported Field Types

The encryptable JSON Encoded payload is anything that comes after a field in a JSON document:


| Type     | Example          | Payload |
| ------------- |:-------------:| :-----|
|string|"magicWord":"xyzzy"|“xyzzy"|
|string|“coffeePrice”:”10”|“10”|
|object|"score": {"dance":10,"looks":3}|{"dance":10,"looks":3}|
|numeric|"myint" : 10|10|
|null|"isnull":null|null|

Note that it should be an application error for a child field to be identified for encryption if it’s parent has already been identified for encryption. 

## Ciphers

### AES-256-CBC-HMAC-SHA256
   * Cipher Mode : CBC
   * padding: PKC7S
   * key size: 256
   * block size: 128
   * IV size: 16

The HMAC signature defined in Table 3 as “sig”, is computed as follows:

#### String Example:

|   |       |
| ------------- |:-------------| 
|Key|!mysecretkey#9^5usdk39d&dlf)03sL|
|alg|AES-256-HMAC-SHA256|
|JSON|{ "message": "The old grey goose jumped over the wrickety gate."}|
|IV|Cfq84/46Qjet3EEQ1HUwSg==|
|Ciphertext|sR6AFEIGWS5Fy9QObNOhbCgfg3vXH4NHVRK1qkhKLQqjkByg2n69lot89qFEJuBsVNTXR77PZR6RjN4h4M9evg==|
|HMAC Password|myauthpassword|
|HMAC Password (base64)|bXlhdXRocGFzc3dvcmQ=|
|HMAC Signature|rT89aCj1WosYjWHHu0mf92S195vYnEGA/reDnYelQsM=|
|JSON in database|{"__crypt_message":{"alg": "AES-256-HMAC-SHA256","kid": "mypublickey","ciphertext": "sR6AFEIGWS5Fy9QObNOhbCgfg3vXH4NHVRK1qkhKLQqjkByg2n69lot89qFEJuBsVNTXR77PZR6RjN4h4M9evg==","sig": "rT89aCj1WosYjWHHu0mf92S195vYnEGA/reDnYelQsM=","iv": "Cfq84/46Qjet3EEQ1HUwSg=="}}|

#### Numeric Example:

|   |       |
| ------------- |:-------------| 
|Key|!mysecretkey#9^5usdk39d&dlf)03sL|
|alg|AES-256-HMAC-SHA256|
|JSON|{"message":10}|
|IV|wAg/Z+c81em+to/rR9T3PA==|
|Ciphertext|bvfUk9qkfCYKS2S5CCJPpg==|
|HMAC Password|myauthpassword|
|HMAC Password (base64)|bXlhdXRocGFzc3dvcmQ=|
|HMAC Signature|LAcwxznVSED4zQbuy+UjacQlvtVYvpVmiiAU5gJJASc=|
|JSON in database|{"__crypt_message": {"alg": "AES-256-HMAC-SHA256","kid": "mypublickey","ciphertext": "bvfUk9qkfCYKS2S5CCJPpg==","sig": "LAcwxznVSED4zQbuy+UjacQlvtVYvpVmiiAU5gJJASc=","iv": "wAg/Z+c81em+to/rR9T3PA=="}}|

#### String (numeric) Example:   

|   |       |
| ------------- |:-------------| 
|Key|!mysecretkey#9^5usdk39d&dlf)03sL|
|alg|AES-256-HMAC-SHA256|
|JSON|{"message":"10"}|
|IV|jdqfaa9Hjpd5rTi2BaEWWg==|
|Ciphertext|r6rK6mO0KQ1p9ws/8Feqyg==|
|HMAC Password|myauthpassword|
|HMAC Password (base64)|bXlhdXRocGFzc3dvcmQ=|
|HMAC Signature|NdvUTdR6XhRZQnQBWVzZjv9MxNIbzmwslQqP6onNdVk=|
|JSON in database|{"__crypt_message":{"alg":"AES-256-HMAC-SHA256","kid":"mypublickey","ciphertext":"r6rK6mO0KQ1p9ws/8Feqyg==","sig":"NdvUTdR6XhRZQnQBWVzZjv9MxNIbzmwslQqP6onNdVk=","iv":"jdqfaa9Hjpd5rTi2BaEWWg=="}}|

#### Array Example

|   |       |
| ------------- |:-------------| 
|Key|!mysecretkey#9^5usdk39d&dlf)03sL|
|alg|AES-256-HMAC-SHA256|
|JSON|{"message":["The","Old","Grey","Goose","Jumped","over","the","wrickety","gate"]}|
|IV|A4OSMlz95cvn6ZDypm58jA==|
|Ciphertext|9aTRYMmbNf6tvVFpbedSsS5Hdhk/OjUIz2mEqp5L5EcVNGoKJBhnuaAu35fNVM2YW/7TscXdiUBaeZZv7Zxg1Zve+A1u1/7dmgbkvAilNSo=|
|HMAC Password|myauthpassword|
|HMAC Password (base64)|bXlhdXRocGFzc3dvcmQ=|
|HMAC Signature|PQ25q0k271CpZ9quOg2m3oAIZVa6Mh9S0mo15nN/hlk=|
|JSON in database|{"__crypt_message": {"alg": "AES-256-HMAC-SHA256","kid": "mypublickey","ciphertext": "9aTRYMmbNf6tvVFpbedSsS5Hdhk/OjUIz2mEqp5L5EcVNGoKJBhnuaAu35fNVM2YW/7TscXdiUBaeZZv7Zxg1Zve+A1u1/7dmgbkvAilNSo=","sig": "PQ25q0k271CpZ9quOg2m3oAIZVa6Mh9S0mo15nN/hlk=", "iv": "A4OSMlz95cvn6ZDypm58jA==" }}|

#### Object Example

|   |       |
| ------------- |:-------------| 
|Key|!mysecretkey#9^5usdk39d&dlf)03sL|
|alg|AES-256-HMAC-SHA256|
|JSON|{"message":{"myValue":"The old grey goose jumped over the wrickety gate.","myInt":10}}|
|IV|kKz59c/ObctWVBVen7VqHw==|
|Ciphertext|3Av4Pvl53VKYliviKLDTh7bCAnaOs2UumoEP4ruGTRK2v/20hkYF8+jPtIhGIpwowSxGRv0DvnC75osW9mFd3D30yBme4RJUKDml9l4Zty4=|
|HMAC Password|myauthpassword|
|HMAC Password (base64)|bXlhdXRocGFzc3dvcmQ=|
|HMAC Signature|inEyX1OYv2VbO60o16zId8eacwE8E1KrKcfgQVlz4kA=|
|JSON in database|{"__crypt_message":{"alg": "AES-256-HMAC-SHA256","kid": "mypublickey","ciphertext": "LIoZ3qPbEAmNbUUsJeRngN2tDo8/gU1AQ+yY88sgcdACMz/DjD8+kAzbZHYiWswGSlDaEHXs9c2hIYiD2/AkX/qfaGcAZLt4rjZpbzuNf+Q=","sig": "inEyX1OYv2VbO60o16zId8eacwE8E1KrKcfgQVlz4kA=","iv": "kKz59c/ObctWVBVen7VqHw=="}}|

Note that in the table above, IV, Ciphertext and HMAC Signature are all Base64 encoded strings. The Key, Plaintext and HMAC password are UTF8.

### RSA-2048-OAEP-SHA1
  * KeySize: 2048
  * Padding: OAEP-SHA1

Key pair for examples and verification (**DO NOT REUSE!**):

-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAwP6s/siq+geZAcN858as1U6VIFeNDjvepl88jyd748idDt1a
hDqw7pGw5WMygq04anWQG3kKUUhElxwG9BJ/z4rxJXO0Vbflv0whgBlTVVxXuXSP
wtyA200CENLO6aTaVN/aettSvA3cEuTit6eg4Ayi0iSO97SI/9Jp4XeI4bA5Ls55
1Y9XR+PVbnaNgDWxGvebpw9GvjeK/hUdMHwP8QhLdyLLjbQ6i3YxOWFYWqjtSQav
CdkpHNui7U1rULxYYFSAhR64dOwoTs2yB8lLMQsjTdIQR6oQZgaKRlVzPzHlJgp0
tISJxvJYXrct7ZEjEFtTLnOMx4E7MbmcN3bsDwIDAQABAoIBAGiiq5CHo4tjyyUV
pAbVxKbxsBCU5zksZI63W9IRii35eo2wnX7Lg1oVS19S5PPMjqXJj5QVj+55zBZR
b8Oss/cGUbAIh2FiDwIkeJVHJdNF+ZnnBHqVqpc7rT8JzH0IkAcsRvwNJVIoAYWM
6w6/p41RzIU6pPjPvOdWYWmIsYIKZAhVnTf8QXDBpBdjzrrlTnocChNtEdkqyCFm
FILOWUiFbzWsHJe5/1o+v+Kw4qQGHNZVpFi2vQCJxTLdEbcUHCmVqgQOs+1hs+Ax
37pkXfVBRh97E5RV0Os8JtH3smw9uCcQveJanmuPVhsa+8zjOK2j1AHjdsaPZgMP
wuleVoECgYEA5mJ72lPRcFjNTTDQfLUHxCq4rWekkS+QsgPyBuE8z5mi7SsHuScV
i+PcLehRY8e6Z464Kl9Ni6c17HcM+Bm8ay70hxPeTlqrVBjxKiTixF1BccbBHPjd
Jl0WCEODxKMp5TAasJsfM7Pg18cYNakmOqK/agc7LJtsyo99jKfFYR8CgYEA1nPz
mGfhZZ2JXsNBNlqyvitV7Uzwa62DMGJUuosODnaz5v4gTPZhMF4gaOwh3wofP882
JZM1YEDF64Nn3tMDXImidoE9tKDMPyT1+obaBEPe8AfhAJGfMrWHgU3Yicd16bxK
vbU3kODpFgBtnE50JcceEyFYTWzZeNRWlsW4ZxECgYEAqTxDGthji5HQDhoDrPgW
omV3j/oIi5ZTRlFbou4mC6IiavInFD2/uClD/n0f/JolNhlC8+1aO3IzTGcPodjV
7i5p9igEL66vGHHSBlFeOzz97CRCi5PMcHgEzUE7NGFfTzqNAJqSyxoh2qAoCpMc
wAn5blutflEWE55gbch4V6UCgYEArFiBU2FgzmZd6O9ocENR1O1E4DHuIctPXEoa
J9TrFgqlqCVhVhjHoLR0vX3P9szOslxX+riks9c6eHyhtHzG/c6K50wUiB6WJsUQ
fidz/OuCtkrOs8NUOs+SuAMU3B2VkKPHOVDy+BcYm5r6fBy80UOF0wAAVDD/UVDs
ybza5tECgYB+ksZiUbZ+8WTXVIB3HJJT8U8bZ8676lrRUJ5YxB+avHh4g/TI+e53
jZKBVvB3Mhp6QFMZITuUTRgiGuAjBap4SZ32Pmyu3TxiWDxKktmvFMPLUVFntDJ0
th2u9Xpw8+T01AOCFc0PKtC8g0Covxu+qWLfqnJnTCx+Q03+dQj9rQ==
-----END RSA PRIVATE KEY-----
 

-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwP6s/siq+geZAcN858as
1U6VIFeNDjvepl88jyd748idDt1ahDqw7pGw5WMygq04anWQG3kKUUhElxwG9BJ/
z4rxJXO0Vbflv0whgBlTVVxXuXSPwtyA200CENLO6aTaVN/aettSvA3cEuTit6eg
4Ayi0iSO97SI/9Jp4XeI4bA5Ls551Y9XR+PVbnaNgDWxGvebpw9GvjeK/hUdMHwP
8QhLdyLLjbQ6i3YxOWFYWqjtSQavCdkpHNui7U1rULxYYFSAhR64dOwoTs2yB8lL
MQsjTdIQR6oQZgaKRlVzPzHlJgp0tISJxvJYXrct7ZEjEFtTLnOMx4E7MbmcN3bs
DwIDAQAB
-----END PUBLIC KEY-----

#### String Example:

|||
|--- |--- |
|PublicKeyName|MyPublicKeyName|
|PrivateKeyName|MyPrivateKeyName|
|alg|RSA-2048-OAEP-SHA1|
|JSON|{ "message": "The old grey goose jumped over the wrickety gate."}|
|JSON in database|{ "__crypt_message": { "alg": "RSA-2048-OAEP-SHA1",  "kid":"MyPublicKeyName", "ciphertext": "JMAfWsDrRdrF/ghjq4xlysN7F6TogIWhkf8Fa6Qvfhqq+syKXmtviIMVSBOXPJvZm8gT/wyryjqBrLFPK9AeeS2mI4FsPCbZzvRS85f12GS0TNIp71wW1R2BNFt+51Oa5jD1SOGT/qrBbpFWICVh2z+8AUbxirrobAFn3L179sAmcj8mP3Hl0+4YeTMx/DI333bsGvpB7bpk4U28/38PMhR8Uc1xjXpW1NUwfqho4PUPMumDPUa5e8p2DMUaGzgl6PVZmX3xqHGpBviWqcKGORQhOsWfI/45IDaWJlk7eHv8yuvJleKKd1cUkkRfZ3R1bui2F9S/HvJdQKClfHc3Ow==" }}|

#### Numeric Example

|||
|--- |--- |
|PublicKeyName|MyPublicKeyName|
|PrivateKeyName|MyPrivateKeyName|
|alg|RSA-2048-OAEP-SHA1|
|JSON|{"message":10}|
|JSON in database|{"__crypt_message": { "alg": "RSA-2048-OAEP-SHA1", "kid": "MyPublicKeyName", "ciphertext": "NTxRjexminStAPC2fv6jXbaLPk4wxOkI8xU4EgmR05qZzmfjN+U1e5Ipi9mC/dOk0RfFklS+30ss45GkpZZnYjlEKQ9gI4ZmTb5Za42SHB6hhrR1ZXCIGfH4UhUwQiM+XzHLvOOiTnxVcSQd3TgL/hlTUbUsIWsqrm5Q9O1R+h8suEjcOnu4mmRI1qMdQLSKXPtqa8N1u00F24QNtS79UeWaZFVqll7FyESyJaz86ZS1/0NXwkfCwPRD0iP7Q/mfKh5+Vtl22PM9k1ar3aHbkJhE11Pm5y7w0Z9K1X73CmcSWYBuOY/SDpIBmqLYtv4o1ANB+bMv7yo+uoCouFrD/A=="}}|

#### String (numeric) Example:

|||
|--- |--- |
|PublicKeyName|MyPublicKeyName|
|PrivateKeyName|MyPrivateKeyName|
|alg|RSA-2048-OAEP-SHA1|
|JSON|{"message":"10"}|
|JSON in database|{ "__crypt_message": {  "alg": "RSA-2048-OAEP-SHA1",  "kid": "MyPublicKeyName",  "ciphertext": "r9y9pGvNYq6/Gv9k6jHQW74/C5zhWmWVlTLQ5cN4KuYDoBNnbZDJ3+bS52Sn14tc4vdHviSAyfm5CelrBu44y3s8nzOVi0D+yeY4rTLCZdEXjncLkS6UaCzr3CjBlzPRyJ6VnWSPayt+ZOi4UJywMSzkIzLXlib0gsjbG5csd4xLgVu+exSsbWkzQSMRnIT6ej87Zawa7rJcNpwk+hl8tsnG1S/4XYi3juq9ddVnULnoJhd7/2PErorgrQ5ol3Gz47t3f6j3kRrtJZlx6XCRwbEw+Mi5sIwZfTweoil9QnIvJHiYb0LMl0oYRA0dVOQNL57fHbT7xINqh8mKuWEjow==" }}|

#### Array Example

|||
|--- |--- |
|PublicKeyName|MyPublicKeyName|
|PrivateKeyName|MyPrivateKeyName|
|alg|RSA-2048-OAEP-SHA1|
|JSON|{"message":["The","Old","Grey","Goose","Jumped","over","the","wrickety","gate"]}|
|JSON in database|
{  "__crypt_message": { "alg": "RSA-2048-OAEP-SHA1", "kid": "MyPublicKeyName",  "ciphertext": "KlYYMyaJRZvNr+tyoK5E75lE+QWSrsvmraBoapl/l9RRHkjien7+AqcmVsS/dRRa9Ad8dmyRvaOA9B46TsJ2FbzNJ8cVNTyLPdAeluU9aM0IiIuMfEYFc5XNC2clpQlsVgxMutiO0wiCEvFX3iNIvZFeYUQofKoe0H1VlyGGZbfLLdNfl+Rlui0IULFTW6UZEnmiIlTxffnGdvlwWlaTpJMfTAIYOieZiPbsraGgjIpPrQtXVSyy/bSqmOp2eva+X7dtD7R3vAHlRptvD4Muhp3jaxIQj0J4NsD4Gw+muHFYG1YnsdJERyWlkMQmnJt89XPL2VUD6ni2Q8TyxFm0LA=="}}|

#### Object Example

|||
|--- |--- |
|PublicKeyName|MyPublicKeyName|
|PrivateKeyName|MyPrivateKeyName|
|alg|RSA-2048-OAEP-SHA1|
|JSON|{"Message":{"MyValue":"The old grey goose jumped over the wrickety gate.","MyInt":10}}|
|JSON in database|{ "__crypt_message": { "alg": "RSA-2048-OAEP-SHA1", "kid": "MyPublicKeyName", "ciphertext": "E2Tlzl6MFTYnnKip7ENdZL8NuA/XkWPllWXu/nws4lYKHxVg8A1XYo+Q229q145glk73S01QmHbB0CZXbzvTo/BgZBb3but1U97qsoPnajFo6BVvigpCt6gaZnYSHuoXsB4L/JBRJuw+0cLX5sr7PsRb5WTV3eTCo/ja2jnqUSOCbWmwasqBY49dvSuJfTwWLcgOWJeg58AZoAGZEqAkuavoxD/+vtRXbFLXO3qdQ4XhsJjnttnaQnjgpKJOzYYwhpF7U0pW0YzT7PvtbJdguMeiYwd0Ypt5L7WiLr89ft0RFDO6K+x66fnxk4hM1c/xeCOAlNR/Mu75ke1/QZpnGg==" }}|

## Language Specifics

### .NET Field Encryption Example

```C#
public class PocoWithObject
{
	[EncryptedField(Provider = "AES-256-HMAC-SHA256")]
	public InnerObject Message { get; set; }
}

public class InnerObject
{
	public string MyValue { get; set; }

	public int MyInt { get; set; }
}

var key = "!mysecretkey#9^5usdk39d&dlf)03sL";

var config = new ClientConfiguration(TestConfiguration.GetConfiguration());
config.EnableFieldEncryption(new AesCryptoProvider(new InsecureKeyStore(
	new KeyValuePair<string, string>("mypublickey", key),
	new KeyValuePair<string, string>("myauthsecret", "myauthpassword")))
{
	KeyName = "mypublickey",
	PrivateKeyName = "myauthsecret"
});

using (var cluster = new Cluster(config))
{
	cluster.Authenticate("Administrator", "password");
	var bucket = cluster.OpenBucket();

	var poco = new PocoWithObject
	{
		Message = new InnerObject
		{
			MyInt = 10,
			MyValue = "The old grey goose jumped over the wrickety gate."
		}
	};
	var result = bucket.Upsert("mypocokey", poco);

	var get = bucket.Get<PocoWithObject>("mypocokey");
}

```

### Java Field Encryption Example

```Java
EncryptionConfig encryptionConfig = new EncryptionConfig();
encryptionConfig.addCryptoProvider(rsaCryptoProvider);
encryptionConfig.addCryptoProvider(aes128CryptoProvider);

CouchbaseEnvironment environment = DefaultCouchbaseEnvironment.builder()
      .encryptionConfig(encryptionConfig)
      .build();

JsonDocument document = JsonDocument.create("testStringEncryptAES",
        JsonObject.create().put("foo", "bar", aes128CryptoProvider.getProviderName()));
bucket.upsert(document);


public static class Entity {

    @Id
    public String id;

    @EncryptedField(provider = "AES-128")
    public String stringa;

    @EncryptedField(provider = "RSA")
    public String stringr;

}
```

### Node.js Field Encryption Example

Node.js does not provide any features which would enable document structure annotations or document type deduction.  This means that implementing any method of _automatic_ document encrypt/decrypt is likely not possible.  In its stead, we will implement methods to enable before-store and after-retrieve cryptographics.  For example:

```Javascript
bucket.get(‘somekey’, function(err, res) {
  var doc = cbfieldcrypt.decrypt(res.value, {
    ‘fieldname’: new cbfieldcrypt.AesCryptoProvider(keystore, ‘keyname’)
  })
});
```
### libcouchbase Field Encryption Example

#### ENCRYPT

```C
static void store_encrypted(lcb_t instance, const char *key, const char *val)
{
    lcb_error_t err;
    lcb_CMDSTORE cmd = {};
    lcbcrypto_CMDENCRYPT ecmd = {};
    lcbcrypto_FIELDSPEC field = {};

    printf("KEY:    %s\n", key);
    printf("PLAIN:  %s\n", val);

    ecmd.version = 0;
    ecmd.prefix = NULL;
    ecmd.doc = val;
    ecmd.ndoc = strlen(val);
    ecmd.out = NULL;
    ecmd.nout = 0;
    ecmd.nfields = 1;
    ecmd.fields = &field;
    field.name = "message";
    field.alg = "AES-256-HMAC-SHA256";

    err = lcbcrypto_encrypt_document(instance, &ecmd);
    if (err != LCB_SUCCESS) {
        die(instance, "Couldn't encrypt field 'message'", err);
    }
    printf("CIPHER: %s\n", ecmd.out);

    LCB_CMD_SET_KEY(&cmd, key, strlen(key));
    LCB_CMD_SET_VALUE(&cmd, ecmd.out, ecmd.nout);
    cmd.operation = LCB_SET;
    cmd.datatype = LCB_DATATYPE_JSON;

    err = lcb_store3(instance, NULL, &cmd);
    free(ecmd.out); // NOTE: it should be compatible with what providers use to allocate memory
    if (err != LCB_SUCCESS) {
        die(instance, "Couldn't schedule storage operation", err);
    }
    lcb_wait(instance);
}
```

#### DECRYPT

```C
static void get_callback(lcb_t instance, int cbtype, const lcb_RESPBASE *rb)
{
    if (rb->rc == LCB_SUCCESS) {
        const lcb_RESPGET *rg = (const lcb_RESPGET *)rb;
        lcbcrypto_CMDDECRYPT dcmd = {};
        lcb_error_t err;
        lcbcrypto_FIELDSPEC field = {};

        printf("VALUE:  %.*s\n", (int)rg->nvalue, rg->value);
        dcmd.version = 0;
        dcmd.prefix = NULL;
        dcmd.doc = rg->value;
        dcmd.ndoc = rg->nvalue;
        dcmd.out = NULL;
        dcmd.nout = 0;
        dcmd.nfields = 1;
        dcmd.fields = &field;
        field.name = "message";
        field.alg = "AES-256-HMAC-SHA256";
        err = lcbcrypto_decrypt_document(instance, &dcmd);
        if (err != LCB_SUCCESS) {
            die(instance, "Couldn't decrypt field 'message'", err);
        }
        if (dcmd.out == NULL) {
            die(instance, "Crypto provider returned success, but document is NULL", LCB_EINVAL);
        }
        printf("PLAIN:  %.*s\n", (int)dcmd.nout, dcmd.out);
        free(dcmd.out); // NOTE: it should be compatible with what providers use to allocate memory
        printf("CAS:    0x%" PRIx64 "\n", rb->cas);
    } else {
        die(instance, lcb_strcbtype(cbtype), rb->rc);
    }
}

static void get_encrypted(lcb_t instance, const char *key)
{
    lcb_CMDGET cmd = {};
    lcb_error_t err;
    LCB_CMD_SET_KEY(&cmd, key, strlen(key));
    printf("KEY:    %s\n", key);
    err = lcb_get3(instance, NULL, &cmd);
    if (err != LCB_SUCCESS) {
        die(instance, "Couldn't schedule get operation", err);
    }
    lcb_wait(instance);
}
```

## Testing Notes

See: “Ciphers” section above - each table contains necessary data.

## SDK Verification

Verification that each SDK’s implementation of the FLE matches the RFC.


|SDK |References| Verified|
|---|---|---|
|GO| https://github.com/couchbaselabs/gocbfieldcrypt/blob/master/basic_test.go ||
|Node| https://github.com/couchbaselabs/node-cbfieldcrypt/blob/master/test/basic.test.js https://github.com/couchbaselabs/devguide-examples/pull/18 ||
|.NET| https://github.com/couchbaselabs/devguide-examples/blob/master/dotnet/FieldEncryption.cs||
|Java| https://developer.couchbase.com/documentation/server/5.5/sdk/java/encrypting-using-sdk.html||
|PHP| https://github.com/couchbaselabs/devguide-examples/blob/master/php/encryption/demo.php||
|LCB| https://github.com/couchbase/libcouchbase/tree/master/example/crypto||
|Python| https://github.com/couchbase/couchbase-python-client/blob/master/couchbase/tests/cases/crypto_t.py||


## Future Work

There are a few items out of scope at this time for v1.0, however they very much likely will be covered in future iterations based upon customer feedback, PM requests, etc:

  * If a N1QL Query or FTS result returns an encrypted field, the work under this effort will not deliver API to decrypt the field.  Individual SDKs may provide volatile/uncommitted API at their option.
  * Connectors such as Kafka and Elastic…
  * Key rolling/management scenarios/use-cases are not covered
  * Algorithm upgrading - for example going from AES-256 to AES-512
  * Support for KMIP based key stores

## Sign Off

|Language |Representative |Date |
|:--- |:--- |:--- |--- |
|C|Sergey Avseyev|2018-06-25|
|Go|Brett Lawson|2018-06-25|
|Java|Subhashni Balakrishnan| - |
|.NET|Jeff Morris|2018-06-22|
|NodeJS|Brett Lawson|2018-06-25|
|PHP|Sergey Avseyev|2018-06-25|
|Python|Ellis Breen|22/6/2018|


