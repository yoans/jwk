[![Build Status](https://travis-ci.org/UberEther/jwk.svg?branch=master)](https://travis-ci.org/UberEther/jwk)
[![NPM Status](https://badge.fury.io/js/uberether-jwk.svg)](http://badge.fury.io/js/uberether-jwk)

# Overview

This library builds upon [node-jose](https://github.com/cisco/node-jose) for the low level operations and adds top-level helpers to support dynamic loading of JWK-sets and simplify calls to crypto operations.

This library provides a class to manage JWK key-sets and perform JKA and JKS operations using those keys.

Keys can be loaded from a URL, or imported from manually loaded JSON.  Individual keys may also be loaded.  URL loaded keys may be automatically refreshed as needed.

All asynchronous methods return Bluebird promises.  If you use bound promises, be aware that these promises will be bound to the objects they were generated by.



# Examples of use:

```js
JWK = require("uberether-jwk");
jwk = new JWK({ jwkSetUrl: "http://jwk.example.com/foo" });

jwk.getKeyAsync("exampleKid")
.then(function(key) { return key.ensureSignature(data, signature); })
.then(function() { /* Signature valid */ })
.catch(function(err) { /* Handle errors */ });

// Direct use of node-jose:
jwk.refreshIfNeededAsync()
.then(function(keystore) {
	return JWK.jose.JWE.createDecrypt(decrypt).verify("xxxxx", "utf8");
});

```

# API

## JWK Set Manager

A class that manages the key sets.  This is the main export for the library.

Key sets may be loaded from a URL or manually loaded.
- If loaded from a URL, you may specify a refresh and expiration period.  When the refresh expires, the next request will load update JKS data in background (without blocking the current request).  If the expire is reached, the next request will wait for the JKS data to reload before proceeding.
    - Refresh and reloads of the URL only occur upon a request.  They are not performed on a timer.
    - Last modified and etag information is tracked and used to prevent re-transmission and processing of data if the JWK set has not been modified.
    - If a request is made for a key that does not exist, then by default the code will refresh from the URL immediately.

### JWK.jose
The instance of Jose used by JWK.  It defaults to the copy loaded by require("node-jose") but can be modified or replaced by the caller.

### new JWK(options)

Constructs a new JWK set manager.

Options Available:
- jwkSetUrl - URL to load the key set from
- request - Optional promisified copy of the "request" library to use.  If not specified, our own internal one is used.
- requestDefaults - Optional default options to inject into all HTTP requests.  See the "request" library's "defaults" method
- refreshDuration - Duration before a background refresh of jwkSetUrl is made.  May be specified in milliseconds OR as a string compatible with the "ms" library.  Default is 1 day.
- expireDuration - Duration before a background refresh of jwkSetUrl is made.  May be specified in milliseconds OR as a string compatible with the "ms" library.  Defaylt is 7 days.

**NOTE:** It is STRONGLY recommended that IF you are setting jwkSetUrl:
- DO register a handler for refreshError
- DO NOT use importJwkSet in the same instance

### JWK.changeJwkSetUrl(newUrl)
Change the URL for loading the JWK set from.  The old keys are not immediately cleared but any loaded data is immediately marked as expired and will be loaded upon the next request.

If the newUrl is falsey, then the existing keys will be retained but no further refreshes will be made.

### JWK.clear()

Clears all loaded and remembered keys.  Next request will force a reload from the jwkSetUrl, if one was provided.

### JWK.refreshIfNeededAsync()

This method is intended for internal use but could in some cirmstaunces be useful to applications.

It will return a promise that is resolved to the node-jose keystore once the JWK data is ready to use.
- If the data is expired (or has never been loaded) then this will be after the URL is loaded and parsed
- If the data is due for refresh, a refresh will be started but the promise will return immediately.

### JWK.refreshNowAsync()

Refreshes from the URL now, returning a promise that is resolved to the node-jose keystore upon completion of the reload.
- If no URL is specified, it will resolve immediately.
- If another refresh is pending, it will return the pending promise.

### JWK.importJwkSetAsync(jwkSet)

Replaces the current key data with the data from the set.  Any remembered keys are added.
- jwkSet must be a valid JWK set JSON object

DO NOT USE BOTH jwkSetUrl AND this API.  The library will allow this but you will almost certainly not get the results you desire.

### JWK.addKeyAsync(key, remember = true)

Adds a key to the key set.
- key may be either a JWK JSON object OR a JWK.Key compatible object
- If remember is true, then the key will be readded whenever a new JWK set JSON is loaded

Returns the node-jose key object for the added key

### JWK.forgetKey(key)

Removes the node-jose key specified from the keystore.

### JWK.replaceKeyAsync(oldKey, newKey)

Replaces the specified key with a new key.  Returns a promse that resolves to the new node-jose key

### JWK.getKeyAsync()

Returns a promise that resolve to a key from the keystore.  Arguments are as per [node-jose.get](https://github.com/cisco/node-jose#retrieving-keys).

If the key is not found, the keystore will refresh.  If the key is still not found, undefined will be returned.

### JWK.toJsonAsync(exportPrivate)
Returns a promise that resolves to the JSON representing the keystore.  If exportPrivate is true, then private keys are also exported.

### JWK.generateKeyAsync(kty, size, props, remember = true)
Generates a new key and saves it in the keystore.  The first three arguments match [node-jose.generate](https://github.com/cisco/node-jose#managing-keys).

The returned promise resolves to the new node-hose key.

### JWK.verifySignatureAsync(input)
Parses a JWS signed payload and returns the parsed data.  Throws an exception if the signature fails to verify.

If the key is not found, the keystore is refreshed.

An exception is thrown on any validation failures.  Return values are as per [node-jose.JWS.createVerify](https://github.com/cisco/node-jose#verifying-a-jws).

### JWK.signAsync(key, content, options)
Signs the specified content with the node-jose key specified.

- options.encoding is the encoding to encode the content with (if it is a string).  Defaults to "utf8".
- See [node-jose.JWS.createSign](https://github.com/cisco/node-jose#signing-content) for more details on other options.

Returns a promise that resolves to the signed object.

### JWK.decryptAsync(input)
Decrypts a JWE encrypted payload and returns the enveloped data.

If the key is not found, the keystore is refreshed.

Return values are as per [node-jose.JWS.createDecrypt](https://github.com/cisco/node-jose#decrypting-a-jwe).

### JWK.encryptAsync(key, content, options)
Encrypts the specified content with the node-jose key specified.

- options.encoding is the encoding to encode the content with (if it is a string).  Defaults to "utf8".
- See [node-jose.JWS.createEncrypt](https://github.com/cisco/node-jose#encrypting-content) for more details on other options.

Returns a promise that resolves to the encrypted object.

### JWK Event: beforeClear

Called before a clear is performed.  May be used to cleanup or preserve data before the clear is executed.

### JWK Event: cleared

Called after a clear is performed.  May be used to perform cleanup.

### JWK Event: beforeRefresh, reqOpts

Called just before the refresh HTTP request is made.  The request options are passed in and may be modified by events, although the use of the requestDefaults options on the constructor is preferred.

Warning: Not every beforeRefresh will match with a refreshed event.  If the data is unmodified, then the refreshed event will not be triggered.

### JWK Event: refreshed

Called after JWK data has been replaced with reloaded data from the requested URL

### JWK Event: refreshError

Emitted if the background refresh fails.  Allows the application to trap and handle the error.  If there are no listeners for refreshError, then the promise will reject with the error.
- If the error is not handled, this will result in an unhandled rejection from Bluebird.
- If you do handle the error, anybody waiting on the promises will be resolved and proceed to process with the old data (which is all we have to work with).

It is strongly recommended you trap refreshError and handle this.

### JWK Event: keyAdded, key, remember

Called after a key is added.  Values for the new key, old key (if one was replaced) and remember (if requested) are passed as arguments

### JWK Event: keyForgotten, oldKey

Called after a key is forgotten.  The old key JWK.Key object is passed as an argument.

### JWK Event: beforeJwkSetImport, jwkSet, meta

Called just before import a JWK set (both via the API and via the URL).  The raw jwkSet object and raw metadata object are passed in.

### JWK Event: jwkSetImported

Called just after the new JWK set has replaced the prior set.

### JWK Event: urlChanged, newUrl, oldUrl

Called just after the URL is modified.  Old and new values are provided.



# Contributing

Any PRs are welcome but please stick to following the general style of the code and stick to [CoffeeScript](http://coffeescript.org/).  I know the opinions on CoffeeScript are...highly varied...I will not go into this debate here - this project is currently written in CoffeeScript and I ask you maintain that for any PRs.