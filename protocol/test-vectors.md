# Test vectors

Every implementation of the protocol, the relay and the four SDKs included, tests against one
file of known answers:
[`spec/vectors.json`](https://github.com/zerobyte-cloud/zerobyte-protocol/blob/master/spec/vectors.json)
in zerobyte-protocol.

## Where it comes from

The file is produced by
[`spec/vectors.py`](https://github.com/zerobyte-cloud/zerobyte-protocol/blob/master/spec/vectors.py),
an independent reference implementation that:

- uses only the Python standard library;
- writes out ChaCha20, Poly1305 and the AEAD construction from RFC 8439 and checks them against the
  RFC's own test vectors (§2.5.2 and §2.8.2) before writing anything;
- implements HKDF from RFC 5869 over `hmac` and `hashlib`;
- shares no code with any SDK or the relay.

```sh
python3 spec/vectors.py > spec/vectors.json
```

Implementations **copy** the file into their test resources. They never generate it from their
own code: a vector produced by the implementation under test proves nothing. To change a vector,
change `vectors.py`, review the diff of `vectors.json`, and copy the result into every SDK.

## Structure

```json
{
  "protocol": "zerobyte-relay-v1",
  "note": "...",
  "labels": { "mailbox": "...", "token": "...", "p2c": "...", "c2p": "..." },
  "credentials": [ ... ],
  "envelopes": [ ... ],
  "rejections": [ ... ],
  "wire": [ ... ],
  "wireRejections": [ ... ],
  "signal": { "labels": { ... }, "windowMilliseconds": 300000, "maximumFrameBytes": 65536,
              "keys": [ ... ], "frames": [ ... ], "rejections": [ ... ] }
}
```

Byte strings are lowercase hex. Sequence numbers are decimal strings, because they can exceed the
range JSON numbers represent exactly.

### `credentials` (2 sets)

| Field | Meaning |
|---|---|
| `linkKey` | 32 bytes |
| `linkId` | 16 bytes |
| `mailbox` | Expected mailbox id |
| `token` | Expected access token |
| `bearer` | Expected bearer text |
| `tokenDigest` | Expected SHA-256 of the token |
| `p2cKey`, `c2pKey` | Expected direction keys |

Test: derive from `linkKey` and `linkId`, compare every field.

### `envelopes` (4 cases)

| Field | Meaning |
|---|---|
| `credentials` | Index into `credentials` |
| `direction` | `1` or `2` |
| `sequence` | Decimal string |
| `nonce` | The fixed nonce to seal with |
| `plaintext` | Input |
| `envelope` | Expected output |

Test both ways: sealing with the given nonce must produce `envelope` byte for byte, and opening
`envelope` must return `plaintext` and `sequence`.

### `rejections` (11 cases)

| Field | Meaning |
|---|---|
| `credentials` | Index into `credentials` |
| `direction` | The direction the receiver reads |
| `after` | The last accepted sequence, or `null` |
| `envelope` | Input |
| `error` | Expected error code |
| `why` | What was done to the envelope, for humans |

Test: opening must fail with exactly `error`. The cases cover a flipped tag byte, a flipped
ciphertext bit, a renumbered sequence, another link's keys, a direction byte changed to match the
reader, an envelope read in the wrong direction, a replay, an unknown version, an envelope shorter
than 38 bytes, an unknown direction byte and sequence 0.

### `envelopes` coverage

The four envelopes cover both directions, an empty plaintext, a plaintext of 516 bytes, sequence
256 (a carry into the second byte) and the largest sequence, 2^64 - 1.

### `wire` and `wireRejections`

`wire` pairs lists of envelopes with the expected fetch body; test both `join` and `split`.
`wireRejections` lists bodies that `split` must refuse, each with a `why`.

### `signal`

Vectors for the [signal extension](extensions.md#signal). Frames have the envelope's layout under
the signal keys; the sequence field holds a counter in milliseconds since 1970.

| Field | Meaning |
|---|---|
| `labels` | The two HKDF labels, `zerobyte-signal-p2c-v1` and `zerobyte-signal-c2p-v1` |
| `keys` | Expected `p2cKey` and `c2pKey` for each entry of `credentials`, in the same order |
| `frames` (2) | `credentials`, `direction`, `counter`, `now`, `nonce`, `plaintext`, `frame`: seal with the nonce, compare byte for byte, then open at `now` |
| `rejections` (5) | `credentials`, `direction`, `after` (last accepted counter or `null`), `now`, `frame`, `error`, `why` |

The rejections cover a flipped tag byte, a frame sealed under the envelope key instead of the
signal key (`authentication_failed`), a counter not above the last accepted one and a counter more
than five minutes from `now` (`replayed`), and a frame read in the wrong direction
(`wrong_direction`).

## Running the tests in each repository

| Repository | Vectors copied to | Command |
|---|---|---|
| zerobyte-relay | `vectors/relay-v1.json` | `npm test` |
| zerobyte-swift-sdk | `Tests/ZeroByteTests/relay-v1.json` | `swift test` |
| zerobyte-android-sdk | `zerobyte/src/test/resources/relay-v1.json` | `./gradlew test` |
| zerobyte-js-sdk | `test/relay-v1.json` | `npm test` |
| zerobyte-dotnet-sdk | `test/ZeroByte.Tests/relay-v1.json` | `dotnet test` |

Every SDK also has tests against a live relay. They run when two environment variables are set:

```sh
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=<registered app id> <test command>
```

`RELAY_SIGNAL_APP=<app id with signal;ice>` adds the live tests of the extensions.

## Timeline v1

[Timeline v1](timeline.md) has its own file,
[`spec/timeline.json`](https://github.com/zerobyte-cloud/zerobyte-protocol/blob/master/spec/timeline.json),
produced the same way by
[`spec/timeline.py`](https://github.com/zerobyte-cloud/zerobyte-protocol/blob/master/spec/timeline.py),
an independent reference that uses only the Python standard library. It holds clock steps, three
commits with their encodings and ids, one message of each type, two merge orders that must reach
the same state, a catch-up selection and the rejections.

```sh
python3 spec/timeline.py > spec/timeline.json
```

| Repository | Copied to |
|---|---|
| zerobyte-swift-sdk | `Tests/ZeroByteTimelineTests/timeline.json` |
| zerobyte-android-sdk | `zerobyte-timeline/src/test/resources/timeline.json` |
| zerobyte-js-sdk | `test/timeline.json` |
| zerobyte-dotnet-sdk | `test/ZeroByte.Tests/timeline.json` |

The same test commands run them, together with two timelines syncing through a relay in memory.
