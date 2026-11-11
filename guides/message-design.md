# Designing messages

ZeroByte delivers opaque byte strings, in order, once each, between two devices. What you put in
them is up to you. This page collects what works.

## Format

Use a format with a type tag and a version, so you can change it later without breaking older app
versions on the other device:

```json
{ "v": 1, "type": "note.updated", "id": "4f1c...", "at": "2026-10-22T18:04:11Z", "body": { ... } }
```

JSON is fine for most apps. For large or binary payloads use a binary format (CBOR, Protocol
Buffers, MessagePack). Compress before sealing if your payload compresses well; ciphertext does not.

The receiver should ignore message types it does not know, so a newer primary can talk to an older
companion.

## Size

A message holds up to 1 MiB of plaintext. Larger data must be split:

```json
{ "v": 1, "type": "file.part", "file": "9b2e...", "part": 3, "of": 12, "data": "<base64>" }
```

Keep in mind the mailbox limits: 64 waiting envelopes per direction and 32 MiB total. A 12 MiB file
in 1 MiB parts fits only if the receiver is collecting while you send. Send parts as the receiver
confirms earlier ones, or wait for a `507` to clear before continuing.

## Changes or state

Two common designs:

**Send changes.** Each message describes one change: "note 17 renamed". Small and cheap, but the
receiver must apply every change in order and never miss one. If the companion is offline for a
long time and the primary keeps changing things, the mailbox fills up.

**Send state.** Each message carries the current state of something: "here is note 17". Larger, but
a later message supersedes an earlier one about the same thing, and a receiver that missed some can
still converge.

A combination works well: send changes normally, and when the sender sees `quota_exceeded` or when
the receiver asks for it, send a full snapshot instead.

**Use a timeline.** When both devices edit the same data, [Timeline v1](../protocol/timeline.md)
does all of this for you. It sends only the keys that changed, merges both devices' changes into
one state, keeps what does not fit the mailbox for the next run, and catches up a device that is
new, lost its copy or missed a commit, over several runs when the catch-up is long.

## Acknowledgements

ZeroByte does not tell the sender when the receiver has read a message. If your app needs that, the
receiver sends a small message back in its own direction:

```json
{ "v": 1, "type": "ack", "through": 42 }
```

`through` can be the sequence number `receive` returned. The sender can then drop anything it was
keeping for re-send.

## Asking for a resync

When a device suspects it is out of date (after a reinstall, a restored backup, or a gap in your
own change numbering), it sends:

```json
{ "v": 1, "type": "resync.request" }
```

and the other side answers with a snapshot. This makes the link self-healing no matter what
happened to the mailbox. See [Link lifecycle](link-lifecycle.md). A timeline asks for and answers
catch-ups itself.

## Idempotency

`receive` returns each envelope once, but your app can still see the same message twice: if it
crashes after handling messages and before saving `lastReceived`, they are fetched again on the
next run (as long as the relay has not pruned them). Make handling idempotent, for example by
carrying an id in each message and ignoring ids already applied, or save `lastReceived` in the
same database transaction as the data.

## What not to put in messages

Nothing needs to be hidden from the relay inside a message: the whole message is encrypted. But
the relay sees envelope **sizes** and **times**. If those alone would reveal something sensitive,
pad messages to fixed sizes and send on a schedule rather than on events.
