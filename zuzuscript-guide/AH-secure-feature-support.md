# Appendix H: `std/secure` Feature Support

This appendix summarizes the current `std/secure` implementation status
by host.

Legend:

- `Yes`: supported and passing the current secure tests.
- `Async`: supported, but browser code must use the `_async` method.
- `Probe`: supported only when the browser Web Crypto implementation
  exposes the required primitive.
- `No`: intentionally unsupported in the current phase.
- `Bug`: intended or advertised support currently has a minor test or
  implementation failure.

The table distinguishes intentional gaps from current bugs. Intentional
gaps should be handled with `Secure.has` or `Secure.require`. `Bug`
entries are not API policy; they should become `Yes` when the current
implementation issue is fixed.

The status below was checked against the secure ztests on 2026-05-06.
The secure-only Perl, Rust, Node, and Electron matrix currently has two
Electron failures: `scrypt` capability expectations disagree with the
runtime report, and ChaCha20-Poly1305 is advertised but not usable.
The browser secure matrix passes.

## H.1 Core And Random Features

| Feature | Perl | Rust | JS/Node | JS/Electron | JS/Browser |
| --- | --- | --- | --- | --- | --- |
| `Secure.capabilities`, `Secure.has`, `Secure.require` | Yes | Yes | Yes | Bug | Yes |
| `SecureRandom.bytes` | Yes | Yes | Yes | Yes | Yes |
| `SecureRandom.token` | Yes | Yes | Yes | Yes | Yes |
| `SecureRandom.int` | Yes | Yes | Yes | Yes | Yes |
| `async_required` reporting | Yes | Yes | Yes | Yes | Yes |

Electron's capability API works in general, but the current secure matrix
has failures around `scrypt` and ChaCha20-Poly1305 reporting.

## H.2 Password Hashing

| Feature | Perl | Rust | JS/Node | JS/Electron | JS/Browser |
| --- | --- | --- | --- | --- | --- |
| `pbkdf2-sha256` hash, verify, and derive | Yes | Yes | Yes | Yes | Async |
| `pbkdf2-sha256` async methods | Yes | Yes | Yes | Yes | Yes |
| `argon2id` hash, verify, and derive | Yes | Yes | No | No | No |
| `scrypt` hash, verify, and derive | Yes | Yes | Yes | Bug | No |
| `crypt` verify and migration support | Yes | No | No | No | No |
| `PasswordHash.needs_rehash` | Yes | Yes | Yes | Yes | Yes |

`argon2id`, `scrypt`, and `crypt` gaps outside the listed hosts are
intentional. Browser builds intentionally avoid password-hashing WASM
packages in this phase and support PBKDF2 through Web Crypto.

## H.3 Key Derivation And Symmetric Ciphers

| Feature | Perl | Rust | JS/Node | JS/Electron | JS/Browser |
| --- | --- | --- | --- | --- | --- |
| `hkdf-sha256` | Yes | Yes | Yes | Yes | Yes |
| `hkdf-sha256_async` | Yes | Yes | Yes | Yes | Yes |
| `aes-256-gcm` key generation | Yes | Yes | Yes | Yes | Yes |
| `aes-256-gcm` encrypt/decrypt | Yes | Yes | Yes | Yes | Async |
| `aes-128-gcm` | Yes | No | No | No | No |
| `aes-192-gcm` | Yes | No | No | No | No |
| `chacha20-poly1305` | Yes | Yes | Yes | Bug | No |

AES-128-GCM and AES-192-GCM are currently Perl-only additions. Browser
builds intentionally expose only AES-256-GCM, and only through async
encrypt/decrypt methods.

## H.4 Signing And Key Agreement

| Feature | Perl | Rust | JS/Node | JS/Electron | JS/Browser |
| --- | --- | --- | --- | --- | --- |
| Ed25519 signing | Yes | Yes | Yes | Yes | No |
| ECDSA P-256 with SHA-256 | Yes | Yes | Yes | Yes | Async |
| ECDSA P-384 with SHA-384 | Yes | Yes | Yes | Yes | Async |
| ECDSA P-521 with SHA-512 | Yes | No | No | No | No |
| Raw signing-key import/export | Yes | Yes | Yes | Yes | Yes |
| PEM signing-key import/export | Yes | Yes | Yes | Yes | Yes |
| X25519 key agreement | Yes | Yes | Yes | Yes | Probe/Async |
| X25519 raw key import/export | Yes | Yes | Yes | Yes | Probe/Async |

Browser Ed25519 is intentionally unsupported even if a specific Web
Crypto implementation exposes it. Browser X25519 is capability-probed
because Web Crypto support is still not uniformly available.

## H.5 X.509 Certificates

| Feature | Perl | Rust | JS/Node | JS/Electron | JS/Browser |
| --- | --- | --- | --- | --- | --- |
| Parse DER certificate | Yes | Yes | Yes | Yes | Yes |
| Parse PEM certificate | Yes | Yes | Yes | Yes | No |
| Parse DER chain | Yes | Yes | Yes | Yes | Yes |
| Parse PEM chain | Yes | Yes | Yes | Yes | No |
| Subject and issuer strings | Yes | Yes | Yes | Yes | Yes |
| Serial number normalization | Yes | Yes | Yes | Yes | Yes |
| Validity as `std/time` `Time` objects | Yes | Yes | Yes | Yes | Yes |
| SHA-256 fingerprint | Yes | Yes | Yes | Yes | Yes |
| SHA-384 fingerprint | Yes | No | No | No | No |
| SHA-512 fingerprint | Yes | No | No | No | No |
| `to_der` | Yes | Yes | Yes | Yes | Yes |
| `to_pem` | Yes | Yes | Yes | Yes | Yes |
| Extract supported certificate public key | Yes | Yes | Yes | Yes | No |
| Verify certificate chain | Yes | Yes | Yes | Yes | No |
| Use system trust roots for chain verification | Yes | Yes | Yes | Yes | No |

Browser certificate support is intentionally DER-only and uses the
in-repository parser. PEM parsing, public-key extraction, and chain
validation are intentionally absent from browser builds in this phase.

## H.6 TLS Identities

| Feature | Perl | Rust | JS/Node | JS/Electron | JS/Browser |
| --- | --- | --- | --- | --- | --- |
| Parse PEM TLS identity | Yes | Yes | Yes | Yes | Yes |
| Inspect PEM identity certificate | Yes | Yes | Yes | Yes | Yes |
| Extract identity private key | Yes | Yes | Yes | Yes | No |
| Parse PKCS#12 TLS identity | Yes | Yes | Yes | Yes | No |
| Script-selected HTTP TLS client identity | No | No | No | No | No |

Browser PEM TLS identities are intentionally inert: code can inspect the
certificate, but the private key is not exposed and browser network
requests cannot be given script-selected client certificates in this
phase. Script-selected HTTP TLS client identity is still future work for
all hosts.

## H.7 Practical Portability Notes

For code intended to run everywhere:

- use `pbkdf2-sha256` unless a stronger host-specific password hash is
  explicitly required,
- use `aes-256-gcm`,
- use `hkdf-sha256` for high-entropy key derivation,
- use ECDSA P-256 or P-384 if browser signing is required,
- use Ed25519 only for CLI-style hosts,
- use DER certificate parsing for browser-compatible certificate
  inspection, and
- call async methods in browser code for password hashing, ciphers,
  signing, and X25519.

For code that targets only CLI-style hosts, check capabilities and choose
stronger or more convenient host features where they are available.
