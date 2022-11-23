# React Native Signal Protocol

> A fork of AidenRourke's [signal-protocol](https://github.com/AidenRourke/signal-protocol)

> Make sure to call `await`


### Installation

Install [Expo-CLI](https://docs.expo.dev/get-started/installation/) first.

```shell
npm install signal-protocol-react-native
expo install expo-crypto expo-random
```

### Setup

**VERY IMPORTANT**: In your project's root `index.js` file, import `expo-random`, `react-native-get-random-values`, and `react-native-securerandom`.
This allows your app to asynchronously load required modules to check secure key generation status.

```javascript
import {AppRegistry} from 'react-native';
import App from './App';
import {name as appName} from './app.json';

// START INSERT
import 'react-native-get-random-values';
import 'react-native-securerandom';
import 'expo-random';
// END INSERT

/// ...
AppRegistry.registerRootComponent(appName, () => App);
```

### Usage:

```javascript
import signal from 'signal-protocol-react-native';
```


```javascript
var KeyHelper = signal.KeyHelper;

function generateIdentity(store) {
    return Promise.all([
        KeyHelper.generateIdentityKeyPair(),
        KeyHelper.generateRegistrationId(),
    ]).then(function(result) {
        store.put('identityKey', result[0]);
        store.put('registrationId', result[1]);
    });
}

function generatePreKeyBundle(store, preKeyId, signedPreKeyId) {
    return Promise.all([
        store.getIdentityKeyPair(),
        store.getLocalRegistrationId()
    ]).then(function(result) {
        var identity = result[0];
        var registrationId = result[1];

        return Promise.all([
            KeyHelper.generatePreKey(preKeyId),
            KeyHelper.generateSignedPreKey(identity, signedPreKeyId),
        ]).then(function(keys) {
            var preKey = keys[0]
            var signedPreKey = keys[1];

            store.storePreKey(preKeyId, preKey.keyPair);
            store.storeSignedPreKey(signedPreKeyId, signedPreKey.keyPair);

            return {
                identityKey: identity.pubKey,
                registrationId : registrationId,
                preKey:  {
                    keyId     : preKeyId,
                    publicKey : preKey.keyPair.pubKey
                },
                signedPreKey: {
                    keyId     : signedPreKeyId,
                    publicKey : signedPreKey.keyPair.pubKey,
                    signature : signedPreKey.signature
                }
            };
        });
    });
}

var ALICE_ADDRESS = new signal.SignalProtocolAddress("xxxxxxxxx", 1);
var BOB_ADDRESS   = new signal.SignalProtocolAddress("yyyyyyyyyyyyy", 1);

    var aliceStore = new signal.SignalProtocolStore();

    var bobStore = new signal.SignalProtocolStore();
    var bobPreKeyId = 1337;
    var bobSignedKeyId = 1;

    var Curve = signal.Curve;

        Promise.all([
            generateIdentity(aliceStore),
            generateIdentity(bobStore),
        ]).then(function() {
            return generatePreKeyBundle(bobStore, bobPreKeyId, bobSignedKeyId);
        }).then(function(preKeyBundle) {
            var builder = new signal.SessionBuilder(aliceStore, BOB_ADDRESS);
            return builder.processPreKey(preKeyBundle).then(function() {

              var originalMessage = util.toArrayBuffer("my message ......");
              var aliceSessionCipher = new signal.SessionCipher(aliceStore, BOB_ADDRESS);
              var bobSessionCipher = new signal.SessionCipher(bobStore, ALICE_ADDRESS);

              aliceSessionCipher.encrypt(originalMessage).then(function(ciphertext) {

                  // check for ciphertext.type to be 3 which includes the PREKEY_BUNDLE
                  return bobSessionCipher.decryptPreKeyWhisperMessage(ciphertext.body, 'binary');

              }).then(function(plaintext) {

                  alert(plaintext);

              });

              bobSessionCipher.encrypt(originalMessage).then(function(ciphertext) {

                  return aliceSessionCipher.decryptWhisperMessage(ciphertext.body, 'binary');

              }).then(function(plaintext) {

                  assertEqualArrayBuffers(plaintext, originalMessage);

              });

            });
        });

```

### Generate an identity + PreKeys

This protocol uses a concept called 'PreKeys'. A PreKey is an ECPublicKey and
an associated unique ID which are stored together by a server. PreKeys can also
be signed.

At install time, clients generate a single signed PreKey, as well as a large
list of unsigned PreKeys, and transmit all of them to the server.

Please note that before running any command that involved `crypto.getRandomValues` you must first call and **await** `KeyHelper.ensureSecure` (see [isomorphic-webcrypto](https://github.com/kevlened/isomorphic-webcrypto)) for more details.

```javascript
import signal from 'signal-protocol-react-native';

var KeyHelper = signal.KeyHelper;

var registrationId = KeyHelper.generateRegistrationId();
// Store registrationId somewhere durable and safe.

KeyHelper.generateIdentityKeyPair().then(function(identityKeyPair) {
    // keyPair -> { pubKey: ArrayBuffer, privKey: ArrayBuffer }
    // Store identityKeyPair somewhere durable and safe.
});

KeyHelper.generatePreKey(keyId).then(function(preKey) {
    store.storePreKey(preKey.keyId, preKey.keyPair);
});

KeyHelper.generateSignedPreKey(identityKeyPair, keyId).then(function(signedPreKey) {
    store.storeSignedPreKey(signedPreKey.keyId, signedPreKey.keyPair);
});

// Register preKeys and signedPreKey with the server
```

### Build a session

Signal Protocol is session-oriented. Clients establish a "session," which is
then used for all subsequent encrypt/decrypt operations. There is no need to
ever tear down a session once one has been established.

Sessions are established in one of two ways:

1. PreKeyBundles. A client that wishes to send a message to a recipient can
   establish a session by retrieving a PreKeyBundle for that recipient from the
   server.
1. PreKeySignalMessages. A client can receive a PreKeySignalMessage from a
   recipient and use it to establish a session.

#### A note on state

An established session encapsulates a lot of state between two clients. That
state is maintained in durable records which need to be kept for the life of
the session.

State is kept in the following places:

* Identity State. Clients will need to maintain the state of their own identity
  key pair, as well as identity keys received from other clients.
* PreKey State. Clients will need to maintain the state of their generated
  PreKeys.
* Signed PreKey States. Clients will need to maintain the state of their signed
  PreKeys.
* Session State. Clients will need to maintain the state of the sessions they
  have established.

A signal client needs to implement a storage interface that will manage
loading and storing of identity, prekeys, signed prekeys, and session state.
See `test/InMemorySignalProtocolStore.js` for an example.

#### Building a session

Once your storage interface is implemented, building a session is fairly straightforward:

```js
var store   = new MySignalProtocolStore();
var address = new signal.SignalProtocolAddress(recipientId, deviceId);

// Instantiate a SessionBuilder for a remote recipientId + deviceId tuple.
var sessionBuilder = new signal.SessionBuilder(store, address);

// Process a prekey fetched from the server. Returns a promise that resolves
// once a session is created and saved in the store, or rejects if the
// identityKey differs from a previously seen identity for this address.
var promise = sessionBuilder.processPreKey({
    registrationId: <Number>,
    identityKey: <ArrayBuffer>,
    signedPreKey: {
        keyId     : <Number>,
        publicKey : <ArrayBuffer>,
        signature : <ArrayBuffer>
    },
    preKey: {
        keyId     : <Number>,
        publicKey : <ArrayBuffer>
    }
});

promise.then(function onsuccess() {
  // encrypt messages
});

promise.catch(function onerror(error) {
  // handle identity key conflict
});
```


### Encrypting

Once you have a session established with an address, you can encrypt messages
using SessionCipher.

```js
var plaintext = "Hello world";
var sessionCipher = new signal.SessionCipher(store, address);
sessionCipher.encrypt(plaintext).then(function(ciphertext) {
    // ciphertext -> { type: <Number>, body: <string> }
    handle(ciphertext.type, ciphertext.body);
});
```

### Decrypting

Ciphertexts come in two flavors: WhisperMessage and PreKeyWhisperMessage.

```js
var address = new signal.SignalProtocolAddress(recipientId, deviceId);
var sessionCipher = new signal.SessionCipher(store, address);

// Decrypt a PreKeyWhisperMessage by first establishing a new session.
// Returns a promise that resolves when the message is decrypted or
// rejects if the identityKey differs from a previously seen identity for this
// address.
sessionCipher.decryptPreKeyWhisperMessage(ciphertext).then(function(plaintext) {
    // handle plaintext ArrayBuffer
}).catch(function(error) {
    // handle identity key conflict
});

// Decrypt a normal message using an existing session
var sessionCipher = new signal.SessionCipher(store, address);
sessionCipher.decryptWhisperMessage(ciphertext).then(function(plaintext) {
    // handle plaintext ArrayBuffer
});
```

## Cryptography Notice

 A number of nations restrict the use or export of cryptography. If you are potentially subject to such restrictions you should seek competent professional legal advice before attempting to develop or distribute cryptographic code.

## License

I (elsehow) release copyright to
Copyright 2015-2016 Open Whisper Systems 
under the GPLv3: http://www.gnu.org/licenses/gpl-3.0.html
