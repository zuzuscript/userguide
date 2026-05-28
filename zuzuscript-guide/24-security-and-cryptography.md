# Chapter 24: Security and Cryptography

By now the guide has touched most of the places a script meets the world:
files, processes, packages, databases, HTTP, web servers, and GUIs. Those
boundaries are exactly where trust and safety start to matter.

Security code is ordinary application code with less room for improvising.
ZuzuScript keeps the public cryptography surface in one runtime-supported
module:

```zzs
from std/secure import
	Secure,
	SecureRandom,
	PasswordHash,
	KeyDerivation,
	Cipher,
	KeyAgreement,
	SigningKey,
	Certificate,
	TlsIdentity;
```

The module is capability-based. A program can ask what the current host
supports, require a feature before it uses it, and choose a portable
fallback where that is appropriate.

Note that a lot of this functionality requires binary strings. In ZuzuScript,
binary strings are quoted in `'single quotes'` and character strings are
quoted in `"double quotes"`.

## 24.1 Check Capabilities First

`Secure.capabilities()` returns a dictionary describing the current host.
The host names are currently `perl`, `rust`, `node`, and `browser`;
Electron reports through the JavaScript implementation and is tracked
separately in Appendix H.

```zzs
from std/secure import Secure;

let caps := Secure.capabilities();

say `secure host: ${caps{host}}`;

if ( Secure.has( "cipher", "aes-256-gcm" ) ) {
	say "AES-256-GCM is available";
}
```

When an algorithm is required for correctness, use `Secure.require`.
It returns `true` on success and throws a clear exception on failure:

```zzs
Secure.require( "password_hash", "pbkdf2-sha256" );
Secure.require( "cipher", "aes-256-gcm" );
```

Use `Secure.has` for optional upgrades and `Secure.require` for
non-negotiable assumptions. Do not discover support by trying random
operations and ignoring errors; that makes security policy hard to read.


## 24.2 Randomness

Use `SecureRandom` for secrets, tokens, nonces that are part of an
application protocol, invitation codes, and any value that must be
unpredictable.

```zzs
from std/secure import SecureRandom;

let reset_token := SecureRandom.token(32);
let api_secret := SecureRandom.bytes(32);
let shard := SecureRandom.int(16);
```

`token(bytes)` returns URL-safe Base64 text with no padding. It is a good
default for reset links and API keys that need to be copied or embedded
in URLs. `bytes(length)` returns raw `BinaryString` data for keys and
binary protocols. `int(max)` returns an unbiased number from `0` through
`max - 1`.


## 24.3 Password Hashing

Store password hashes, not passwords. `PasswordHash.hash` returns a
self-describing string containing the algorithm, salt, work parameters,
and derived hash.

```zzs
from std/secure import PasswordHash;

function create_user ( db, username, password ) {
	let encoded := PasswordHash.hash(password);
	let insert := db.prepare(
		"insert into users (username, password_hash) values (?, ?)"
	);
	insert.execute( username, encoded );
}
```

Verification uses the stored encoding:

```zzs
function password_matches ( password, encoded ) {
	return PasswordHash.verify( password, encoded );
}
```

`PasswordHash.needs_rehash` lets an application upgrade old parameters
after a successful login:

```zzs
if ( PasswordHash.verify( password, user{password_hash} ) ) {
	if ( PasswordHash.needs_rehash( user{password_hash} ) ) {
		user{password_hash} := PasswordHash.hash(password);
	}
}
```

The portable baseline is `pbkdf2-sha256`. Perl and Rust currently also
support `argon2id`; Perl, Rust, and Node support `scrypt`; Perl supports
legacy `crypt`. Use `crypt` only to verify or migrate old data.

Browser code must use the async password-hash methods:

```zzs
let encoded := await {
	PasswordHash.hash_async(password);
};

let ok := await {
	PasswordHash.verify_async( password, encoded );
};
```


## 24.4 Deriving Keys

`KeyDerivation.hkdf_sha256` is for high-entropy key material, such as an
X25519 shared secret. It is not a password hashing function.

```zzs
from std/secure import KeyDerivation;

let encryption_key := KeyDerivation.hkdf_sha256(
	shared_secret,
	32,
	salt,
	'my-protocol v1 encryption',
);
```

If the input is a human password or passphrase, use
`PasswordHash.derive_key` with an explicit salt instead:

```zzs
from std/secure import PasswordHash, SecureRandom;

let salt := SecureRandom.bytes(16);
let key := PasswordHash.derive_key(
	"correct horse battery staple",
	{ algorithm: "pbkdf2-sha256", salt: salt, length: 32 },
);
```

The salt and derivation parameters need to be stored with the encrypted
data. The password itself must not be stored.


## 24.5 Authenticated Encryption

`Cipher` provides authenticated encryption. The portable cipher is
`aes-256-gcm`.

```zzs
from std/secure import Cipher;

let key := Cipher.generate_key();
let aad := 'invoice:v1:customer:123';
let plaintext := 'payment token';

let sealed := Cipher.encrypt(
	plaintext,
	key,
	{ aad: aad },
);

let opened := Cipher.decrypt(
	sealed,
	key,
	{ aad: aad },
);
```

The envelope is a dictionary:

```zzs
{
	version: 1,
	algorithm: "aes-256-gcm",
	nonce: '...',
	ciphertext: '...',
	tag: '...',
}
```

The tag authenticates the ciphertext and associated data. If the key,
nonce, ciphertext, tag, or associated data is wrong, decryption throws.

Associated data is not encrypted. Use it for context that must be bound
to the ciphertext, such as a record type, protocol version, tenant id, or
database row id.

Browser code must use the async encryption and decryption methods:

```zzs
let sealed := await {
	Cipher.encrypt_async( plaintext, key, { aad: aad } );
};

let opened := await {
	Cipher.decrypt_async( sealed, key, { aad: aad } );
};
```


## 24.6 Signing

Use signing when another component needs to verify that a message came
from a holder of a private key and was not changed.

```zzs
from std/secure import SigningKey;

let signing_key := SigningKey.generate("ed25519");
let public_key := signing_key.public_key();
let message := 'release:2026-05-06';

let signature := signing_key.sign(message);

if ( public_key.verify( message, signature ) ) {
	say "signature accepted";
}
```

Perl, Rust, Node, and Electron support Ed25519. ECDSA P-256 with SHA-256
and ECDSA P-384 with SHA-384 are available on every current host, though
browser code must use async methods. Perl also supports ECDSA P-521 with
SHA-512.

```zzs
let key := await {
	SigningKey.generate_async("ecdsa-p256-sha256");
};

let signature := await {
	key.sign_async(message);
};

let ok := await {
	key.public_key().verify_async( message, signature );
};
```

Do not sign text after implicit encoding decisions. Convert it to the
exact bytes you mean to sign, and include protocol context in the signed
payload.


## 24.7 Key Agreement

X25519 key agreement lets two parties derive the same shared secret
without sending that secret over the wire. The raw shared secret should
normally be passed through HKDF before it is used.

```zzs
from std/secure import KeyAgreement, KeyDerivation;

let alice := KeyAgreement.generate("x25519");
let bob := KeyAgreement.generate("x25519");

let shared := alice.derive( bob.public_key() );

let key := KeyDerivation.hkdf_sha256(
	shared,
	32,
	null,
	'example x25519 aes key',
);
```

In browser code, use the async key-agreement methods when the browser
reports `x25519` support:

```zzs
let alice := await {
	KeyAgreement.generate_async("x25519");
};

let shared := await {
	alice.derive_async(peer_public_key);
};
```


## 24.8 Certificates

`Certificate` parses X.509 certificates for inspection and chain
verification. Perl, Rust, Node, and Electron support PEM and DER input.
Browser hosts support DER `BinaryString` input only.

```zzs
from std/secure import Certificate;

let cert := Certificate.parse(pem_text);

say cert.subject();
say cert.issuer();
say cert.serial_number();
say cert.not_before().epoch();
say cert.not_after().epoch();
```

Fingerprints are returned as raw bytes:

```zzs
let fp := cert.fingerprint("sha256");
```

Use `to_der` for byte-for-byte DER output and `to_pem` for canonical PEM
text. On non-browser hosts, `public_key()` extracts supported signing
public keys from certificates.

```zzs
let public_key := cert.public_key();
```

Chain verification is available on Perl, Rust, Node, and Electron:

```zzs
let result := Certificate.verify_chain(
	[ leaf, intermediate ],
	{
		roots: root_cert,
		hostname: "example.org",
		use_system_roots: false,
	},
);

if ( not result{valid} ) {
	say `certificate rejected: ${result{reason}}`;
}
```

The browser runtime intentionally does not perform chain validation in
this phase.


## 24.9 TLS Identities

`TlsIdentity` represents client-certificate identity material. It is
parsed by `std/secure` so programs can inspect the leaf certificate and,
on CLI-style hosts, access supported private keys.

```zzs
from std/secure import TlsIdentity;

let identity := TlsIdentity.from_pem( certificate_pem, private_key_pem );
let cert := identity.certificate();

say cert.subject();
```

Perl, Rust, Node, and Electron also support PKCS#12 input:

```zzs
let identity := TlsIdentity.from_pkcs12( p12_bytes, password );
```

Browser PEM identities are intentionally inert in this phase: the
certificate can be inspected, but the private key is not exposed and
scripts cannot select TLS client certificates for browser network
requests.


## 24.10 Portable Security Code

The safest portable pattern is:

- choose a portable baseline first,
- check capabilities before optional stronger algorithms,
- use async methods if you need your code to run in a browser,
- treat unsupported algorithms as policy decisions, not surprises,
- store algorithm names and parameters next to encrypted or hashed data,
  and
- keep keys and passwords out of logs, exceptions, and serialized debug
  dumps.

Here is a small capability-driven password helper:

```zzs
from std/secure import PasswordHash, Secure;

function preferred_password_algorithm () {
	if ( Secure.has( "password_hash", "argon2id" ) ) {
		return "argon2id";
	}

	return PasswordHash.default_algorithm();
}

async function hash_password ( password ) {
	return await {
		PasswordHash.hash_async(
			password,
			{ algorithm: preferred_password_algorithm() },
		);
	};
}
```


## 24.11 Chapter Recap

`std/secure` gives ZuzuScript one place for cryptographic operations:
randomness, password hashing, key derivation, authenticated encryption,
signing, key agreement, certificate inspection, and TLS identity parsing.

The module is deliberately capability-based because different hosts have
different safe primitives available. Appendix H lists the current support
matrix in detail.

That closes the technical tour. Chapter 25 takes one last step back:
what you have learned, what you can build next, and where to look when
you need a quick answer.
