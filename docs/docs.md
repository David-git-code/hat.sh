# [Introduction](#introduction)

---

Hat.sh is a free [opensource] web app that provides secure file encryption in the browser.

<br>

# [Features](#features)

---

### Security

- [XChaCha20-Poly1305] - for symmetric encryption.
- [Argon2id] - for password-based key derivation.

The libsodium library is used for all cryptographic algorithms. [Technical details here](#technical-details).

<br>

### Privacy

- The app runs locally in your browser.
- No data is ever collected or sent to anyone.​

<br>

# [Installation](#installation)

---
If you wish to self host hat.sh please follow these instructions: 

<br>

Download or clone the repository

```bash
git clone --branch v2-beta https://github.com/sh-dv/hat.sh.git hat.sh-v2-beta
```

<br>

Go to the app directory

```bash
cd hat.sh-v2-beta or [app directory]
```

<br>

Open terminal and install the packages

```bash
npm install
```

<br>

Run the app in dev mode

```bash
npm run dev
```

<br>

The app should be running in dev enviroment on `localhost:3000`

<br>

If you plan on running the app in production mode :

```bash
npm run build && npm run serve
```

<br>

# [Usage](#usage)

---

### File Encryption

1. Open hat.sh
2. Navigate to the Encryption panel
3. Drag & Drop or Select the file that you wish to encrypt
4. Enter the encryption password
5. Download the encrypted file

> You should always use a strong password!

### File Decryption

1. Open hat.sh
2. Navigate to the Decryption panel
3. Drag & Drop or Select the file that you wish to decrypt
4. Enter the decryption password
5. Download the decrypted file

<br>

# [Limitations](#limitations)

---

### Folder & Multiple Files Encryption

This feature is not available for security reasons. If you wish to encrypt a whole directory or multiple files then you should make a Zip and encrypt it.

### File Metadata

Files encrypted with the app are identifiable by looking at the file signature that is used by the app to verify the content of a file, Such signatures are also known as magic numbers or Magic Bytes. These Bytes are authenticated and cannot be changed.

### Safari and Mobile Browsers

Safari and Mobile browsers are limited to a file size of 1GB due to some issues related to service-workers. In addition, this limitation also applies when the app fails to register the service-worker (e.g FireFox Private Browsing).

<br>

# [Best Practices](#best-practices)

---

### Choosing Passwords

The majority of individuals struggle to create and remember passwords, resulting in weak passwords and password reuse. Password-based encryption is substantially less safe as a result of these improper practices. That's why it is recommended to use the built in password generator and use a password manager like [Bitwarden], where you are able to store the safe password.

<br>

If you want to choose a password that you are able to memorize then you should type a passphrase made of 8 words or more.

### Sharing Encrypted Files

If you plan on sending the file you have encrypted, you can do that in any safe file sharing app.

### Sharing Decryption Passwords

Sharing decryption password can be done using a safe end-to-end encrypted messaging app. It's recommended to use a _Disappearing Messages_ feature, and to delete the password after the recepient has decrypted the file.

> Never choose the same password for different files.

<br>

# [Technical Details](#technical-details)

---

### Password hashing and Key derivation

Password hashing functions derive a secret key of any size from a password and a salt.


```javascript
let salt = sodium.randombytes_buf(sodium.crypto_pwhash_SALTBYTES);
let key = sodium.crypto_pwhash(
  sodium.crypto_secretstream_xchacha20poly1305_KEYBYTES,
  password,
  salt,
  sodium.crypto_pwhash_OPSLIMIT_INTERACTIVE,
  sodium.crypto_pwhash_MEMLIMIT_INTERACTIVE,
  sodium.crypto_pwhash_ALG_ARGON2ID13
);
```

The `crypto_pwhash()` function derives an 256 bits long key from a password and a salt salt whose fixed length is 128 bits, which should be unpredictable.

`randombytes_buf()` is the easiest way to fill the 128 bits of the salt.

<br>

`OPSLIMIT` represents a maximum amount of computations to perform.

`MEMLIMIT` is the maximum amount of RAM that the function will use, in bytes.

<br>

`crypto_pwhash_OPSLIMIT_INTERACTIVE` and `crypto_pwhash_MEMLIMIT_INTERACTIVE` provide base line for these two parameters. This currently requires 64 MiB of dedicated RAM. which is suitable for in-browser operations.
<br>
`crypto_pwhash_ALG_ARGON2ID13` using the Argon2id algorithm version 1.3.

<br>

### File Encryption (stream)

In order to use the app to encrypt a file, the user has to provide a valid file and a password. this password gets hashed and a secure key is derived from it with Argon2id to encrypt the file.

```javascript
let res = sodium.crypto_secretstream_xchacha20poly1305_init_push(key);
header = res.header;
state = res.state;

let tag = last
  ? sodium.crypto_secretstream_xchacha20poly1305_TAG_FINAL
  : sodium.crypto_secretstream_xchacha20poly1305_TAG_MESSAGE;

let encryptedChunk = sodium.crypto_secretstream_xchacha20poly1305_push(
  state,
  new Uint8Array(chunk),
  null,
  tag
);

stream.enqueue(signature, salt, header, encryptedChunk);
```

The `crypto_secretstream_xchacha20poly1305_init_push` function creates an encrypted stream where it initializes a `state` using the key and an internal, automatically generated initialization vector. It then stores the stream header into `header` that has a size of 192 bits.



This is the first function to call in order to create an encrypted stream. The key will not be required any more for subsequent operations.

<br>

An encrypted stream starts with a short header, whose size is 192 bits. That header must be sent/stored before the sequence of encrypted messages, as it is required to decrypt the stream. The header content doesn't have to be secret because decryption with a different header would fail.



A tag is attached to each message accoring to the value of `last`, which indicates if that is the last chunk of the file or not. That tag can be any of:


1. `crypto_secretstream_xchacha20poly1305_TAG_MESSAGE`: This doesn't add any information about the nature of the message.
2. `crypto_secretstream_xchacha20poly1305_TAG_FINAL`: This indicates that the message marks the end of the stream, and erases the secret key used to encrypt the previous sequence.


The `crypto_secretstream_xchacha20poly1305_push()` function encrypts the file `chunk` using the `state` and the `tag`, without any additional information (`null`).
<br>

the XChaCha20 stream cipher Poly1305 MAC authentication are used for encryption.


`stream.enqueue()` function adds the hat.sh signature(magic bytes), salt and header followed by the encrypted chunks.

### File Decryption (stream)

```javascript
let state = sodium.crypto_secretstream_xchacha20poly1305_init_pull(header, key);

let result = sodium.crypto_secretstream_xchacha20poly1305_pull(
  state,
  new Uint8Array(chunk)
);

if (result) {
  let decryptedChunk = result.message;
  stream.enqueue(decryptedChunk);

  if (!last) {
    // continue decryption
  }
}
```

The `crypto_secretstream_xchacha20poly1305_init_pull()` function initializes a state given a secret `key` and a `header`. The key is derived from the password provided during the decryption, and the header sliced from the file. The key will not be required any more for subsequent operations.

<br>

The `crypto_secretstream_xchacha20poly1305_pull()` function verifies that the `chunk` contains a valid ciphertext and authentication tag for the given `state`.

This function will stay in a loop, until a message with the `crypto_secretstream_xchacha20poly1305_TAG_FINAL` tag is found.

If the decryption key is incorrect the function returns an error.

If the ciphertext or the authentication tag appear to be invalid it returns an error.

### XChaCha20-Poly1305

XChaCha20 is a variant of ChaCha20 with an extended nonce, allowing random nonces to be safe.

XChaCha20 doesn't require any lookup tables and avoids the possibility of timing attacks.

Internally, XChaCha20 works like a block cipher used in counter mode. It uses the HChaCha20 hash function to derive a subkey and a subnonce from the original key and extended nonce, and a dedicated 64-bit block counter to avoid incrementing the nonce after each block.

<br>

XChaCha20 is generally recommended over plain ChaCha20 due to its extended nonce size, and its comparable performance.

Authentication used is Poly1305, a Wegman-Carter authenticator designed by D. J. Bernstein.

Poly1305 takes a 256 bit, one-time key and a message and produces a 128 bit tag that authenticates the message such that an attacker has a negligible chance of producing a valid tag for a inauthentic message.

Poly1305 keys have to be secret, unpredictable and unique.

<br>

The XChaCha20-Poly1305 construction can safely encrypt a practically unlimited number of messages with the same key, without any practical limit to the size of a message. As an alternative to counters, its large nonce size (192-bit) allows random nonces to be safely used.

XChaCha20-Poly1305 applies the construction described in Daniel Bernstein's [Extending the Salsa20 nonce paper] to the ChaCha20 cipher in order to extend the nonce size to 192-bit.

This extended nonce size allows random nonces to be safely used, and also facilitates the construction of misuse-resistant schemes.

<br>

The XChaCha20-Poly1305 implementation in libsodium is portable across all supported architectures and It will [soon] become an IETF standard.

- Encryption: XChaCha20 stream cipher
- Authentication: Poly1305 MAC

### V2 vs V1

- switching to xchacha20poly1305 for symmetric stream encryption and Argon2id for password-based key derivation. instead of AES-256-GCM and PBKDF2.
- using the libsodium library for all cryptography instead of the WebCryptoApi.
- in this version, the app doesn't read the whole file in memory. instead, it's sliced into 64MB chunks that are processed one by one.
- since we are not using any server-side processing, the app registers a fake download URL (/file) that is going to be handled by the service-worker fetch api.
- if all validations are passed, a new stream is initialized. then, file chunks are transferred from the main app to the
  service-worker file via messages.
- each chunk is encrypted/decrypted on it's own and added to the stream.
- after each chunk is written on disk it is going to be immediately garbage collected by the browser, this leads to never having more than a few chunks in the memory at the same time.

# [FAQ](#faq)

---

### Does the app log or store any of my data?

No, hat.sh never stores any of your data. It only runs locally in your browser.

<hr style="height: 1px">

### Is hat.sh free?

Yes, Hat.sh is free and always will be. However, please consider donating to support the project.

<hr style="height: 1px">

### Which file types are supported? Is there a file size limit?

Hat.sh accepts all file types. There's no file size limit, meaning files of any size can be encrypted.

Safari browser and mobile/smartphones browsers are limited to 1GB.

<hr style="height: 1px">

### I forgot my password, can I still decrypt my files?

No, we don't know your password. Always make sure to store your passwords in a password manager.

<hr style="height: 1px">

### Does the app connect to the internet?

Once you visit the site and the page loads, it runs only offline.

<hr style="height: 1px">

### How can I contribute?

Hat.sh is an open-source application. You can help make it better by making commits on GitHub. The project is maintained in my free time. Donations of any size are appreciated.

<hr style="height: 1px">

### How do I report bugs?

Please report bugs via [Github] by opening an issue labeled with "bug".

<hr style="height: 1px">

### How do I report a security vulnerability?

If you identify a valid security issue, please write an email to hatsh-security@pm.me

There is no bounty available at the moment, but your github account will be credited in the acknowledgements section in the app documentation.

<hr style="height: 1px">

### Why should I use hat.sh?

1. The app uses fast modern secure cryptographic algorithms.
2. It's super fast and easy to use.
3. It runs in the browser, no need to setup or install anything.
4. It's free opensource software and can be self hosted.

<hr style="height: 1px">

### When should I not use hat.sh?

1. If you want to encrypt a disk (e.g [VeraCrypt]).
2. If you want to Frequently access encrypted files (e.g [Cryptomator]).
3. If you want to encrypt multiple files and directories at once (e.g [Kryptor]).
4. If you want to encrypt files for another person that only they can decrypt (e.g [Kryptor]).
5. If you want something that adheres to industry standards, use [GPG].

<br>

[//]: # "links"
[xchacha20-poly1305]: https://libsodium.gitbook.io/doc/secret-key_cryptography/aead/chacha20-poly1305/xchacha20-poly1305_construction
[argon2id]: https://github.com/p-h-c/phc-winner-argon2
[opensource]: https://github.com/sh-dv/hat.sh
[bitwarden]: https://bitwarden.com/
[extending the salsa20 nonce paper]: https://cr.yp.to/snuffle/xsalsa-20081128.pdf
[soon]: https://tools.ietf.org/html/draft-irtf-cfrg-xchacha
[github]: https://github.com/sh-dv/hat.sh
[veracrypt]: https://veracrypt.fr
[cryptomator]: https://cryptomator.org
[kryptor]: https://github.com/samuel-lucas6/Kryptor
[gpg]: https://gnupg.org
