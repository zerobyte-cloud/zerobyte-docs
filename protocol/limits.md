# Limits

| Limit | Value | Enforced by | Answer when exceeded |
|---|---|---|---|
| Plaintext | 1 MiB (1,048,576 bytes) | Sender | `too_large` before sealing |
| Envelope | 38 B to 1 MiB + 38 B | Relay | `413` above, `400` below |
| Envelopes waiting per direction | 64 | Relay | `507` |
| Mailbox size, both directions | 32 MiB | Relay | `507` |
| Envelopes per fetch | 16 | Relay | Larger `limit` is reduced to 16 |
| Envelopes per wire body | 64 | Client | Body rejected as malformed |
| Sequence | 1 to 2^64 - 1 | Both | `malformed` in a client, `404` for an invalid path segment |
| Watch sockets per direction | 8 | Relay | `429` with `Retry-After: 60` |
| Idle mailbox | 30 to 31 days after the last write or claim | Relay | Mailbox erased |
| Requests per mailbox | See [rate limits](extensions.md#rate-limits) | Relay | `429` with `Retry-After` |

These are the defaults. An operator may give an app its own envelope count (16 to 4096), mailbox
size (1 to 256 MiB), largest plaintext (1 KiB to 1 MiB) and idle time (1 to 365 days); see
[configuration](../relay/configuration.md#app-profiles). A client learns that it hit one from
`413` and `507`, exactly as with the defaults.

## What counts toward the quotas

Only envelopes waiting to be read. Pruning frees both the slot and the bytes immediately. A
receiver that keeps up never comes close to either quota: the limits exist so that a link whose
other device has gone away for good cannot grow without bound.

## The idle timer

Every claim and every stored envelope restarts a 30-day timer. The relay sets its alarm a day
later than that and moves it only when it falls short, so most writes cost no extra storage
write; a mailbox goes 30 to 31 days after its last write or claim. Fetching, pruning and watching do
not. When the timer fires, the relay erases the mailbox exactly as `DELETE /mailbox/<id>` would.

Consequences:

- A link where only one side writes stays alive as long as that side writes at least once every 30
  days.
- A link where nobody writes for 30 days loses its mailbox. Both devices still hold the link key,
  so the next request claims a fresh, empty mailbox under the same id and the link continues.
  Envelopes that were waiting when it expired are lost. `X-Relay-Last` starts again from nothing,
  so a sender continues from 1. A receiver holding a higher `last_received` would refuse those as
  replays. See [Link lifecycle](../guides/link-lifecycle.md) for how to handle this.

## Sizing your messages

With 64 slots and 32 MiB per mailbox, a direction can hold, for example, 64 messages of 512 KiB,
or 32 messages of 1 MiB before the mailbox total runs out. If your companion may be offline for
days while the primary keeps producing changes, coalesce changes into fewer, larger messages, or
send a "full state" message that supersedes older ones. See
[Designing messages](../guides/message-design.md). A [timeline](timeline.md) does this itself: it
sends only the keys that changed, packs its commits into as few envelopes as fit, and keeps what the
mailbox has no room for until its next run.
