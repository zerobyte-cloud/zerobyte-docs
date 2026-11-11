# Relay Protocol v1

Relay Protocol v1 is everything a client or a relay needs to interoperate. The canonical text is
[`relay-v1.md`](https://github.com/zerobyte-cloud/zerobyte-protocol/blob/master/relay-v1.md) in
zerobyte-protocol. The pages here explain the same rules at more length, with the reasoning
behind them and worked examples.

| Page | Covers |
|---|---|
| [Keys and derivation](keys.md) | Link key, link id, HKDF labels, the bearer token and its digest |
| [Envelope format](envelope.md) | Layout, authenticated data, the order of checks a receiver applies, error codes |
| [HTTP API](http-api.md) | Every endpoint, request and response, status codes, headers, the wire body |
| [Watch WebSocket](watch.md) | Notifications of new envelopes, authentication from browsers, reconnecting |
| [Client behaviour](client-behaviour.md) | Claiming, choosing sequence numbers, receiving and pruning, persistence |
| [Limits](limits.md) | Sizes, counts, idle expiry |
| [Extensions](extensions.md) | Optional, per app: rate limits, push, signal, ICE |
| [Timeline v1](timeline.md) | One keyed state on both devices, synced as commits on top of this protocol |
| [Test vectors](test-vectors.md) | The known-answer file every implementation tests against |

## Conventions

- Byte orders are big-endian.
- Hex is lowercase.
- Base64url is RFC 4648 §5 without padding.
- "MiB" is 1,048,576 bytes.
- `‖` is concatenation.

## Algorithms

| Purpose | Algorithm | Reference |
|---|---|---|
| Key derivation | HKDF-SHA256 | RFC 5869 |
| Authenticated encryption | ChaCha20-Poly1305, 96-bit nonce | RFC 8439 |
| Token digest | SHA-256 | FIPS 180-4 |
| App id | UUID version 7 | RFC 9562 §5.7 |
| Signal frames ([extension](extensions.md#signal)) | HKDF-SHA256 and ChaCha20-Poly1305, keys of their own | RFC 5869, RFC 8439 |
| Push registrations at rest ([extension](extensions.md#push)) | AES-256-GCM under a key derived from the bearer token | NIST SP 800-38D |

## Versioning

The envelope's first byte is the protocol version, `0x01`. URLs start with `/v1/`. The HKDF
labels end in `-v1`. A change to any of them is a new protocol version; a relay and the SDKs would
support both versions side by side during a transition. Additions that old clients can ignore,
such as a new response header, do not change the version.

The HKDF labels start with `zerobyte-relay-`. They
are part of the wire format and stay as they are.
