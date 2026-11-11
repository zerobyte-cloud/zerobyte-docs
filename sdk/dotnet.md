# .NET SDK

[zerobyte-dotnet-sdk](https://github.com/zerobyte-cloud/zerobyte-dotnet-sdk). Apache-2.0. Package
ids `ZeroByte` and, for apps that use push notifications, `ZeroByte.Apns` (Apple) and
`ZeroByte.Fcm` (Android); `ZeroByte.Timeline` for a [timeline](#timeline).

| | |
|---|---|
| Target | .NET 10 (`net10.0`): WinUI, WPF, Windows Forms, MAUI, ASP.NET Core, worker services, console |
| Uses | `System.Security.Cryptography` (HKDF, SHA-256, ChaCha20-Poly1305), `HttpClient`, `ClientWebSocket` |
| Dependencies | None |

## Install

The package is not on NuGet yet. Reference the project directly, or build a package and add it
from a local folder:

```sh
git clone --branch v1.2.0 https://github.com/zerobyte-cloud/zerobyte-dotnet-sdk.git
dotnet pack zerobyte-dotnet-sdk/src/ZeroByte -o ./packages
dotnet add package ZeroByte --source ./packages
```

Or, as a project reference:

```sh
dotnet add reference ../zerobyte-dotnet-sdk/src/ZeroByte/ZeroByte.csproj
dotnet add reference ../zerobyte-dotnet-sdk/src/ZeroByte.Apns/ZeroByte.Apns.csproj   # only with APNs
dotnet add reference ../zerobyte-dotnet-sdk/src/ZeroByte.Fcm/ZeroByte.Fcm.csproj     # only with FCM
dotnet add reference ../zerobyte-dotnet-sdk/src/ZeroByte.Timeline/ZeroByte.Timeline.csproj   # only for a timeline
```

## Platform support

ChaCha20-Poly1305 comes from the operating system. It is available on Windows 10 20H2 and newer,
on Linux with OpenSSL 1.1 or newer, and on macOS. Check before use:

```csharp
if (!Envelope.IsSupported)
    throw new PlatformNotSupportedException("ChaCha20-Poly1305 is not available on this system.");
```

Every `Envelope` call throws on a system without it.

## Set up a link

```csharp
using ZeroByte;

var relayBase = new Uri("https://relay.example.com/v1/");
const string AppId = "019a0d4e-6b1c-7f3a-9c2d-5e8f7a6b4c3d";

var credentials = new Credentials(linkKey, linkId);              // byte[32], Guid
var relay = new Relay(AppId, credentials, relayBase);
var link = new Link(credentials, Direction.C2P, relay, store.LastReceived);   // ulong?
```

`Credentials` throws `ZeroByteException` with `Code == "invalid_key"` if the key is not 32 bytes or
the id not 16. A `Guid` is read in RFC 4122 byte order (`ToByteArray(bigEndian: true)`), which
matches the UUID's text form and the other SDKs. Passing `guid.ToByteArray()` as bytes instead
would produce a different link id.

## Send

```csharp
ulong sequence = await link.SendAsync("hello"u8.ToArray());
```

Throws `ZeroByteException("too_large")` above 1 MiB and `RelayException` for relay failures.
Network failures surface as `HttpRequestException`.

## Receive

```csharp
async Task PullAsync(CancellationToken ct)
{
    foreach (var message in await link.ReceiveAsync(ct))
        Handle(message.Sequence, message.Plaintext);
    store.LastReceived = link.LastReceived;
}
```

`ReceiveAsync` returns `List<Opened>`; `Opened` is a `record struct` with `Sequence` and
`Plaintext`.

## Watch

```csharp
async Task WatchAsync(CancellationToken ct)
{
    var delay = TimeSpan.FromSeconds(1);
    while (!ct.IsCancellationRequested)
    {
        try
        {
            await relay.WatchAsync(
                Direction.P2C,                                   // the direction you read
                onSequence: _ => _ = PullAsync(ct),
                onOpen: () => { delay = TimeSpan.FromSeconds(1); _ = PullAsync(ct); },
                ct: ct);
        }
        catch (Exception) when (!ct.IsCancellationRequested) { }
        await Task.Delay(delay, ct);
        delay = TimeSpan.FromSeconds(Math.Min(delay.TotalSeconds * 2, 60));
    }
}
```

`WatchAsync` connects, calls `onOpen` once the socket is open, calls `onSequence` for each
announced sequence, and returns when the socket closes or `ct` is cancelled. It authenticates with
the `Authorization` header and offers `zerobyte.v1`.

`Link` runs one operation at a time, so overlapping `PullAsync` calls are safe. In a UI app,
marshal `Handle` back to the UI thread. The socket sends `ping` every 30 seconds, which keeps NATs
open and tells a relay with push that this device is awake.

## Push

On relays whose app has `push=<topic>`. The reader registers on the direction it reads; the
writer asks for a push per message. See [Push](../protocol/extensions.md#push).

```csharp
using ZeroByte.Apns;

var tag = PushRegistration.RandomTag();                     // store it with the link
await relay.RegisterPushAsync(
    Apns.Registration(deviceToken, ApnsEnvironment.Production, tag, alert: "New message"),
    Direction.P2C);

await link.SendAsync(bytes, Notify.Alert);                  // on the other device
```

The push carries only `{"zerobyte":{"tag":"…"}}`; `Apns.TagFromPayload(json)` reads it. For
another push module the relay has, pass `new PushRegistration(service, fields, tag)`.

On Android (.NET MAUI with Firebase), register the FCM token instead, and read the data-only
message in your messaging service:

```csharp
using ZeroByte.Fcm;

await relay.RegisterPushAsync(Fcm.Registration(fcmToken, tag, alert: "New data"), Direction.P2C);

if (Fcm.Parse(message.Data) is { } push)                   // IReadOnlyDictionary<string, string>
{
    // find the link by push.Tag, receive, and for Notify.Alert show a notification,
    // falling back to push.Alert
}
```

## Signal and ICE

On relays whose app has `signal`: a rendezvous for setting up a direct WebRTC connection. Frames
are sealed under signal keys derived from the link key; the relay cannot read or forge them. See
[Signal](../protocol/extensions.md#signal).

```csharp
var ice = await relay.IceServersAsync(ct);                  // on relays with ICE configured
await using var channel = relay.Signal(Direction.P2C);
await foreach (var e in channel.EventsAsync(ct))
{
    switch (e)
    {
        case SignalEvent.Peer { Present: true }: await channel.SendAsync(offer, ct); break;
        case SignalEvent.Message m: await HandleAsync(m.Data); break;    // answer, candidates
    }
}
// channel.CloseCode: 4000 replaced by a newer connection, 1003/1009 bad frame, 1008 rate limited
```

## Timeline

`ZeroByte.Timeline` keeps one keyed state on both devices and sends only what changed, as commits
that every device merges into the same order ([Timeline v1](../protocol/timeline.md)). A key's
entry is the one with the latest stamp, so both copies converge whatever order things arrive in,
and a change made after seeing another always wins over it.

```csharp
using ZeroByte.Timeline;

var timeline = File.Exists(path) ? Timeline.Decode(File.ReadAllBytes(path)) : new Timeline();

// Setting a key to the value it already holds stages nothing.
timeline.Set(new TimelineSlot("plan", plan.Id), planBytes);

var outcome = await TimelineRelay.SyncAsync(
    timeline, credentials, relay, Direction.C2P,
    persist: t => File.WriteAllBytesAsync(path, t.Encode()),
    notify: Notify.Silent);

foreach (var update in outcome.Updates)
    Apply(update.Slot, update.Value);                       // null: deleted
```

- `SyncAsync` reads and merges, persists, prunes, commits, answers a catch-up and writes, in that
  order. It calls `persist` after reading and after writing; save the timeline there.
- A device that can produce its own values again (from a database, say) creates its timeline with
  `forgetsOwnValues` naming those collections and passes `values` to `SyncAsync`; the timeline
  then keeps only their digests.
- Two devices on the same network can skip the relay: send `timeline.State().Encode()`, answer the
  other's state with `timeline.CatchUp(heads)` and `Receive` each part.
- A value must fit in one message (1 MiB); split larger values over keys.
- A `Timeline` is not thread-safe: run one `SyncAsync` per timeline at a time.

## Unpair

```csharp
await link.EraseAsync();
DeleteStoredLink();
store.LastReceived = null;
```

## Store the link key

On Windows, protect the link key with DPAPI for the current user:

```csharp
using System.Security.Cryptography;

byte[] protectedKey = ProtectedData.Protect(linkKey, optionalEntropy: null,
    DataProtectionScope.CurrentUser);
File.WriteAllBytes(path, protectedKey);

byte[] linkKey = ProtectedData.Unprotect(File.ReadAllBytes(path), null,
    DataProtectionScope.CurrentUser);
```

`ProtectedData` is in the `System.Security.Cryptography.ProtectedData` package. On macOS use the
Keychain, on Linux the Secret Service (libsecret). See [Storing the link key](../guides/key-storage.md).

## Sharing an `HttpClient`

`Relay` creates its own `HttpClient` with redirects and cookies disabled. To share one, for example
from `IHttpClientFactory`, pass it in and configure it the same way:

```csharp
services.AddHttpClient("zerobyte")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        AllowAutoRedirect = false,
        UseCookies = false,
    });

var relay = new Relay(AppId, credentials, relayBase, factory.CreateClient("zerobyte"));
```

## API reference

```csharp
public enum Direction : byte { P2C = 1, C2P = 2 }
public static class DirectionExtensions { public static Direction Reversed(this Direction d); }

public sealed class Credentials
{
    public Credentials(ReadOnlySpan<byte> linkKey, ReadOnlySpan<byte> linkID);
    public Credentials(ReadOnlySpan<byte> linkKey, Guid linkID);
    public byte[] MailboxID { get; }          // 16 bytes
    public byte[] AccessToken { get; }        // 32 bytes
    public byte[] SignalKey(Direction d);     // 32 bytes, for signal frames
    public string MailboxName { get; }        // 32 lowercase hex characters
    public string BearerToken { get; }        // 43 characters, base64url
    public string TokenDigest { get; }        // 64 lowercase hex characters
}

public sealed class Link
{
    public Link(Credentials credentials, Direction sends, IRelayTransport relay,
                ulong? lastReceived = null);
    public Direction Sends { get; }
    public Direction Receives { get; }
    public ulong? LastReceived { get; }
    public Task<ulong> SendAsync(byte[] plaintext, CancellationToken ct = default);
    public Task<ulong> SendAsync(byte[] plaintext, Notify notify, CancellationToken ct = default);
    public Task<List<Opened>> ReceiveAsync(CancellationToken ct = default);
    public Task EraseAsync(CancellationToken ct = default);
}

public readonly record struct Opened(ulong Sequence, byte[] Plaintext);

public sealed class Relay : IRelayTransport
{
    public Relay(string appID, Credentials credentials, Uri @base, HttpClient? http = null);
    public Uri Mailbox { get; }
    public static readonly TimeSpan KeepAliveInterval;      // 30 seconds
    public Task WatchAsync(Direction direction, Action<ulong> onSequence, Action? onOpen = null,
                           CancellationToken ct = default);
    public Task PutAsync(byte[] envelope, Direction direction, ulong sequence, Notify notify,
                         CancellationToken ct = default);
    public Task RegisterPushAsync(PushRegistration registration, Direction direction,
                                  CancellationToken ct = default);
    public Task UnregisterPushAsync(Direction direction, CancellationToken ct = default);
    public Task<IceConfiguration> IceServersAsync(CancellationToken ct = default);
    public SignalChannel Signal(Direction sends);
    // IRelayTransport methods
}

public enum Notify { None, Alert, Silent }

public sealed record PushRegistration(string Service, IReadOnlyDictionary<string, string> Fields,
                                      string Tag, string? Alert = null)
{
    public static string RandomTag();
    public string ToJson();
}

public sealed record IceServer(IReadOnlyList<string> Urls, string? Username, string? Credential);
public sealed record IceConfiguration(IReadOnlyList<IceServer> IceServers, int Ttl);

public sealed class SignalChannel : IAsyncDisposable
{
    public Direction Sends { get; }
    public int? CloseCode { get; }
    public IAsyncEnumerable<SignalEvent> EventsAsync(CancellationToken ct = default);
    public Task SendAsync(byte[] plaintext, CancellationToken ct = default);
    public Task CloseAsync();
}

public abstract record SignalEvent
{
    public sealed record Peer(bool Present) : SignalEvent;
    public sealed record Message(byte[] Data) : SignalEvent;
}

public static class SignalFrame
{
    public const int MaxFrameBytes = 65_536;
    public const ulong WindowMilliseconds = 300_000;
    public static byte[] Seal(ReadOnlySpan<byte> plaintext, Direction direction, ulong counter,
                              Credentials credentials, byte[]? nonce = null);
    public static Opened Open(ReadOnlySpan<byte> frame, Direction direction, ulong? after,
                              ulong now, Credentials credentials);
}

// ZeroByte.Apns
public enum ApnsEnvironment { Production, Development }
public static class Apns
{
    public static PushRegistration Registration(ReadOnlySpan<byte> deviceToken,
        ApnsEnvironment environment, string tag, string? alert = null);
    public static string? TagFromPayload(string payloadJson);
    public static string? TagFromPayload(JsonElement payload);
}

// ZeroByte.Fcm
public sealed record FcmPush(string Tag, Notify Notify, string? Alert);
public static class Fcm
{
    public static PushRegistration Registration(string token, string tag, string? alert = null);
    public static FcmPush? Parse(IReadOnlyDictionary<string, string> data);
}

// ZeroByte.Timeline
public sealed class TimelineSlot(string collection, string key);   // Collection, Key

public sealed class Timeline
{
    public Timeline(ReplicaID? replica = null, IEnumerable<string>? forgetsOwnValues = null,
                    int maximumMessageBytes = TimelineLimits.MaxMessageBytes);
    public static Timeline Decode(ReadOnlySpan<byte> data);
    public byte[] Encode();
    public byte[]? Value(TimelineSlot slot);
    public List<TimelineEntry> Entries(string collection);
    public bool Set(TimelineSlot slot, ReadOnlySpan<byte> value);
    public bool Delete(TimelineSlot slot);
    public bool HasPendingChanges { get; }
    public TimelineReceipt Receive(ReadOnlySpan<byte> data);
    public TimelineMessage.State State(TimelineRequest request = TimelineRequest.None);
    public List<TimelineMessage.CatchUp> CatchUp(
        IReadOnlyDictionary<ReplicaID, TimelineHead> requesterHeads, bool includingOwn = true,
        Func<TimelineSlot, byte[]?>? values = null);
}

public static class TimelineRelay
{
    public static Task<TimelineSyncOutcome> SyncAsync(
        Timeline timeline, Credentials credentials, IRelayTransport transport, Direction sends,
        Func<Timeline, Task> persist, Notify notify = Notify.None, int? commitBudget = null,
        Func<TimelineSlot, byte[]?>? values = null, long? now = null,
        CancellationToken ct = default);   // Updates, Merged, Written, Waiting, IsPeerCurrent
}

public static class TimelineLimits
{
    public const int MaxMessageBytes = 1_048_576;
    public const int MaxCollectionBytes = 64;
    public const int MaxKeyBytes = 1_024;
}

public sealed class TimelineException(string code) : Exception   // Code: malformed,
                                                                 // unsupported_version, too_large, diverged

public interface IRelayTransport
{
    Task ClaimAsync(CancellationToken ct = default);
    Task PutAsync(byte[] envelope, Direction direction, ulong sequence, CancellationToken ct = default);
    Task<Page> FetchAsync(Direction direction, ulong? after, CancellationToken ct = default);
    Task<ulong?> LastSequenceAsync(Direction direction, CancellationToken ct = default);
    Task PruneAsync(Direction direction, ulong through, CancellationToken ct = default);
    Task EraseAsync(CancellationToken ct = default);
}

public sealed record Page(IReadOnlyList<byte[]> Envelopes, bool More);

public static class Envelope
{
    public const int Overhead = 38;
    public const int MaxPlaintextBytes = 1_048_576;
    public const int MaxEnvelopeBytes = MaxPlaintextBytes + Overhead;
    public static bool IsSupported { get; }
    public static byte[] Seal(ReadOnlySpan<byte> plaintext, Direction direction, ulong sequence,
                              Credentials credentials, byte[]? nonce = null);
    public static Opened Open(ReadOnlySpan<byte> envelope, Direction direction, ulong? after,
                              Credentials credentials);
    public static (Direction Direction, ulong Sequence) ReadHeader(ReadOnlySpan<byte> envelope);
}

public static class Wire
{
    public const int MaximumCount = 64;
    public static byte[] Join(IReadOnlyList<byte[]> envelopes);
    public static List<byte[]> Split(ReadOnlySpan<byte> body);
}

public sealed class ZeroByteException : Exception { public string Code { get; } }
public sealed class RelayException : Exception
{
    public string Code { get; }
    public int Status { get; }
    public ulong? Last { get; }
    public TimeSpan? RetryAfter { get; }      // on rate_limited
}
```

The `nonce` parameter of `Envelope.Seal` is for known-answer tests only. Leave it out.

## Testing

```sh
dotnet test
RELAY_BASE=http://127.0.0.1:8080/v1/ RELAY_APP=<app id> dotnet test
```

Set `RELAY_SIGNAL_APP` to an app with `signal` and `push=<topic>` to test the extensions too.

On macOS with Homebrew's .NET, set `DOTNET_ROOT=/opt/homebrew/opt/dotnet/libexec` if the test host
cannot find the runtime.
