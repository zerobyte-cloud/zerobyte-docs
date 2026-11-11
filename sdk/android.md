# Android and JVM SDK

[zerobyte-android-sdk](https://github.com/zerobyte-cloud/zerobyte-android-sdk). Apache-2.0.

| | |
|---|---|
| Platforms | Android API 28 and newer; any JVM 11 and newer |
| Language | Kotlin, usable from Java |
| Uses | `javax.crypto` (ChaCha20-Poly1305, HmacSHA256), `java.security.MessageDigest`, `HttpURLConnection` |
| Dependencies | Kotlin standard library only; the optional `zerobyte-okhttp` adds OkHttp for the WebSockets |
| Coordinates | `cloud.zerobyte:zerobyte:1.2.0`, `cloud.zerobyte:zerobyte-okhttp:1.2.0`, `cloud.zerobyte:zerobyte-timeline:1.2.0` |

## Install

The library is not on Maven Central yet. Publish it to your local Maven repository:

```sh
git clone --branch v1.2.0 https://github.com/zerobyte-cloud/zerobyte-android-sdk.git
cd zerobyte-android-sdk
./gradlew publishToMavenLocal
```

Then in your app's Gradle files:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        mavenLocal()
    }
}

// app/build.gradle.kts
dependencies {
    implementation("cloud.zerobyte:zerobyte:1.2.0")
    implementation("cloud.zerobyte:zerobyte-okhttp:1.2.0")     // watch and signal sockets
    implementation("cloud.zerobyte:zerobyte-timeline:1.2.0")   // only for a timeline
}
```

For a team or CI, publish the artefact to your own Maven repository instead of `mavenLocal()`, or
include the `zerobyte` module as a Gradle included build.

The app needs the `INTERNET` permission:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

## Set up a link

```kotlin
import cloud.zerobyte.*

const val RELAY_BASE = "https://relay.example.com/v1/"
const val APP_ID = "019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d"

val credentials = Credentials(linkKey, linkID)          // ByteArray(32), java.util.UUID
val relay = Relay(APP_ID, credentials, RELAY_BASE)
val link = Link(credentials, Direction.C2P, relay, store.lastReceived)   // ULong?
```

`Credentials` throws `ZeroByteException("invalid_key")` if the key is not 32 bytes or the id not
16. The link id may be a `UUID` or a 16-byte `ByteArray`.

## Blocking calls

Every network call is **blocking**. On Android, call them off the main thread, for example on
`Dispatchers.IO`:

```kotlin
suspend fun send(text: String): ULong = withContext(Dispatchers.IO) {
    link.send(text.toByteArray())
}
```

`Link` is thread-safe: concurrent calls run one at a time.

## Send

```kotlin
val sequence: ULong = link.send(bytes)
```

Throws `ZeroByteException("too_large")` above 1 MiB and `RelayException` for relay failures.
Network failures surface as `IOException`; `RelayException` is a subclass of `IOException`.

## Receive

```kotlin
suspend fun pull() = withContext(Dispatchers.IO) {
    for (message in link.receive()) handle(message.sequence, message.plaintext)
    store.lastReceived = link.lastReceived
}
```

`receive()` returns `List<Opened>` in order; `Opened` has `sequence: ULong` and
`plaintext: ByteArray`.

## Watch

Android has no platform WebSocket client. `zerobyte-okhttp` opens the watch socket with OkHttp,
sends the bearer and the `zerobyte.v1` subprotocol, and sends `ping` every 30 seconds:

```kotlin
import cloud.zerobyte.okhttp.*

val watch = relay.watch(okHttp, Direction.P2C, object : WatchListener {   // the direction you read
    override fun onOpen() { scope.launch { pull() } }
    override fun onSequence(sequence: ULong) { scope.launch { pull() } }
    override fun onClosed(code: Int, reason: String) {
        if (reason == "erased") onUnpairedRemotely() else scheduleReconnect()
    }
    override fun onFailure(error: Throwable) { scheduleReconnect() }
})
```

Close it in `onStop` (`watch.close()`) and open it again in `onStart`: the `onOpen` pull catches
up. Use exponential backoff in `scheduleReconnect`. The `ping` keeps NATs open and tells a relay
with push that this device is awake, so it sends no push while the socket is open.

With another WebSocket client, open `relay.watchURL(direction)` with the header
`Authorization: relay.authorization()` and the subprotocol `zerobyte.v1`, treat every text message
that is a number as a sequence, and send `ping` every `Relay.KEEP_ALIVE_MILLISECONDS`. Clients
that cannot set headers may offer the subprotocol `zerobyte.bearer.<bearer token>` instead.

## Push

On relays whose app has `push=<topic>` and whose operator configured FCM. The reader registers on
the direction it reads, after every launch and whenever Firebase hands out a new token; the
writer asks for a push per message. See [Push](../protocol/extensions.md#push).

```kotlin
val tag = PushRegistration.randomTag()                    // store it with the link
val token = FirebaseMessaging.getInstance().token.await()
relay.registerPush(Fcm.registration(token, tag, "New data"), Direction.P2C)
link.send(bytes, Notify.ALERT)                            // on the other device
```

Every ZeroByte FCM message is data-only, so `onMessageReceived` runs in the foreground and in the
background alike. Find the link by the tag, receive, and draw the notification yourself:

```kotlin
class Messages : FirebaseMessagingService() {
    override fun onMessageReceived(message: RemoteMessage) {
        val push = Fcm.parse(message.data) ?: return
        val link = links.byTag(push.tag) ?: return
        val opened = runCatching { link.receive() }.getOrDefault(emptyList())
        if (push.notify == Notify.ALERT) notify(opened, fallback = push.alert)
    }

    override fun onNewToken(token: String) = registerAgain(token)
}
```

A silent push wakes the app for a sync without a notification. For other push services, build a
`PushRegistration("<service>", mapOf(/* its fields */), tag)`.

## Signal and ICE

On relays whose app has `signal`: a rendezvous for setting up a direct WebRTC connection. Frames
are sealed under signal keys derived from the link key; the relay cannot read or forge them. See
[Signal](../protocol/extensions.md#signal).

```kotlin
val signal = relay.signal(okHttp, Direction.P2C, object : SignalListener {
    override fun onPeer(present: Boolean) { if (present) signal.send(offer) }
    override fun onMessage(data: ByteArray) = handle(data)   // answer, candidates
})
val ice = withContext(Dispatchers.IO) { relay.iceServers() }   // STUN/TURN for PeerConnection
```

Without `zerobyte-okhttp`, open `relay.signalURL(sends)` like the watch socket and route every
message through `relay.signalSession(sends)` (one per connection): `onText`, `onBinary` and
`seal`. Close code 4000 means a newer connection of the same side replaced this one; 1003 or 1009
a bad frame; 1008 rate limited.

## Timeline

`zerobyte-timeline` keeps one keyed state on both devices and sends only what changed, as commits
that every device merges into the same order ([Timeline v1](../protocol/timeline.md)). A key's
entry is the one with the latest stamp, so both copies converge whatever order things arrive in,
and a change made after seeing another always wins over it.

```kotlin
import cloud.zerobyte.timeline.*

val timeline = saved?.let(Timeline::decode) ?: Timeline()

// Setting a key to the value it already holds stages nothing.
timeline.set(TimelineSlot("plan", plan.id), planBytes)

val outcome = withContext(Dispatchers.IO) {        // blocking, like the relay calls
    TimelineRelay.sync(
        timeline, credentials, relay, Direction.C2P,
        notify = Notify.SILENT,
    ) { save(it.encoded()) }
}

for (update in outcome.updates) {
    apply(update.slot, update.value)  // null: deleted
}
```

- `sync` reads and merges, persists, prunes, commits, answers a catch-up and writes, in that
  order. It calls the closure to persist after reading and after writing; save the timeline there.
- A run with nothing new to send writes no envelope: a read that changed nothing is not
  acknowledged, and a catch-up request with nothing to catch up on gets no answer.
- A long catch-up, for a device that is new or lost its copy, goes out over several runs of about
  8 MiB each (`TimelineRelay.CATCH_UP_BUDGET`, or `commitBudget`). The timeline saves its place,
  so the next run carries on.
- `commitBudget` also caps how much one run commits; the rest stays pending.
  `hasPendingChange(slot)` says whether a key's change is still waiting.
- `outcome.waiting` counts messages kept for the next run because the mailbox was full;
  `outcome.isPeerCurrent` says whether the other device has every commit of this one.
- A device that can produce its own values again (from a database, say) creates its timeline with
  `forgetsOwnValues` naming those collections and passes `values` to `sync`; the timeline then
  keeps only their digests. `values` must return the value as committed: a catch-up leaves out a
  key whose value has changed since, until the change is set and committed.
- Two devices on the same network can skip the relay: send `timeline.state().encoded()`, answer
  the other's state with `timeline.catchUp(heads)` and `receive` each part. Pass commits made
  there with `commit()` to `TimelineRelay.enqueue` as well: the relay carries them unless the
  other device confirms them first.
- A value must fit in one message (1 MiB); split larger values over keys.
- A `Timeline` is not thread-safe: use it from one thread at a time.

## Unpair

```kotlin
withContext(Dispatchers.IO) { link.erase() }
keyStore.deleteLink()
store.lastReceived = null
```

## Store the link key

Android Keystore cannot hold an arbitrary 32-byte secret for export. Wrap the link key with a
Keystore AES key and store the wrapped bytes:

```kotlin
val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
val alias = "zerobyte-link-wrap"
if (!keyStore.containsAlias(alias)) {
    KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore").apply {
        init(
            KeyGenParameterSpec.Builder(alias,
                KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .build())
    }.generateKey()
}
val wrapKey = keyStore.getKey(alias, null) as SecretKey

fun wrap(linkKey: ByteArray): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.ENCRYPT_MODE, wrapKey)
    return cipher.iv + cipher.doFinal(linkKey)                // 12-byte IV, then ciphertext
}

fun unwrap(stored: ByteArray): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, wrapKey, GCMParameterSpec(128, stored, 0, 12))
    return cipher.doFinal(stored, 12, stored.size - 12)
}
```

Store the wrapped bytes in app-private storage and exclude them from cloud backup
(`android:dataExtractionRules`). See [Storing the link key](../guides/key-storage.md).

## API reference

### `Credentials`

```kotlin
class Credentials(linkKey: ByteArray, linkID: ByteArray) {
    constructor(linkKey: ByteArray, linkID: UUID)
    val mailboxID: ByteArray          // 16 bytes
    val accessToken: ByteArray        // 32 bytes
    fun signalKey(direction: Direction): ByteArray   // 32 bytes, for signal frames
    val mailboxName: String           // 32 lowercase hex characters
    val bearerToken: String           // 43 characters, base64url
    val tokenDigest: String           // 64 lowercase hex characters
    companion object { fun uuidBytes(uuid: UUID): ByteArray }
}
```

### `Direction`

```kotlin
enum class Direction(val wire: Byte, val path: String) {
    P2C(1, "p2c"), C2P(2, "c2p");
    val reversed: Direction
    companion object { fun of(wire: Byte): Direction? }
}
```

### `Link`

```kotlin
class Link @JvmOverloads constructor(
    credentials: Credentials, val sends: Direction, relay: RelayTransport,
    lastReceived: ULong? = null,
) {
    val receives: Direction
    var lastReceived: ULong?              // read-only from outside
    fun send(plaintext: ByteArray, notify: Notify = Notify.NONE): ULong
    fun receive(): List<Opened>
    fun erase()
}

class Opened(val sequence: ULong, val plaintext: ByteArray)
```

### `Relay`

```kotlin
class Relay @JvmOverloads constructor(
    appID: String, credentials: Credentials,
    base: String, timeoutMillis: Int = 30_000,
) : RelayTransport {
    val mailbox: String
    fun watchURL(direction: Direction): String
    fun signalURL(sends: Direction): String
    fun signalSession(sends: Direction): SignalSession
    fun authorization(): String           // "Bearer <token>", for a WebSocket upgrade
    fun put(envelope: ByteArray, direction: Direction, sequence: ULong, notify: Notify)
    fun registerPush(registration: PushRegistration, direction: Direction)
    fun unregisterPush(direction: Direction)
    fun iceServers(): IceConfiguration
    companion object { const val KEEP_ALIVE_MILLISECONDS = 30_000L }
}
```

### Extensions

```kotlin
enum class Notify { NONE, ALERT, SILENT }

class PushRegistration @JvmOverloads constructor(
    val service: String, val fields: Map<String, String>, val tag: String, val alert: String? = null,
) {
    fun toJson(): String
    companion object { fun randomTag(): String }
}

object Fcm {
    fun registration(token: String, tag: String, alert: String? = null): PushRegistration
    fun parse(data: Map<String, String>): FcmPush?
}
data class FcmPush(val tag: String, val notify: Notify, val alert: String?)

class IceServer(val urls: List<String>, val username: String?, val credential: String?)
class IceConfiguration(val iceServers: List<IceServer>, val ttl: Int)

class SignalSession(credentials: Credentials, val sends: Direction) {
    fun onText(text: String): SignalEvent?
    fun onBinary(frame: ByteArray, now: Long = System.currentTimeMillis()): SignalEvent?
    fun seal(plaintext: ByteArray): ByteArray
}

sealed class SignalEvent {
    data class Peer(val present: Boolean) : SignalEvent()
    class Message(val data: ByteArray) : SignalEvent()
}

object SignalFrame {
    const val MAX_FRAME_BYTES = 65_536
    const val WINDOW_MILLISECONDS = 300_000L
    fun seal(plaintext: ByteArray, direction: Direction, counter: ULong, credentials: Credentials): ByteArray
    fun open(frame: ByteArray, direction: Direction, after: ULong?, now: ULong,
             credentials: Credentials): Opened
}
```

### `zerobyte-okhttp`

```kotlin
package cloud.zerobyte.okhttp

fun Relay.watch(client: OkHttpClient, direction: Direction, listener: WatchListener): Watch
fun Relay.signal(client: OkHttpClient, sends: Direction, listener: SignalListener): SignalSocket

interface WatchListener {
    fun onOpen() {}
    fun onSequence(sequence: ULong)
    fun onClosed(code: Int, reason: String) {}
    fun onFailure(error: Throwable) {}
}
interface SignalListener {
    fun onPeer(present: Boolean)
    fun onMessage(data: ByteArray)
    fun onClosed(code: Int, reason: String) {}
    fun onFailure(error: Throwable) {}
}
class Watch : Closeable
class SignalSocket : Closeable { fun send(plaintext: ByteArray): Boolean }
```

Callbacks run on OkHttp's threads.

### `zerobyte-timeline`

```kotlin
package cloud.zerobyte.timeline

class TimelineSlot(val collection: String, val key: String) : Comparable<TimelineSlot>

class Timeline @JvmOverloads constructor(
    val replica: ReplicaID = ReplicaID.random(),
    forgetsOwnValues: Set<String> = emptySet(),
    maximumMessageBytes: Int = TimelineLimits.MAX_MESSAGE_BYTES,
) {
    fun encoded(): ByteArray
    fun value(slot: TimelineSlot): ByteArray?
    fun entries(collection: String): List<TimelineEntry>
    fun set(slot: TimelineSlot, value: ByteArray): Boolean
    fun delete(slot: TimelineSlot): Boolean
    val hasPendingChanges: Boolean
    fun hasPendingChange(slot: TimelineSlot): Boolean
    fun commit(now: Long = System.currentTimeMillis(), maximumBytes: Int? = null): List<TimelineCommit>
    fun receive(data: ByteArray): TimelineReceipt
    fun state(request: TimelineRequest = TimelineRequest.NONE): TimelineMessage.State
    fun catchUp(requesterHeads: Map<ReplicaID, TimelineHead>, includingOwn: Boolean = true,
                values: (TimelineSlot) -> ByteArray? = { null }): List<TimelineMessage.CatchUp>
    companion object { fun decode(data: ByteArray): Timeline }
}

object TimelineRelay {
    const val CATCH_UP_BUDGET: Long = 8L * 1_048_576     // catch-up bytes per run
    fun sync(timeline: Timeline, credentials: Credentials, transport: RelayTransport,
             sends: Direction, now: Long = System.currentTimeMillis(), commitBudget: Int? = null,
             notify: Notify = Notify.NONE, values: (TimelineSlot) -> ByteArray? = { null },
             persist: (Timeline) -> Unit): TimelineSyncOutcome
        // updates, merged, written, waiting, isPeerCurrent
    fun enqueue(commits: List<TimelineCommit>, timeline: Timeline)
}

object TimelineLimits {
    const val MAX_MESSAGE_BYTES = 1_048_576
    const val MAX_COLLECTION_BYTES = 64
    const val MAX_KEY_BYTES = 1_024
}

class TimelineException(val code: String) : Exception   // malformed, unsupported_version,
                                                        // too_large, diverged
```

### `RelayTransport`

```kotlin
interface RelayTransport {
    fun claim()
    fun put(envelope: ByteArray, direction: Direction, sequence: ULong)
    fun fetch(direction: Direction, after: ULong?): Page
    fun lastSequence(direction: Direction): ULong?
    fun prune(direction: Direction, through: ULong)
    fun erase()
}

class Page(val envelopes: List<ByteArray>, val more: Boolean)
```

### `Envelope` and `Wire`

```kotlin
object Envelope {
    const val OVERHEAD = 38
    const val MAX_PLAINTEXT_BYTES = 1_048_576
    const val MAX_ENVELOPE_BYTES = MAX_PLAINTEXT_BYTES + OVERHEAD
    fun seal(plaintext: ByteArray, direction: Direction, sequence: ULong,
             credentials: Credentials): ByteArray
    fun open(envelope: ByteArray, direction: Direction, after: ULong?,
             credentials: Credentials): Opened
    fun readHeader(envelope: ByteArray): Pair<Direction, ULong>
}

object Wire {
    const val MAXIMUM_COUNT = 64
    fun join(envelopes: List<ByteArray>): ByteArray
    fun split(body: ByteArray): List<ByteArray>
}
```

`seal` has an optional `nonce` parameter for known-answer tests only. Leave it out.

### Errors

```kotlin
class ZeroByteException(val code: String) : Exception
class RelayException(
    val code: String, val status: Int, val last: ULong? = null,
    val retryAfterSeconds: Long? = null,      // on rate_limited
) : IOException
```

## From Java

The API uses Kotlin's unsigned `ULong` for sequence numbers. Kotlin hides or renames functions and
constructors that take unsigned types as parameters when seen from Java, so parts of `Link` and
`Envelope` are awkward or unavailable there. Call the SDK from Kotlin code in your project, and
expose to Java whatever small surface your app needs.

## Testing

```sh
./gradlew test
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=<app id> ./gradlew test
```

The tests run on the JVM without an emulator. Set `RELAY_SIGNAL_APP` to an app with `signal` and
`push=<topic>` to test the extensions too.
