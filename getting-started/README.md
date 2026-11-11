# Overview

ZeroByte is three things that share one specification:

1. **Relay Protocol v1**: how two devices derive keys from a shared secret, how they seal
   messages into envelopes, and the small HTTP API a relay offers.
2. **The relay**: a blind mailbox server. One file of protocol logic, run either as a Node server
   with one SQLite file or as a Cloudflare Worker with one Durable Object per mailbox. No
   dependencies.
3. **The SDKs**: Swift, Kotlin, JavaScript and .NET libraries that implement the client side. Each
   is tested against the same known-answer vectors.

## The problem it solves

A phone app with a desktop or watch companion needs to move data between them. The usual
options each cost something:

- A cloud backend with accounts sees every byte and becomes a liability.
- Peer-to-peer on the local network only works while both devices are on the same network and
  awake at the same time.
- Platform sync services tie you to one vendor and often to one platform.

ZeroByte sits between these. The relay is a store-and-forward mailbox, so the devices do not need
to be online together. Everything it stores is end-to-end encrypted under keys only the two
devices hold, so the relay operator learns nothing beyond traffic metadata. You can run the relay
yourself in a few minutes.

Optional [extensions](../protocol/extensions.md) add what a dedicated app needs on top: pushes that
wake the other device (APNs and FCM, as separate modules), and a sealed signal channel with STUN/TURN for
setting up a direct, peer-to-peer connection while both devices are online.

[Timeline v1](../protocol/timeline.md), an optional library in every SDK, keeps one keyed state on
both devices over the same envelopes. Each device sends only the keys it changed, both merge every
change into the same order, and a device that is new or lost its copy catches up from the other.
It needs nothing from the relay.

## Where it fits

ZeroByte is for data the servers in between should not be able to read, moving between two
devices that met once:

- **Sports performance.** An athlete's phone owns training and health data; the team's sports
  scientist analyses it on a Mac where the models run locally. The two are rarely on the same
  network.
- **Health.** A patient's app sends readings to a clinician's device. The relay holds them sealed
  until they are collected.
- **Private AI on your own hardware.** A phone sends data to a model on its owner's computer and
  gets the answer back, with no cloud service in the middle that could read either.
- **Sync without a backend.** A phone and a laptop, or a tablet and a watch, keep one app's private
  data in step without a server that stores it or an account to protect. A
  [timeline](../protocol/timeline.md) merges both devices' changes into one state.

## How it fits together

```
 Primary device                       Relay                        Companion device
 ───────────────                      ─────                        ────────────────
 link key K, link id L                                             link key K, link id L
   │ HKDF                                                            │ HKDF
   ├─ mailbox id ───────────────▶  /v1/<app>/mailbox/<id>  ◀──────────┤ mailbox id
   ├─ bearer token ─────────────▶  stores SHA-256(token)   ◀──────────┤ bearer token
   ├─ p2c key (seal)                                                  ├─ p2c key (open)
   └─ c2p key (open)                                                  └─ c2p key (seal)

 send:  seal(p2c) ──PUT p2c/<seq>──▶  [ envelope ]  ──GET p2c?after──▶ open(p2c), then prune
 watch:                               WebSocket: "17" ─────────────▶ fetch now
```

1. **Pair.** Your app gives both devices the same 32-byte link key and 16-byte link id. ZeroByte
   does not take part in pairing and never sees the key. See [Pairing devices](../guides/pairing.md).
2. **Derive.** Each SDK derives, from the link key and link id, the mailbox id, the bearer token
   and one encryption key per direction.
3. **Send.** `Link.send` seals a message with ChaCha20-Poly1305, numbers it, and stores it in the
   mailbox.
4. **Receive.** The other device watches its direction over a WebSocket, fetches new envelopes,
   opens them and prunes what it has read from the relay.
5. **Unpair.** Either device erases the mailbox. A mailbox nobody writes to for 30 days erases
   itself.

## What ZeroByte is not

- **Not a general sync engine.** A link delivers opaque messages in order between exactly two
  devices. [Timeline v1](../protocol/timeline.md) keeps one keyed state on both, where the latest
  change to a key wins; finer merges, such as of text or counters, are your app's job. See
  [Designing messages](../guides/message-design.md).
- **Not a backup.** Envelopes are deleted once read and the whole mailbox is deleted after 30 idle
  days. Nothing on the relay is meant to last.
- **Not group messaging.** A link connects two devices. A phone with three companions has three
  links.
- **Not anonymous.** The relay sees IP addresses while requests last, request times and envelope
  sizes. See [Security model](../SECURITY.md).
- **Not a pairing protocol.** You choose how the two devices agree the link key. The guide
  describes two sound ways.

## Requirements at a glance

| Part | Requirement |
|---|---|
| Relay on Node | Node 22.13 or newer, one writable directory |
| Relay on Docker | Docker with Compose, a DNS name for HTTPS |
| Relay on Cloudflare | A Cloudflare account with Workers and Durable Objects |
| Swift SDK | iOS 16, macOS 13, watchOS 9, tvOS 16, visionOS 1 |
| Android SDK | Android API 28, or any JVM 11 |
| JavaScript SDK | Browsers with WebCrypto, Node 20, Deno, Bun, Cloudflare Workers |
| .NET SDK | .NET 10; ChaCha20-Poly1305 support (Windows 10 20H2 or newer) |

## Next

- [Concepts](concepts.md) explains the vocabulary used everywhere else.
- [Quickstart](quickstart.md) runs a relay on your machine and sends a first message.
