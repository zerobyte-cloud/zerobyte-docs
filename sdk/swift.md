# Swift SDK

[zerobyte-swift-sdk](https://github.com/zerobyte-cloud/zerobyte-swift-sdk). Apache-2.0.

| | |
|---|---|
| Platforms | iOS 16, iPadOS 16, macOS 13, watchOS 9, tvOS 16, visionOS 1 |
| Swift | 6, strict concurrency |
| Uses | CryptoKit (HKDF, SHA-256, ChaCha20-Poly1305), URLSession (HTTPS, WebSocket) |
| Dependencies | None |

## Install

In Xcode: File > Add Package Dependencies, enter
`https://github.com/zerobyte-cloud/zerobyte-swift-sdk.git`, and choose version 1.2.0 or later.
`ZeroByteTimeline` needs 1.2.0; the other libraries work from 1.0.0.

In `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/zerobyte-cloud/zerobyte-swift-sdk.git", from: "1.2.0")
],
targets: [
    .target(name: "MyApp", dependencies: [
        .product(name: "ZeroByte", package: "zerobyte-swift-sdk"),
        .product(name: "ZeroBytePush", package: "zerobyte-swift-sdk"),   // only with APNs
        .product(name: "ZeroByteTimeline", package: "zerobyte-swift-sdk"),   // only for a timeline
    ])
]
```

| Library | What it is |
|---|---|
| `ZeroByte` | Links, envelopes, the relay, push registration, signal and ICE |
| `ZeroBytePush` | APNs registrations and push payloads, for apps that use Apple push notifications |
| `ZeroByteTimeline` | One keyed state both devices keep, synced as commits that merge into one timeline |

## Set up a link

```swift
import ZeroByte

let relayBase = URL(string: "https://relay.example.com/v1/")!
let appID = "019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d"

let credentials = try Credentials(linkKey: linkKey, linkID: linkID)   // Data (32 bytes), UUID
let relay = Relay(appID: appID, credentials: credentials, base: relayBase)
let link = Link(
    credentials: credentials,
    sends: .primaryToCompanion,          // .companionToPrimary on the other device
    transport: relay,
    lastReceived: store.lastReceived)    // UInt64?, nil on a new link
```

`Credentials(linkKey:linkID:)` throws `ZeroByteError.invalidKey` if the key is not 32 bytes or the
id not 16. The `linkID` may be a `UUID` or 16 bytes of `Data`.

## Send

```swift
let sequence = try await link.send(Data("hello".utf8))
```

`send` is `@discardableResult` and returns the sequence the message was stored under. It throws
`ZeroByteError.tooLarge` above 1 MiB, and `RelayError` for relay failures.

## Receive

```swift
for message in try await link.receive() {
    handle(message.plaintext)          // message.sequence is the UInt64 sequence
}
store.lastReceived = await link.lastReceived
```

`receive()` returns `[(sequence: UInt64, plaintext: Data)]` in order. Save `lastReceived` after you
have handled the messages.

## Watch

```swift
func watch() async {
    var delay: Duration = .seconds(1)
    while !Task.isCancelled {
        let notifications = relay.watch(direction: .companionToPrimary)   // opens the socket
        await pull()                                    // catch up on (re)connect
        for await _ in notifications {
            delay = .seconds(1)
            await pull()
        }
        try? await Task.sleep(for: delay)               // the socket closed: back off
        delay = min(delay * 2, .seconds(60))
    }
}

func pull() async {
    do {
        for message in try await link.receive() { handle(message.plaintext) }
        store.lastReceived = await link.lastReceived
    } catch {
        // network or relay error; the next notification or reconnect retries
    }
}
```

`watch(direction:)` starts the socket and returns an `AsyncStream<UInt64>` of sequence
numbers. The stream ends when the socket closes; cancelling the consuming task closes the socket.
Watch the direction you **read**, the reverse of `sends`. The socket sends `ping` every 30 seconds,
which keeps NATs open and tells a relay with push that this device is awake.

`URLSessionWebSocketTask` does not report the moment the handshake completes, so the stream has no
"open" event. The `pull()` right after starting the watch covers almost everything; an envelope
stored in the short window before the handshake completes is picked up by the next notification.
If your app may sit idle with the socket open and nothing else being sent, also pull on a timer
while watching, for example every few minutes.

Run the watch while the app is active. On iOS, cancel the task when the scene goes to the
background and start it again on `.active`:

```swift
.onChange(of: scenePhase) { _, phase in
    if phase == .active { watchTask = Task { await watch() } }
    else { watchTask?.cancel() }
}
```

## Push

On relays whose app has `push=<topic>`. The reader registers on the direction it reads; the
writer asks for a push per message. See [Push](../protocol/extensions.md#push).

```swift
import ZeroBytePush

// The reader, after didRegisterForRemoteNotificationsWithDeviceToken:
let tag = PushRegistration.randomTag()                 // store it with the link
try await relay.registerPush(
    .apns(deviceToken: token, environment: .production, tag: tag, alert: "New message"),
    direction: .companionToPrimary)

// The writer:
try await link.send(Data("hello".utf8), notify: .alert)
```

The push carries only the tag. In the notification service extension, find the link by
`ZeroBytePushPayload.tag(in: request.content.userInfo)`, call `receive()`, and replace the alert
with the decrypted message. `unregisterPush(direction:)` stops the pushes.

## Signal and ICE

On relays whose app has `signal`: a rendezvous for setting up a direct WebRTC connection. Frames
are sealed under signal keys derived from the link key; the relay cannot read or forge them. See
[Signal](../protocol/extensions.md#signal).

```swift
let channel = relay.signal(sends: .primaryToCompanion)
let ice = try await relay.iceServers()                 // on relays with ICE configured
for await event in await channel.events() {
    switch event {
    case .peer(true): try await channel.send(offer)    // the other device is here
    case .peer(false): break                           // it left
    case .message(let data): handle(data)              // answer, candidates
    }
}
```

## Timeline

`ZeroByteTimeline` keeps one keyed state on both devices and sends only what changed, as commits
that every device merges into the same order ([Timeline v1](../protocol/timeline.md)). A key's
entry is the one with the latest stamp, so both copies converge whatever order things arrive in,
and a change made after seeing another always wins over it.

```swift
import ZeroByteTimeline

var timeline = (try? Timeline(encoded: saved)) ?? Timeline()

// Setting a key to the value it already holds stages nothing.
try timeline.set(planData, at: TimelineSlot("plan", plan.id.uuidString))

let outcome = try await TimelineRelay.sync(
    &timeline, credentials: credentials, transport: relay,
    sends: .companionToPrimary, notify: .silent
) { save($0.encoded()) }

for update in outcome.updates {
    apply(update.slot, update.value)  // nil: deleted
}
```

- `sync` reads and merges, persists, prunes, commits, answers a catch-up and writes, in that
  order. It calls the closure to persist after reading and after writing; save the timeline there.
- A run with nothing new to send writes no envelope: a read that changed nothing is not
  acknowledged, and a catch-up request with nothing to catch up on gets no answer.
- A long catch-up, for a device that is new or lost its copy, goes out over several runs of about
  8 MiB each (`TimelineRelay.catchUpBudget`, or `commitBudget:`). The timeline saves its place, so
  the next run carries on.
- `commitBudget:` also caps how much one run commits; the rest stays pending.
  `hasPendingChange(at:)` says whether a key's change is still waiting.
- `outcome.waiting` counts messages kept for the next run because the mailbox was full;
  `outcome.isPeerCurrent` says whether the other device has every commit of this one.
- A device that can produce its own values again (from a database, say) creates its timeline with
  `forgetsOwnValues:` naming those collections and passes `values:` to `sync`; the timeline then
  keeps only their digests. `values:` must return the value as committed: a catch-up leaves out a
  key whose value has changed since, until the change is set and committed.
- Two devices on the same network can skip the relay: send `timeline.state()`, answer the other's
  state with `timeline.catchUp(after:)` and `receive` each part. Pass commits made there with
  `commit(now:)` to `TimelineRelay.enqueue(_:in:)` as well: the relay carries them unless the
  other device confirms them first.
- A value must fit in one message (1 MiB); split larger values over keys.

## Unpair

```swift
try await link.erase()
try keychain.deleteLink()
store.lastReceived = nil
```

## Store the link key

Keep the link key in the Keychain as a generic password, available after first unlock, on this
device only, not synchronised:

```swift
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrService as String: "com.example.app.zerobyte",
    kSecAttrAccount as String: linkID.uuidString,
    kSecValueData as String: linkKey,
    kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
    kSecAttrSynchronizable as String: false,
]
let status = SecItemAdd(query as CFDictionary, nil)
```

`ThisDeviceOnly` keeps the key out of iCloud Keychain and out of backups restored to another
device. See [Storing the link key](../guides/key-storage.md).

## API reference

### `Credentials`

```swift
public struct Credentials: Sendable, Hashable {
    public init(linkKey: Data, linkID: Data) throws
    public init(linkKey: Data, linkID: UUID) throws
    public let mailboxID: Data            // 16 bytes
    public let accessToken: Data          // 32 bytes
    public var mailboxName: String        // 32 lowercase hex characters
    public var bearerToken: String        // 43 characters, base64url
    public var tokenDigest: String        // 64 lowercase hex characters
}
```

### `Direction`

```swift
public enum Direction: UInt8 {
    case primaryToCompanion = 1
    case companionToPrimary = 2
    public var pathComponent: String      // "p2c" or "c2p"
    public var reversed: Direction
}
```

### `Link`

```swift
public actor Link {
    public init(credentials: Credentials, sends: Direction, transport: any RelayTransport,
                lastReceived: UInt64? = nil)
    public let credentials: Credentials
    public let sends: Direction
    public private(set) var lastReceived: UInt64?
    @discardableResult public func send(_ plaintext: Data, notify: Notify = .none) async throws -> UInt64
    public func receive() async throws -> [(sequence: UInt64, plaintext: Data)]
    public func erase() async throws
}
```

### `Relay`

```swift
public struct Relay: RelayTransport {
    public static let fetchLimit = 16
    public init(appID: String, credentials: Credentials, base: URL)
    public init(endpoint: URL, credentials: Credentials)   // full https://host/v1/<app id>/
    public static let keepAliveInterval: Duration          // 30 seconds
    public let endpoint: URL
    public func watch(direction: Direction) -> AsyncStream<UInt64>
    public func put(_ envelope: Data, direction: Direction, sequence: UInt64,
                    notify: Notify) async throws
    public func registerPush(_ registration: PushRegistration, direction: Direction) async throws
    public func unregisterPush(direction: Direction) async throws
    public func iceServers() async throws -> ICEConfiguration
    public func signal(sends direction: Direction) -> SignalChannel
    // RelayTransport methods below
}
```

### Extensions

```swift
public enum Notify: String { case none, alert, silent }

public struct PushRegistration {
    public init(service: String, fields: [String: String], tag: String, alert: String? = nil)
    public static func randomTag() -> String
    public var json: Data
}

public struct ICEConfiguration: Decodable { public let iceServers: [ICEServer]; public let ttl: Int }
public struct ICEServer: Decodable { public let urls: [String]; public let username, credential: String? }

public actor SignalChannel {
    public enum Event { case peer(Bool), message(Data) }
    public enum Failure: Error { case notConnected, closed(code: Int) }
    public func events() -> AsyncStream<Event>
    public func send(_ plaintext: Data) async throws
    public func close()
}

public enum SignalFrame {
    public static let maximumFrameBytes: Int          // 65_536
    public static let window: UInt64                  // 300_000 ms
    // seal / open, like Envelope, with a counter in place of the sequence
}

// ZeroBytePush
public enum APNsEnvironment: String { case production, development }
extension PushRegistration {
    public static func apns(deviceToken: Data, environment: APNsEnvironment, tag: String,
                            alert: String? = nil) -> PushRegistration
}
public enum ZeroBytePushPayload { public static func tag(in userInfo: [AnyHashable: Any]) -> String? }
```

`Relay` uses an ephemeral `URLSession` with no cookies, no cache and no credential store, a 30
second request timeout, and `waitsForConnectivity = false`, so a request without a network fails
fast instead of waiting.

### `RelayTransport`

```swift
public protocol RelayTransport: Sendable {
    func claim() async throws
    func put(_ envelope: Data, direction: Direction, sequence: UInt64) async throws
    func fetch(direction: Direction, after sequence: UInt64?) async throws
        -> (envelopes: [Data], more: Bool)
    func lastSequence(direction: Direction) async throws -> UInt64?
    func prune(direction: Direction, through sequence: UInt64) async throws
    func erase() async throws
}
```

### `Envelope` and `Wire`

```swift
public enum Envelope {
    public static let version: UInt8                  // 1
    public static let overhead: Int                   // 38
    public static let maximumPlaintextBytes: Int      // 1_048_576
    public static let maximumEnvelopeBytes: Int
    public static func seal(_ plaintext: Data, direction: Direction, sequence: UInt64,
                            credentials: Credentials) throws -> Data
    public static func open(_ envelope: Data, direction: Direction, after lastSequence: UInt64?,
                            credentials: Credentials) throws -> (sequence: UInt64, plaintext: Data)
    public static func header(of envelope: Data) throws -> (direction: Direction, sequence: UInt64)
}

public enum Wire {
    public static let maximumCount = 64
    public static func join(_ envelopes: [Data]) -> Data
    public static func split(_ data: Data) throws -> [Data]
}
```

### Timeline

```swift
// ZeroByteTimeline
public struct TimelineSlot: Hashable, Comparable {
    public init(_ collection: String, _ key: String)
    public let collection: String, key: String
}

public struct Timeline {
    public init(replica: ReplicaID = .random(), forgetsOwnValues: Set<String> = [],
                maximumMessageBytes: Int = TimelineLimits.maximumMessageBytes)
    public init(encoded data: Data) throws
    public func encoded() -> Data
    public func value(_ slot: TimelineSlot) -> Data?
    public func entries(in collection: String) -> [TimelineEntry]
    @discardableResult public mutating func set(_ value: Data, at slot: TimelineSlot) throws -> Bool
    @discardableResult public mutating func delete(_ slot: TimelineSlot) throws -> Bool
    public var hasPendingChanges: Bool
    public func hasPendingChange(at slot: TimelineSlot) -> Bool
    public mutating func commit(now: Date, maximumBytes: Int? = nil) -> [TimelineCommit]
    public mutating func receive(_ data: Data) throws -> TimelineReceipt
    public func state(request: TimelineRequest = []) -> TimelineMessage
    public func catchUp(after requesterHeads: [ReplicaID: TimelineHead], includingOwn: Bool = true,
                        values: (TimelineSlot) -> Data? = { _ in nil }) -> [TimelineMessage]
}

public enum TimelineRelay {
    public static let catchUpBudget = 8 * 1_048_576     // catch-up bytes per run
    public static func sync(_ timeline: inout Timeline, credentials: Credentials,
                            transport: any RelayTransport, sends: Direction, now: Date = Date(),
                            commitBudget: Int? = nil, notify: Notify = .none,
                            values: (TimelineSlot) -> Data? = { _ in nil },
                            persist: (Timeline) async throws -> Void) async throws
        -> TimelineSyncOutcome            // updates, merged, written, waiting, isPeerCurrent
    public static func enqueue(_ commits: [TimelineCommit], in timeline: inout Timeline)
}

public enum TimelineLimits {
    public static let maximumMessageBytes = 1_048_576
    public static let maximumCollectionBytes = 64
    public static let maximumKeyBytes = 1_024
}

public enum TimelineError: Error { case malformed, unsupportedVersion, tooLarge, diverged }
```

### Errors

```swift
public enum ZeroByteError: Error {
    case invalidKey, malformedEnvelope, unsupportedVersion(UInt8), wrongDirection,
         replayed(sequence: UInt64), authenticationFailed, tooLarge
}

public enum RelayError: Error {
    case unauthorized, sequenceConflict(last: UInt64?), quotaExceeded,
         rateLimited(retryAfter: TimeInterval?), unavailable(status: Int), malformedResponse
}
```

## Testing

```sh
swift test
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=<app id> swift test   # also against a live relay
```

Set `RELAY_SIGNAL_APP` to an app with `signal` and `push=<topic>` to test the extensions too.

In your own tests, implement `RelayTransport` in memory and pass it to two `Link`s. See
[Testing your integration](../guides/testing.md).
