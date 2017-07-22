# <img src="https://raw.githubusercontent.com/kaepora/miniLock/master/src/img/icon128.png" width="48" alt="" /> miniLock  
## File encryption software that does more with less.

[<img src="https://travis-ci.org/kaepora/miniLock.svg?branch=master" width="80">](https://travis-ci.org/kaepora/miniLock)

* **[Code](https://github.com/kaepora/miniLock)** | **[Issues and Discussion](https://github.com/kaepora/miniLock/issues)**
* **HOPE X [Video](https://vimeo.com/101237413) | [Slides](https://minilock.io/files/HOPEX.pdf)**
* **Follow on [Twitter](https://twitter.com/minilockapp)** for latest project news

### Download
[<img src="https://minilock.io/img/chromeStore.png" width="250">](https://chrome.google.com/webstore/detail/minilock/mknafpnfcafklkflhenhaldedggpciao)

miniLock is currently audited, peer-reviewed software. An initial public release is available for Google Chrome and Chrome OS.  


### Software Status
miniLock was subjected to a cryptographic code audit carried out by [Cure53](https://cure53.de/) and with the support of the [Open Technology Fund](https://www.opentechfund.org/). Quoting from the conclusion of the audit report (**[PDF](https://minilock.io/files/cure53.pdf)**):

> Cure53 was tasked to test against the application security of miniLock and evaluate its cryptographic properties and promises. Over the course of four days of manual testing, no severe errors have been spotted. The code is soundly and neatly written, well structured, minimal and therefore offers no sinks for direct exploitation.

miniLock also ships with a Unit Test Kit located in `test`.

### 0. Overview
miniLock is a small, portable file encryption software. The idea behind its design is that passphrase memorized by the user, along with their email address, can act as a complete, portable basis for a persistent public key identity and provide a full substitute for other key pair models, such as having the key pair stored on disk media (the PGP approach).  

Advancements in elliptic curve cryptography, specifically in systems such as `curve25519`, allow us to generate key pairs where the lengths of both public and private keys are relatively very small. This means that public keys become far easier to share (miniLock public keys, called *miniLock IDs*, fit inside less than half a tweet). This also means that a human-memorizable passphrase of adequate entropy can be used as the basis for deriving a private key.

When first opened, miniLock asks the user for their email address and a passphrase which it then uses to derive the user's private and public keys. Via this model, the user can establish their key pair on any computer that has miniLock installed using only this passphrase, without having to manage key files or identities and so on. Thanks to the small key sizes present in `curve25519`, we are guaranteed small, easily tweetable public keys and private keys that can be derived from passphrases. miniLock also contains checks to ensure the passphrases entered by the user are of sufficient entropy. miniLock will refuse weak passphrases completely and instead suggest stronger passphrases for use by the user.

miniLock then allows the user to encrypt files to other miniLock users via their miniLock IDs and decrypt files sent to them. miniLock's encryption format supports encrypting a single file to multiple recipients with a negligible increase in file size. Another feature is that analyzing a miniLock-encrypted file does not yield the miniLock IDs or identities of the sender or the recipient(s). Upon decryption, a legitimate recipient will be able to know and verify the identity of the sender, but will still be unable to determine the identity of other potential recipients.

miniLock file encryption provides both confidentiality and integrity. miniLock uses the [TweetNaCL](http://tweetnacl.cr.yp.to) cryptography library, [ported to JavaScript](https://github.com/dchest/tweetnacl-js), entirely due to its focus on simplicity, auditability and small size. Similarly, miniLock is designed to be as simple, portable, auditable and usable as possible. miniLock also uses [scrypt](http://www.tarsnap.com/scrypt.html) for "memory-hard" key derivation.

### 1. User Flow
This section outlines an example user flow in order to help demonstrate how miniLock is supposed to help people.

Alice wants to send a scan of her passport to Bob. Sending it over email would compromise personal information, so Alice decided to first encrypt the scan using miniLock.

Bob opens miniLock and enters his email address and passphrase. miniLock displays his miniLock ID, which is tied to his passphrase and is persistent. He sends Alice his miniLock ID, which looks something like this:
`quBSaJLXKsRiaSrhgkPnswKocth711H29ZamMi1H9j4Mb`

Alice drags and drops her passport scan into miniLock and enters Bob's miniLock ID as the recipient. She clicks the encrypt button and sends the resulting `.minilock` file to Bob. Once Bob drags the encrypted file into miniLock, it automatically detects it as a miniLock-encrypted file destined to Bob, and decrypts and saves the passport scan on his computer.

### 2. Key Derivation
miniLock uses the [zxcvbn](https://github.com/dropbox/zxcvbn) library in order to impose a strict limit on the amount of detected entropy present in entered passphrases. miniLock will not allow passphrases that fall below the threshold of 100 bits of entropy: if a passphrase of lower entropy is detected, miniLock will refuse it and will not allow access to encryption or decryption functions.

Users are encouraged to use passphrases which are easier to remember but harder to guess. If a user fails to enter a sufficiently entropic passphrase, miniLock will use a built-in dictionary of the 58,110 most common words in the English language to suggest a seven-word passphrase. This gives us a passphrase with approximately 111 bits of entropy, since 58110<sup>7</sup> ~= 2<sup>111</sup>.

Once we obtain a suitable passphrase, we hash it using `BLAKE2s` and pass the resulting 32 bytes through `scrypt` in order to obtain the user's private `curve25519` key. `scrypt` is invoked using the following parameters::

* N = 2<sup>17</sup>
* r = 8,
* p = 1,
* L = 32

miniLock uses the email address entered by the user as the `scrypt` key derivation salt. Email addresses are unique by nature and therefore provide a good basis for a salt.

Once we obtain our 32-byte private key, the public key is derived for use with the TweetNaCL `curve25519-xsalsa20-poly1305` construction.

The user's `miniLock ID` consists of 33 bytes. The first 32 bytes are the user's `curve25519` public key. The last byte acts as a checksum: it is derived by hashing the first 32 bytes with `BLAKE2s` set to a 1-byte output. After constructing the 33 bytes of the miniLock ID, it is encoded into a Base58 representation, meant to be easily communicable via email or instant messaging.


### 3. File format
miniLock saves encrypted files as binary blobs with the following format:

```javascript
miniLock magic bytes (8 bytes)
Header length in bytes (4 bytes, little-endian)
Header bytes
Ciphertext bytes
```

miniLock magic bytes identify that this is a miniLock-encrypted file:

```javascript
0x6d, 0x69, 0x6e, 0x69,
0x4c, 0x6f, 0x63, 0x6b
```

The header itself is a stringified JSON object which contains information necessary for the recipients to decrypt the file. The JSON object has the following format:

```json
{
version: Version of the miniLock protocol used for this file (Currently 1) (Number)
ephemeral: Public key from ephemeral key pair used to encrypt decryptInfo object (Base64),
decryptInfo: {
	(One copy of the below object for every recipient)
	Unique nonce for decrypting this object (Base64): {
		senderID: Sender's miniLock ID (Base58),
		recipientID: miniLock ID of this recipient (used for verfication) (Base58),
		fileInfo: {
			fileKey: Key for file decryption (Base64),
			fileNonce: Nonce for file decryption (Base64),
			fileHash: BLAKE2s hash (32 bytes) of the ciphertext bytes. (Base64)
		} (fileInfo is encrypted to recipient's public key using long-term key pair) (Base64),
	} (encrypted to recipient's public key using ephemeral key pair) (Base64)
}
```

Note that the nonce used to encrypt `decryptInfo` is the same as the one used to encrypt `fileInfo`. Nonce reuse in this scenario is permitted since we are encrypting using different keys.

### 4. File encryption
The sender begins by generating a new ephemeral `curve25519` key pair, `senderEphemeralSecret` and `senderEphemeralPublic`.

The sender's long-term keys are denoted as `senderSecret` and `senderPublic`.

A recipient `a`'s long-term keys are denoted as `recipientSecret[a]` and `recipientPublic[a]`.

The sender appends the bytes signalling the beginning of the header to the final encrypted file.

The file's filename is padded with 0x00 bytes until its length equals 256 bytes. The filename is then prepended to the plaintext prior to encryption. The filename is encrypted as its own 256-byte chunk (see chunking format below).

A random 32-byte `fileKey` and a random 16-byte `fileNonce` are generated and used to symmetrically encrypt the plaintext bytes using TweetNaCL's `xsalsa20-poly1305` construction. We encrypt the plaintext bytes by splitting the plaintext into 1048576-byte chunks. Each chunk is then encrypted using the following model:

```javascript
fullNonce0 = fileNonce || 0
encryptedChunk0 = length(chunk0) || nacl.secretbox(chunk0, fullNonce0, fileKey)
fullNonce1 = fileNonce || 1
encryptedChunk1 = length(chunk1) || nacl.secretbox(chunk1, fullNonce1, fileKey)
...
```

In the above example, the 24-byte `fullNonce` is acquired by concatenating the 16-byte `fileNonce` and 8-byte little-endian chunk number. Also,`length(chunk)` is a 4-byte little-endian plaintext chunk length.

The last chunk is encrypted as follows:
```javascript
fullNonceN = fileNonce || setMostSignificantBit(N)
encryptedChunkN = length(chunkN) || nacl.secretbox(chunkN, fullNonceN, fileKey)
```

The sender generates a `fileInfo` JSON object containing `fileKey`, `fileNonce` and `fileHash`. For every recipient `n`, the sender encrypts `fileInfo` using `senderSecret` and `recipientPublic[n]` and stores it within a `decryptInfo` object inside the JSON header along with `senderID`, as described in §3.

The name of the `decryptInfo` property in which the aforementioned elements are stored is a 24-byte nonce. The sender uses this nonce, along with `senderEphemeralSecret`, to encrypt the underlying JSON object asymmetrically to `recipientPublic[n]`, using TweetNaCL's `curve25519-xsalsa20-poly1305` construction. Note that this is done once for every recipient, creating a different `decryptInfo` object for every recipient, each labeled by their unique nonces.

Finally, the sender appends the bytes signalling the end of the header, followed by the ciphertext bytes.

TweetNaCL's `curve25519-xsalsa20-poly1305` construction provides authenticated encryption, guaranteeing both confidentiality and ciphertext integrity. The above header construction makes it impossible to determine the sender or recipient(s) of a miniLock-encrypted file simply by analyzing the ciphertext.

### 5. File decryption
In order to decrypt the file, the recipient needs the information stored within the `decryptInfo` section of the header. They also will need the `ephemeral` property of the header in order to derive the shared secret, in conjunction with their long-term secret key, which can be used to decrypt their copy of the `decryptInfo` header object.

If there are multiple properties within `decryptInfo`, the recipient must iterate through every property until she obtains an authenticated decryption of the underlying object. Once a successful authenticated decryption of a `decryptInfo` property occurs, the recipient can then use the obtained `senderID` along with their long-term secret key to decrypt `fileKey` and use it in conjunction with `fileNonce` to perform an authenticated decryption of the ciphertext bytes.

Before or during file decryption, the recipient compares the 32-byte `BLAKE2s` hash of the ciphertext against the decrypted `fileHash`. The recipient also compares the decrypted `recipientID` against their own miniLock ID. If any of these checks fail, decryption is aborted and an error is returned.

In order to decrypt the ciphertext bytes, the recipient breaks the ciphertext down to chunks consisting of the original 1048576 bytes of the plaintext chunk, plus the 4 bytes defining the chunk length and the 16 bytes defining the `poly1305` authentication code of that particular ciphertext chunk. Each chunk is then decrypted sequentially using the following model:

```javascript
fullNonce0 = fileNonce || 0
decryptedChunk0 = nacl.secretbox.open(chunk0, fullNonce0, fileKey)
fullNonce1 = fileNonce || 1
decryptedChunk1 = nacl.secretbox.open(chunk1, fullNonce1, fileKey)
...
```

After decryption, the recipient retrieves the filename from the first 256 bytes of the plaintext. The recipient strips the padding bytes from the filename and is now capable of saving the decrypted file.

If the authenticated asymmetric decryption of any header object fails, or the authenticated symmetric decryption of the file ciphertext fails, decryption is aborted and an error is returned.

### 6. Key Identity Authentication
In PGP, public keys can be substantially larger than miniLock IDs, therefore necessitating the generation of key fingerprints which can then be used for out-of-band key identity authentication. With miniLock, users are able to authenticate out-of-band directly using the miniLock ID, due to its small length (approximately 45 Base58-encoded characters). Therefore, no specialized key identity authentication mechanism is required.

### 7. Error Codes
miniLock will output these error codes when running into encryption or decryption errors. The user interface can then handle these errors in order to display information that is relevant to users:

**Encryption errors**
* `Error 1`: General encryption error

**Decryption errors**
* `Error 2`: General decryption error
* `Error 3`: Could not parse header
* `Error 4`: Invalid header version
* `Error 5`: Could not validate sender ID
* `Error 6`: File is not encrypted for this recipient
* `Error 7`: Could not validate ciphertext hash

### 8. Caveats
miniLock is not intended to protect against malicious files being sent and received. It is the user's responsibility to vet the safety of the files they send or receive over miniLock. miniLock cannot protect against malware being sent over it.

### 9. Thanks
Sincere thanks are presented to Dr. Matthew D. Green and Meredith L. Patterson, who gave feedback on an early draft of this document.

Sincere thanks are presented to Trevor Perrin for his invaluable contribution to miniLock's design, which introduced sender ID anonymity in the ciphertext.

Sincere thanks are presented to Dmitry Chestnykh for his work on porting TweetNaCL to JavaScript and his general cooperation with the miniLock project, including many helpful and crucial suggestions.

Sincere thanks are presented to Dr. Mario Heiderich and his team at [Cure53](https://cure53.de/) for their work on performing a full audit of the miniLock codebase. We also sincerely thank the [Open Technology Fund](https://www.opentechfund.org/) for funding the audit.

Finally, sincere thanks are presented to the wonderful, constructive members of the miniLock community who have contributed many improvements and ideas to the miniLock design and codebase. You rock!

### 10. Credits
#### miniLock
Copyright 2014 [Nadim Kobeissi](http://nadim.computer). Released under the AGPLv3 license.

#### TweetNaCL
[Daniel J. Bernstein](http://cr.yp.to/djb.html), Wesley Janssen, [Tanja Lange](http://hyperelliptic.org/tanja), [Peter Schwabe](http://www.cryptojedi.org/users/peter/), [Matthew Dempsky](https://github.com/mdempsky)

#### TweetNaCL-JS
[Dmitry Chestnykh](http://github.com/dchest), [Devi Mandiri](https://github.com/devi)

#### scrypt
[Colin Percival](http://www.tarsnap.com/scrypt.html)
