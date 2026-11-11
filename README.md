# ZeroByte documentation

ZeroByte moves private data between two paired devices of one app. Every message is sealed on
the device that sends it. A relay files the sealed envelope under a mailbox and deletes it once
the other device has collected it. The relay never holds a key, so it cannot read, alter or
forge what it carries.

Use it for a phone and its companions: a laptop, a tablet, a watch, a browser tab, a desktop
app. The two devices agree a 32-byte link key once, when they pair. Every identifier and every
key is derived from it on the devices.

This repository is the developer documentation for the open protocol, the open-source relay and
the four SDKs. It covers everything you need to run your own relay and build on it.

A mirror of this documentation is published at [docs.zerobyte.cloud](https://docs.zerobyte.cloud).
This repository is the source; if the two differ, this repository is current.

## Contents

### Getting started

- [Overview](getting-started/README.md): what ZeroByte is, what it is not, and how the parts fit
- [Concepts](getting-started/concepts.md): link, primary and companion, directions, mailbox, app
  id, sequence numbers
- [Quickstart](getting-started/quickstart.md): a local relay and two devices exchanging a message
  in ten minutes

### Protocol

- [Protocol overview](protocol/README.md)
- [Keys and derivation](protocol/keys.md)
- [Envelope format](protocol/envelope.md)
- [HTTP API](protocol/http-api.md)
- [Watch WebSocket](protocol/watch.md)
- [Client behaviour](protocol/client-behaviour.md)
- [Limits](protocol/limits.md)
- [Extensions: push, signal, ICE](protocol/extensions.md)
- [Timeline v1](protocol/timeline.md): one state kept on both devices, synced as commits
- [Test vectors](protocol/test-vectors.md)

### Running a relay

- [Relay overview](relay/README.md)
- [Node](relay/node.md)
- [Docker with automatic HTTPS](relay/docker.md)
- [Cloudflare Workers](relay/cloudflare-workers.md)
- [Configuration reference](relay/configuration.md)
- [Operations](relay/operations.md): TLS, storage, upgrades, monitoring, what the relay keeps

### SDKs

- [SDK overview](sdk/README.md): what every SDK has in common
- [Swift](sdk/swift.md): iOS, iPadOS, macOS, watchOS, tvOS, visionOS
- [Android and JVM](sdk/android.md): Kotlin
- [JavaScript](sdk/javascript.md): browsers, Node, Deno, Bun, Workers
- [.NET](sdk/dotnet.md): Windows desktop and services

### Guides

- [Pairing devices](guides/pairing.md)
- [Storing the link key](guides/key-storage.md)
- [Designing messages](guides/message-design.md)
- [Link lifecycle](guides/link-lifecycle.md): first claim, unpairing, expiry, recovery
- [Testing your integration](guides/testing.md)
- [Implementing a new SDK](guides/implementing-an-sdk.md)
- [Troubleshooting](guides/troubleshooting.md)

### Project

- [Security model](SECURITY.md)
- [Contributing](contributing.md)

## Repositories

| Repository | Contents | Licence |
|---|---|---|
| [zerobyte-protocol](https://github.com/zerobyte-cloud/zerobyte-protocol) | The specification and the test vectors | CC-BY-4.0; vectors also Apache-2.0 |
| [zerobyte-relay](https://github.com/zerobyte-cloud/zerobyte-relay) | The relay: Node server and Cloudflare Worker | AGPL-3.0-only |
| [zerobyte-swift-sdk](https://github.com/zerobyte-cloud/zerobyte-swift-sdk) | Swift package | Apache-2.0 |
| [zerobyte-android-sdk](https://github.com/zerobyte-cloud/zerobyte-android-sdk) | Kotlin library | Apache-2.0 |
| [zerobyte-js-sdk](https://github.com/zerobyte-cloud/zerobyte-js-sdk) | JavaScript package | Apache-2.0 |
| [zerobyte-dotnet-sdk](https://github.com/zerobyte-cloud/zerobyte-dotnet-sdk) | .NET library | Apache-2.0 |
| [zerobyte-docs](https://github.com/zerobyte-cloud/zerobyte-docs) | This documentation | CC-BY-4.0; code samples also Apache-2.0 |

The canonical specification is
[`relay-v1.md`](https://github.com/zerobyte-cloud/zerobyte-protocol/blob/master/relay-v1.md) in
zerobyte-protocol. Where this documentation and the specification disagree, the specification
wins; please [report the difference](contributing.md).

## Licence

Text under [CC-BY-4.0](LICENSE). Code samples are also available under
[Apache-2.0](LICENSE-APACHE), so you can copy them into your app under the same terms as the SDKs. See
[NOTICE](NOTICE).
