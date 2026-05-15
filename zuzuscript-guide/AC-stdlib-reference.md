# Appendix C: Standard Library Reference

This document lists every module in the ZuzuScript standard library.

## javascript
Bridge to host JavaScript evaluation. This module is implemented by
`zuzu-js`; other runtimes should treat it as implementation-specific.

**Exports:** `JS`, `JSResult`.

## perl
Bridge to the host Perl interpreter, allowing controlled evaluation of Perl code and wrapping returned Perl values. It is primarily used for interop and data exchange from ZuzuScript. You should not expect this to work in alternative implementations of ZuzuScript, so it should be used sparingly.

**Exports:** `Perl`, `PerlResult`.

## std/archive
Provides a small archive/compression API for reading and writing common container and compressed formats. It focuses on a portable archive dict shape containing file entries as binary payloads.

**Exports:** `Archive`.

## std/cache/lru
Provides a small in-memory least-recently-used cache abstraction with bounded capacity. It is useful for memoization and for caching expensive computed values.

**Exports:** `Cache`.

## std/clib
Provides dynamic C library loading and scalar FFI calls through explicit
function signatures. Support is runtime-dependent and intended for
carefully controlled native interop.

**Exports:** `CLib`.

## std/colour
Provides colour parsing helpers. It normalizes three- and six-digit
hexadecimal colours and CSS3 extended colour keywords to lowercase
`#rrggbb` strings.

**Exports:** `parse_colour`.

## std/config
Provides a high-level configuration framework for loading, merging, querying, and saving app config data. It supports format auto-detection, deep dictionary overlays, flat env-style key overlays, and direct path queries using `@`, `@@`, and `@?`.

**Exports:** `Config`.

## std/data/cbor
Implements CBOR (Concise Binary Object Representation) encoding/decoding utilities in the standard library. It also includes a tagged-value wrapper for CBOR tags.

**Exports:** `CBOR`, `TaggedValue`.

## std/data/csv
Provides CSV-style parsing and writing with configurable delimiters, quoting, and streaming file iteration. It also includes convenience helpers for importing/exporting database tables through `std/db`.

**Exports:** `CSV`, `CSVReader`.

## std/data/ini
Implements INI text parsing and serialization with configurable formatting options. It is intended for config-file style key/value and section data.

**Exports:** `INI`.

## std/data/json
Provides JSON parsing and serialization with optional pretty/canonical/UTF-8 behaviours. It is the standard JSON codec for structured Zuzu values.

**Exports:** `JSON`.

## std/data/kdl
Provides a pure-Zuzu KDL document model, parser, and serializer. It
returns explicit document, node, and value objects, and can load from or
dump to `std/io` paths when filesystem access is allowed.

**Exports:** `KDL`, `KDLDocument`, `KDLNode`, `KDLValue`.

## std/data/kdl/json
Provides JSON-in-KDL conversion helpers. It converts parsed KDL documents
or nodes to native Zuzu data structures, and converts JSON-like Zuzu data
back to KDL documents.

**Exports:** `kdl_to_json`, `json_to_kdl`.

## std/data/kdl/xml
Provides XML-in-KDL conversion helpers. It converts KDL documents or nodes
to XML documents, and converts XML documents or nodes back to KDL
documents.

**Exports:** `kdl_to_xml`, `xml_to_kdl`.

## std/data/toml
Provides TOML parsing and serialization for dictionary-like configuration data. It includes helpers for loading from and dumping to paths.

**Exports:** `TOML`.

## std/data/toon
Implements the TOON data format parser/encoder used by ZuzuScript tests and tooling. It supports options for strictness, indentation, and key/path expansion behaviour.

**Exports:** `TOON`.

## std/data/xml
Provides XML parsing, loading/dumping helpers, and object wrappers for documents and nodes. It supports traversal, querying, and mutation of XML trees.

**Exports:** `XML`, `XMLDocument`, `XMLNode`, `DOMNode`,
`DOMDocument`, `DOMElement`, `DOMComment`, `DOMText`.

## std/data/xml/escape
Provides lightweight XML entity escaping and unescaping helpers for plain text values. It supports named entities and numeric entities (decimal and hexadecimal).

**Exports:** `escape_xml`, `unescape_xml`.

## std/data/yaml
Implements YAML parsing and serialization with formatting controls for typical data-exchange workflows. It includes file load/dump convenience methods.

**Exports:** `YAML`.

## std/db
Provides database connectivity and statement execution helpers over the runtime DB backend. It exposes connection and prepared-statement handle abstractions.

**Exports:** `DB`, `DatabaseHandle`, `StatementHandle`.

## std/defer
Defines a guard object used to model deferred cleanup behaviour. The guard can be enabled/disabled and runs its callback during teardown.

**Exports:** `Guard`.

## std/digest/crc32
Provides CRC-32 digest functions returning numeric, hexadecimal, or base64 forms. It is intended for checksums rather than cryptographic hashing.

**Exports:** `crc32`, `crc32_b64`, `crc32_hex`

## std/digest/md5
Provides MD5 digest helpers for raw bytes and text encodings. It includes raw binary, hex, and base64 output forms.

**Exports:** `md5`, `md5_b64`, `md5_hex`.

## std/digest/sha
Provides SHA-family hashing functions and HMAC helpers across several bit widths. It supports raw binary, hex, and base64 result encodings.

**Exports:** `sha1`, `sha1_b64`, `sha1_hex`, `sha224`, `sha224_b64`, `sha224_hex`, `sha256`, `sha256_b64`, `sha256_hex`, `sha384`, `sha384_b64`, `sha384_hex`, `sha512`, `sha512_b64`, `sha512_hex`, `hmac_sha1`, `hmac_sha1_b64`, `hmac_sha1_hex`, `hmac_sha224`, `hmac_sha224_b64`, `hmac_sha224_hex`, `hmac_sha256`, `hmac_sha256_b64`, `hmac_sha256_hex`, `hmac_sha384`, `hmac_sha384_b64`, `hmac_sha384_hex`, `hmac_sha512`, `hmac_sha512_b64`, `hmac_sha512_hex`.

## std/dump
Provides a structured dumper that turns Zuzu values into readable, code-like text. It supports pretty-printing, optional colour output, and cycle detection safeguards.

**Exports:** `Dumper`.

## std/eval
Provides dynamic evaluation of Zuzu source strings within the runtime. It is primarily intended for controlled metaprogramming and tooling scenarios.

**Exports:** `eval`.

## std/getopt
Provides command-line option parsing utilities with both spec-based and schema-based interfaces. It centralizes argv decoding and result shaping.

**Exports:** `Getopt`.

## std/gui
Provides user-facing GUI constructor helpers, XML layout helpers, and
binding utilities over the runtime-supported GUI object model. The Perl
runtime uses Prima; `zuzu-rust` uses GTK4.

**Exports:** `Window`, `VBox`, `HBox`, `Frame`, `Label`, `Text`,
`RichText`, `Image`, `Input`, `DatePicker`, `Checkbox`, `Radio`,
`RadioGroup`, `Select`, `Menu`, `MenuItem`, `Button`, `Separator`,
`Slider`, `Progress`, `Tabs`, `Tab`, `ListView`, `TreeView`, `Widget`,
`Event`, `ListenerToken`, `BindingToken`, `EM`, `bind`, `unbind`,
`gui_from_xml`, `gui_from_xml_file`, `gui_to_xml`, `native_file_open`,
`native_file_save`, `native_directory_open`, `native_directory_save`,
`native_colour_picker`.

## std/gui/dialogue
Provides modal dialogue helpers for alerts, confirmations, prompts,
file and directory pickers, and colour selection. When GUI access is
denied, it falls back to terminal prompts using `std/tui`.

**Exports:** `alert`, `alert_window`, `confirm`, `confirm_window`,
`prompt`, `prompt_window`, `file_open`, `file_open_window`,
`file_save`, `file_save_window`, `directory_open`,
`directory_open_window`, `directory_save`, `directory_save_window`,
`colour_picker`, `colour_picker_window`.

## std/gui/objects
Provides the runtime-supported GUI object classes and backend-native
dialogue hooks used by `std/gui`. Most user code should import
`std/gui` instead of this lower-level module.

**Exports:** `Widget`, `Window`, `VBox`, `HBox`, `Frame`, `Label`,
`Text`, `RichText`, `Image`, `Input`, `DatePicker`, `Checkbox`,
`Radio`, `RadioGroup`, `Select`, `Menu`, `MenuItem`, `Button`,
`Separator`, `Slider`, `Progress`, `Tabs`, `Tab`, `ListView`,
`TreeView`, `Event`, `ListenerToken`, `native_file_open`,
`native_file_save`, `native_directory_open`, `native_directory_save`,
`native_colour_picker`, `meta`.

## std/internals
Exposes low-level runtime helper functions intended for internal/advanced modules. These helpers cover reflection-ish tasks, object allocation without `__build__`, and special property access. The API provided by this module should not be considered stable.

**Exports:** `ansi_esc`, `class_name`, `getprop`, `make_instance`, `object_slots`, `ref_id`, `setprop`.

## std/io
Provides path and filesystem APIs plus standard input/output stream objects. It is the main I/O surface for reading/writing files and traversing directories.

**Exports:** `Path`, `PathIterator`, `STDERR`, `STDIN`, `STDOUT`.

## std/io/socks
Provides socket helpers for TCP, UDP, and Unix-domain communication. It includes both listening and client connection entry points.

**Exports:** `bind_udp`, `connect_tcp`, `connect_udp`, `connect_unix`, `listen_tcp`, `listen_unix`.

## std/lingua/en
Provides English-language formatting helpers for lists and numbers. It supports cardinal and ordinal wording with configurable cutoff behaviour.

**Exports:** `english_list`, `english_number`, `english_ordinal`.

## std/log
Provides configurable structured-ish logging with levels and optional timestamps. It offers convenience level methods and a generic log entrypoint.

**Exports:** `Log`.

## std/mail
Provides mail address, header, body, message parsing, date formatting, and
message serialization helpers. It is intended for RFC-style message
composition and parsing; delivery is handled by `std/net/smtp`.

**Exports:** `Address`, `Head`, `Body`, `Message`, `Parser`,
`Serializer`, `parse_datetime`, `format_datetime`.

## std/marshal
Provides Zuzu Marshal CBOR v1 binary serialization through `dump`,
`load`, and `safe_to_dump`. It supports scalars, core collections,
Times, Paths, marshalable user functions/classes/traits/objects, shared
references, and supported cycles. `load` may evaluate embedded code
records and run `__on_load__`, so marshal blobs are only appropriate for
trusted data. Version 1 preserves weak object slots and weak collection
entries using weak storage records; dead or non-strongly-reachable weak
references load as weak `null` records.

**Exports:** `dump`, `load`, `safe_to_dump`,
`MarshallingException`, `UnmarshallingException`.

## std/math
Provides general math constants and static helpers for trigonometric, exponential, and numeric operations. It serves as the primary standard math namespace.

**Exports:** `Math`, `π`.

## std/math/bignum
Provides arbitrary-precision numeric operations via a `BigNum` class. It supports decimal/hex constructors plus broad arithmetic and conversion methods.

**Exports:** `BigNum`.

## std/math/range
Provides helper logic to abbreviate numeric lists into compact range-like representations. It is mainly geared toward reporting and diagnostics output.

**Exports:** `abbreviate`.

## std/math/roman
Provides conversion of numbers into Roman numeral text. It includes support for integer and fractional twelfths notation.

**Exports:** `roman`.

## std/net/dns
Provides runtime-supported DNS lookup helpers for forward record lookups,
address extraction, and reverse DNS. Synchronous and asynchronous entry
points are available where the runtime supports them.

**Exports:** `lookup`, `lookup_async`, `addresses`,
`addresses_async`, `reverse`, `reverse_async`.

## std/net/http
Provides HTTP request/response primitives and user-agent utilities for network calls. It also includes cookie-jar support for request state.

**Exports:** `CookieJar`, `Request`, `Response`, `UserAgent`.

## std/net/smtp
Provides runtime-supported low-level mail delivery over SMTP or
sendmail-compatible transports. It works with explicit envelope senders,
recipients, ordered headers, and raw binary message bodies.

**Exports:** `Mailer`, `MailResult`.

## std/net/url
Provides URL escaping/unescaping, parsing, and template filling helpers. It is used for URL normalization and query/path manipulation tasks.

**Exports:** `escape`, `fill_template`, `parse`, `unescape`.

## std/path/kdl
Provides KDL Query Language selectors for KDL documents and nodes. It
offers the same read-oriented path API shape as `std/path/z` and
`std/path/simple`, plus KDL-specific value and property helpers.

**Exports:** `KDLQuery`.

## std/path/simple
Provides a simplified path-expression utility for navigating nested structures with concise selectors. It is a lighter-weight alternative to full ZPath queries.

**Exports:** `SimplePath`.

## std/path/z
Provides a full ZPath query engine. See L<https://zpath.me>.

**Exports:** `ZPath`.

## std/path/zz
Provides ZuzuScript-flavoured path selectors. It reuses the ZPath
traversal, parser, lexer, assignment, and reference machinery while using
ZZPath expression operators and functions.

**Exports:** `ZZPath`.

## std/proc
Provides process execution and signal helpers, including pipeline orchestration and environment utilities. It is the standard interface for subprocess control.

**Exports:** `Env`, `Proc`.

## std/result
Provides a small, subclassable `Result` object for returning explicit
success or failure values without throwing an exception. It is modelled on
a simplified Rust-style result type.

**Exports:** `Result`.

## std/secure
Provides runtime-supported secure services and capability inspection for
random generation, password hashing, key derivation, authenticated
encryption, signing, key agreement, certificate handling, and TLS identity
objects.

**Exports:** `Secure`, `SecureRandom`, `PasswordHash`,
`KeyDerivation`, `Cipher`, `KeyAgreement`, `SigningKey`,
`Certificate`, `PrivateKey`, `PublicKey`, `SealedBox`, `TlsIdentity`.

## std/string
Provides common string manipulation and formatting helpers used across user modules and serializers. It includes substring, search, case conversion, and join/split helpers.

**Exports:** `camel`, `chomp`, `index`, `join`, `kebab`, `pad`, `pattern_to_regexp`, `replace`, `rindex`, `search`, `snake`, `split`, `sprint`, `substr`, `title`, `trim`.

## std/string/base64
Provides base64 and URL-safe base64 encode/decode helpers. It converts between binary values and textual transport forms.

**Exports:** `decode`, `decode_urlsafe`, `encode`, `encode_urlsafe`.

## std/string/quoted_printable
Provides quoted-printable encoding and decoding helpers for RFC 2045-style
byte transport. Encoding returns ASCII text; decoding returns a
`BinaryString`.

**Exports:** `encode`, `decode`.

## std/task
Provides the task type, cancellation objects, channels, timers, and
combinators used with `async`, `await`, and `spawn`.

**Exports:** `Task`, `Channel`, `CancellationToken`,
`CancellationSource`, `CancelledException`, `ChannelClosedException`,
`TimeoutException`, `resolved`, `failed`, `sleep`, `yield`, `all`,
`race`, `timeout`.

## std/template/z
Provides a full ZTemplate templating engine. See L<https://zpath.me>.

**Exports:** `ZTemplate`.

## std/template/zz
Provides a ZZPath-backed template engine. It subclasses `std/template/z`'s
`ZTemplate` and renders template expressions using `std/path/zz`.

**Exports:** `ZZTemplate`.

## std/time
Provides time value and parser utilities, including format-driven parsing of date/time strings. It is the standard time abstraction for core scripts.

**Exports:** `Time`, `TimeParser`.

## std/tui
Provides terminal UI helpers for ANSI colour, stdout writing, readline,
and file/directory completion. It is used by GUI dialogue fallbacks when
GUI access is denied.

**Exports:** `ansi_esc`, `supports_ansi`, `colour_text`, `write`,
`write_line`, `readline`, `readline_supports_completion`,
`filename_completions`, `directory_completions`.

## std/uuid
Provides UUID generation helpers for both textual and binary forms. It is intended for unique identifier generation in app and test contexts.

**Exports:** `create_uuid`, `create_uuid_binary`.

## std/web
Provides request, response, route, and route collection objects for simple
web applications. It includes parameter parsing, cookies, rendering
helpers, route matching, and dispatch support.

**Exports:** `Request`, `Response`, `RouteMatch`, `Route`, `Routes`.

## std/worker
Provides isolated workers for shared-nothing parallel work. Worker
payloads, return values, and messages are copied through `std/marshal`;
if `std/worker` imports successfully, workers are available. The module
requires the `worker` runtime capability.

**Exports:** `Worker`, `WorkerHandle`.

## std/zuzuzoo
Provides distribution metadata queries, source archive inspection,
dependency-aware install and removal planning, installation, verification,
latest-version checks, upgrade checks, and formatting helpers for Zuzu
distribution workflows.

**Exports:** `ZuzuzooLock`, `Zuzuzoo`, `compare_versions`,
`list_installed`, `query`, `query_distribution`, `is_installed`,
`installed_version`, `pretty_json`, `format_json`, `fetch_source`,
`load_distribution`, `dependency_roots`, `find_dependency`,
`plan_install`, `plan_remove`, `verify`, `latest`, `can_upgrade`,
`install`, `remove`, `run_distribution_tests`, `execute_removal`,
`format_install_plan`, `format_remove_plan`.

## test/more
Provides TAP-style assertion helpers for tests, including ok/is/isnt-style checks and subtests. It is the common ergonomic testing layer for Zuzu test scripts.

**Exports:** `BailOutException`, `bail_out`, `diag`, `done_testing`, `exception`, `explain`, `fail`, `is`, `isnt`, `like`, `ok`, `pass`, `plan`, `subtest`, `unlike`.

## test/parser
Provides parsing helpers for TAP-like test output into structured counters and result rows. It is used by the test toolchain to interpret test streams.

**Exports:** `parse`, `parse_lines`.
