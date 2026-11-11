# Timeline v1

Timeline v1 keeps one keyed state on both devices of a link and sends only what changed. The
canonical text is
[`timeline-v1.md`](https://github.com/zerobyte-cloud/zerobyte-protocol/blob/v1.2.0/timeline-v1.md)
in zerobyte-protocol. This page explains how it works and when to use it.

It runs on top of Relay Protocol v1 and changes nothing in it. A relay that carries envelopes
carries a timeline, and it sees what it saw before: envelope sizes and times.

## When to use it

Use a timeline when both devices edit the same data, such as a plan, a list or settings, and both
must end up with the same copy. Use plain messages ([Designing messages](../guides/message-design.md))
when data flows one way, or when each message is an event rather than a state.

## The model

| Term | Meaning |
|---|---|
| Replica | One device's copy. It names itself with a random 16-byte replica id and draws a new one whenever it has lost its state |
| Slot | A collection (1 to 64 bytes of UTF-8) and a key (1 to 1,024 bytes), both chosen by the app |
| Entry | What a slot holds: a value of bytes, or a deletion |
| Commit | One replica's changes, numbered 1, 2, 3 … per replica and chained to the one before by its SHA-256 id |
| Stamp | A commit's clock, then its replica id: the order of the timeline |

A slot holds the entry with the greatest stamp. A deletion competes like a value, so a value
written before a deletion never comes back. The merge is commutative, associative and idempotent:
two replicas that merged the same commits hold the same state, whatever order the commits arrived
in.

## The clock

The clock is a 64-bit hybrid logical clock: milliseconds since 1970 shifted left by 16 bits, plus
a 16-bit counter. A commit takes `max(last + 1, now_ms << 16)`, where `last` is the greatest clock
the replica has written or merged. A change made after seeing another therefore always orders after
it, whatever the two devices' wall clocks say. Wall clocks decide only between changes made without
seeing each other.

## Messages

Each message is the plaintext of one envelope: at most 1 MiB, or the lower limit a relay set for
the app.

| Type | Name | Carries |
|---|---|---|
| `0x01` | commits | One or more commits |
| `0x02` | state | The heads this replica has merged; may ask for a catch-up |
| `0x03` | catch-up | Every entry the requester has not seen, in parts; only the final part carries heads |

A replica asks for a catch-up when it is new, when it has lost its state, or when it found a gap in
another replica's commits.

## One run over the relay

Each replica writes in its own direction: the primary `p2c`, the companion `c2p`. One run:

1. Fetch the other direction, open each envelope and merge each message in order.
2. Persist, then prune what was fetched.
3. Commit pending changes.
4. Answer a catch-up request, if there is one.
5. Write the commits and catch-up parts not yet written; then a state if the run merged something
   and wrote no commit, or if it asks for a catch-up.
6. On `quota_exceeded`, stop and keep the rest for the next run.

The last envelope of a run may ask the relay for a silent push to wake the other device.

Two devices on the same network can skip the relay: each sends its state, answers the other's with
a full catch-up, and closes. Commits the relay delivers later are skipped as already merged.

## Errors

| Error | Means |
|---|---|
| `malformed` | Bytes out of order, out of range, or left over |
| `unsupported_version` | A commit version or a message type this replica does not know |
| `too_large` | A commit or message over the limit, or a single change that cannot fit in a commit of its own |
| `diverged` | A replica's history does not continue the one already merged: the same sequence with another id, or a wrong parent or clock |

## Versioning

A commit's first byte is the Timeline version, `0x01`. Timeline versions are independent of the
relay protocol's: Timeline v1 runs on Relay Protocol v1.

## SDKs

Every SDK implements Timeline v1 from 1.2.0: `ZeroByteTimeline` in [Swift](../sdk/swift.md#timeline),
`zerobyte-timeline` in [Kotlin](../sdk/android.md#timeline), `@zerobyte-cloud/sdk/timeline` in
[JavaScript](../sdk/javascript.md#timeline) and `ZeroByte.Timeline` in [.NET](../sdk/dotnet.md#timeline).
All four pass the test vectors below.

## Test vectors

`spec/timeline.json` in zerobyte-protocol holds clock steps, three commits with their encodings and
ids, one message of each type, two merge orders that must reach the same state, a catch-up
selection and the rejections. `spec/timeline.py`, an independent reference that uses only the
Python standard library, produces it. Implementations copy the file; they never regenerate it.
