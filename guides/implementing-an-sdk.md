# Implementing an SDK

For a platform the four SDKs do not cover (Rust, Go, Python, Flutter, a game engine), you can
write your own client. The protocol is small: one key derivation, one AEAD, six HTTP calls and one
WebSocket. This page is the checklist. The specification is in [Relay Protocol v1](../protocol/README.md).

## What you need from the platform

| Need | Typical source |
|---|---|
| HKDF-SHA256 | The standard library, or HMAC-SHA256 plus 20 lines (RFC 5869) |
| SHA-256 | The standard library |
| ChaCha20-Poly1305 (IETF, 12-byte nonce) | The standard library, libsodium (`crypto_aead_chacha20poly1305_ietf_*`), BoringSSL, RustCrypto |
| A secure random generator | The operating system |
| HTTPS with control over headers | Any HTTP client |
| WebSocket with subprotocols | Optional; needed only for watching |

Do not implement ChaCha20 or Poly1305 yourself unless the platform has nothing. If you must, test
against RFC 8439 §2.5.2 and §2.8.2 before anything else.

## Layers

Build it in this order, each tested before the next:

1. **Credentials.** Derive mailbox id, access token and the two direction keys from the link key
   and link id. Encode the mailbox name (hex), bearer (base64url, no padding) and token digest
   (SHA-256 of the 32 raw token bytes, hex). See [Keys and derivation](../protocol/keys.md).
2. **Envelope.** `seal` and `open`, plus reading the header without opening. Apply the receiver's
   checks in the documented order. See [Envelope format](../protocol/envelope.md).
3. **Wire.** Split and join the multi-envelope body of a fetch. See
   [HTTP API](../protocol/http-api.md#wire-body).
4. **Relay client.** The six calls: claim, put, fetch, last sequence, prune, erase. Map status codes
   to errors. Keep it behind an interface so tests can replace it.
5. **Link.** Sequence choice and conflict retry on send, the fetch-open-prune loop on receive,
   erase. One operation at a time. See [Client behaviour](../protocol/client-behaviour.md).
6. **Watch.** Optional. See [Watch WebSocket](../protocol/watch.md).
7. **Timeline.** Optional: one keyed state on both devices, carried in the link's envelopes. See
   [Timeline v1](../protocol/timeline.md), and test it against `spec/timeline.json` as well.

## Test vectors

Copy [`spec/vectors.json`](../protocol/test-vectors.md) into your test resources and load it in
your tests. Do not generate it.

| Section | Test |
|---|---|
| `credentials` | Every derived value matches, byte for byte, for both sets |
| `envelopes` | `seal` with the given nonce produces exactly `envelope`; `open` returns `plaintext` and the sequence |
| `rejections` | `open` fails with the named error for each of the 11 cases, and does not return plaintext |
| `wire` | `split` returns the listed envelopes; `join` of them returns the body |
| `wireRejections` | `split` fails for each of the 5 cases |

Your `seal` needs an internal way to take a fixed nonce for these tests. Do not expose it in the
public API, or mark it clearly as test-only.

Sequence numbers in the file are decimal strings. Parse them as unsigned 64-bit integers: the
vectors include 2^64 - 1, which overflows signed 64-bit types and loses precision in a double.

## Checklist

Keys:

- [ ] Rejects a link key that is not 32 bytes and a link id that is not 16.
- [ ] Uses the link id as the HKDF **salt** and the labels as **info**, with the exact label strings.
- [ ] Reads a UUID link id in RFC 4122 byte order (as written), not a platform's mixed-endian order.

Envelopes:

- [ ] Uses a fresh random nonce for every seal, including a retry after a conflict.
- [ ] Builds the AAD as the 10-byte header followed by the 16-byte mailbox id.
- [ ] Uses the direction key for the envelope's direction, not the sender's role.
- [ ] Refuses plaintext above 1 MiB before sealing.
- [ ] On open, checks length, version, direction and sequence (against the last accepted) before
      decrypting, and returns no plaintext on any failure.
- [ ] Never sends sequence 0: sequences run from 1 to 2^64 - 1, and the relay answers `404` to 0.

Relay client:

- [ ] Builds URLs from a base that ends in `/v1/`, the app id and the mailbox name.
- [ ] Sends `Authorization: Bearer <token>` on every request, and the token digest in the claim body.
- [ ] Does not follow redirects, does not store cookies, does not cache.
- [ ] Reads `X-Relay-Last` on `409` and on `HEAD`, and `X-Relay-More` on fetch.
- [ ] Parses the fetch body as the wire format and rejects a malformed one.
- [ ] Never logs the bearer token or the link key.

Link:

- [ ] Gets the starting sequence from `HEAD` at the first send in each process.
- [ ] On `409`, continues from `X-Relay-Last` + 1 and seals again with a new nonce.
- [ ] On receive, advances the cursor past envelopes that fail to open, prunes everything seen, and
      stops when the cursor does not move.
- [ ] Updates `lastReceived` only for envelopes that opened.
- [ ] Runs one operation at a time per link.

Watch:

- [ ] Offers `zerobyte.v1`, and authenticates with the header where possible or the
      `zerobyte.bearer.<token>` subprotocol where headers are impossible.
- [ ] Pulls once when the socket opens.
- [ ] Reports `1000 erased` distinctly from other closes.

## Interoperability check

When the vectors pass, run your client against a local relay together with an existing SDK: one
device side yours, the other side, for example, the JavaScript SDK from a Node script. Send in both
directions, restart each side, and unpair from each side. The
[quickstart](../getting-started/quickstart.md) scripts are a ready-made other side.

## Publishing

If you publish your SDK, say which protocol version it implements (`zerobyte-relay-v1`), which
vectors file it was tested against, and do not name it as an official ZeroByte SDK. Tell me in an
issue on [zerobyte-protocol](https://github.com/zerobyte-cloud/zerobyte-protocol/issues) and I may
link to it.
