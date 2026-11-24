# Unity integration

The `com.strybok.sdk` package receives verified HTTPS links through Unity's own deep link API
(`Application.absoluteURL` and `Application.deepLinkActivated`), asks the Strybok API which
in-app route a link stands for, and hands your game each link exactly once. It follows the
shared [SDK contract](./sdk-contract.md).

> **Status:** covered by Edit Mode tests (Unity 6000.5 and 6000.6 editors on macOS), including the
> HTTP transport against a local server. Generating the Xcode project and exporting the Gradle
> project were checked for the entitlement and the manifest entry. **Not verified on a physical
> device or simulator, and not run on Unity 2021.3 / 2022.3 yet** (see the
> [manual test plan](./manual-test-plan.md)); not published to a registry.
>
> Deferred deep linking is **not supported**: a link tapped before the app is installed is not
> restored after installation.

Requirements: Unity 2021.3 or newer, iOS and Android players, Mono or IL2CPP. The package has no
third-party dependencies and uses no reflection, so it needs no `link.xml` for code stripping.

## 1. Prerequisites

1. Create an application in Strybok with your iOS bundle identifier and Android package name
   (_Player Settings > Other Settings > Identification_).
2. Read its setup data: `GET /v1/apps/:appId/integration` returns the managed hostname
   (for example `my-app.strybok.link`).
3. Do **step 1** of the [Android](./android.md) and [iOS](./ios.md) guides (team ID, bundle ID,
   package name, certificate fingerprints) so Strybok serves the association files. Step 2 of
   those guides — the manifest and the entitlement — is done by this package, see below.

## 2. Add the package

_Window > Package Manager > + > Add package from git URL_:

```text
https://github.com/strybok/strybok-unity.git#0.2.0
```

The part after `#` pins a tag or commit. The package is not in a registry (neither OpenUPM nor
the Asset Store). Or clone the repository and reference the clone in `Packages/manifest.json`
(the path is relative to the `Packages` folder):

```json
{
  "dependencies": {
    "com.strybok.sdk": "file:../../strybok-unity"
  }
}
```

## 3. Configure

Open _Tools > Strybok > Settings_. This creates `Assets/Resources/StrybokSettings.asset`:

| Field                          | Meaning                                                                       |
| ------------------------------ | ----------------------------------------------------------------------------- |
| **Api Base Url**               | `https://api.strybok.com`                                                     |
| **Link Hosts**                 | Your managed hostname(s), e.g. `my-app.strybok.link`. No `https://`, no path. |
| **Report App Opens**           | Send the best-effort `app_open` event (section 8).                            |
| **Timeout Seconds**            | Time for one resolve attempt (default 5).                                     |
| **Max Retries**                | Extra attempts after a timeout, network error or 5xx (default 2).             |
| **Client Key**                 | Publishable key `pk_…`, only for creating links to share (section 6).         |
| **Add Ios Associated Domains** | Let the build add the Associated Domains entitlement (section 4).             |
| **Add Android Intent Filter**  | Let the build add the App Links intent filter (section 5).                    |

The asset is used at runtime **and** by the build steps below, so the hostname lives in one
place. It must stay in a `Resources` folder under the name `StrybokSettings`.

## 4. iOS: Associated Domains

With **Add Ios Associated Domains** on, a post-process build step adds

```xml
<key>com.apple.developer.associated-domains</key>
<array>
    <string>applinks:my-app.strybok.link</string>
</array>
```

to the entitlements file of the `Unity-iPhone` target in the generated Xcode project (it creates
`Unity-iPhone.entitlements` when the target has none, and keeps domains other plugins added).
Nothing else is needed in Unity: Unity's app controller already forwards Universal Links to
`Application.absoluteURL` / `Application.deepLinkActivated`.

Still to do by you:

- The **Associated Domains** capability must be enabled for the App ID in the Apple Developer
  portal. Xcode's automatic signing does that; with manual signing, regenerate the provisioning
  profile after enabling it.
- Verify on a device as described in the [iOS guide](./ios.md#3-verify-on-a-device).

Manual alternative (turn the toggle off): in Xcode select the _Unity-iPhone_ target >
_Signing & Capabilities_ > _+ Capability_ > _Associated Domains_ and add
`applinks:my-app.strybok.link`. This has to be repeated whenever Unity regenerates the Xcode
project with _Replace_. The development-only `applinks:<host>?mode=developer` form is not written
by the build step; add it by hand in Xcode when you need it.

## 5. Android: App Links intent filter

With **Add Android Intent Filter** on, a build step adds this to the launcher activity
(`UnityPlayerActivity` and/or `UnityPlayerGameActivity`) of the generated Gradle project, once
per link host:

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="my-app.strybok.link" />
</intent-filter>
```

Your project files are not modified; a custom `Assets/Plugins/Android/AndroidManifest.xml` keeps
working and is patched in the generated copy only. Hosts that the activity already declares are
skipped.

Manual alternative (turn the toggle off): enable _Player Settings > Publishing Settings >
Custom Main Manifest_ and paste the block above inside the Unity `<activity>` of
`Assets/Plugins/Android/AndroidManifest.xml`.

Then verify with `adb shell pm get-app-links <package>` as described in the
[Android guide](./android.md#3-verify-on-a-device). Remember the fingerprint of the certificate
that signs the build you test (the Unity debug keystore for development builds).

## 6. Use it

Either add the **Strybok Behaviour** component (_Add Component > Strybok > Strybok Behaviour_)
to an object in your first scene — it initializes the SDK from the settings asset and survives
scene loads — or initialize from code:

```csharp
using Strybok;
using UnityEngine;
using UnityEngine.SceneManagement;

public sealed class DeepLinkRouter : MonoBehaviour
{
    private void Awake()
    {
        DontDestroyOnLoad(gameObject);
        StrybokSdk.LinkReceived += OnLink;
        StrybokSdk.LinkFailed += error => Debug.LogWarning("Link failed: " + error);

        // Skip this call when a Strybok Behaviour is in the scene.
        StrybokSdk.Initialize(new StrybokOptions
        {
            ApiBaseUrl = "https://api.strybok.com",
            AllowedHosts = { "my-app.strybok.link" },
        });
        // or: StrybokSdk.Initialize();  // reads the StrybokSettings asset
    }

    private void OnDestroy()
    {
        StrybokSdk.LinkReceived -= OnLink;
    }

    // Runs on the main thread. link.RoutePath == "/invite/abc123",
    // link.Payload["invitationId"] == "abc123"
    private void OnLink(ResolvedDeepLink link)
    {
        string[] segments = link.RoutePath.Trim('/').Split('/');
        switch (segments[0])
        {
            case "invite" when segments.Length == 2:
                InviteScreen.PendingCode = segments[1];
                SceneManager.LoadScene("Invite");
                break;
            case "shop":
                ShopScreen.PendingItemId = link.GetString("itemId");
                SceneManager.LoadScene("Shop");
                break;
            default:
                Debug.Log("Unknown route " + link.RoutePath); // Stay where we are.
                break;
        }
    }
}
```

One subscription covers every case:

| Situation                                 | What happens                                                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| **Cold start** (app launched by the link) | `Initialize` resolves the launch link. It is kept until you subscribe to `LinkReceived`, then delivered once. |
| **Background** (app resumed by the link)  | Delivered on `LinkReceived`.                                                                                  |
| **Foreground**                            | Delivered on `LinkReceived`.                                                                                  |

Links resolved while nobody is subscribed are queued and handed to the **first** subscriber,
synchronously inside its `+=`. Subscribe from an object that lives as long as link handling
should (hence `DontDestroyOnLoad` above), and unsubscribe when it is destroyed.

If the platform reports the same tap twice (launch URL _and_ `deepLinkActivated`), the duplicate
is dropped (`DedupeWindow`, 2 s by default). The same link tapped again later is a new delivery.

Call `Initialize` early, in the first scene. If it is called late, the most recent link is still
picked up from `Application.absoluteURL`, but an older one that arrived in between is lost.

Prefer to decide the first scene yourself? Await the launch link instead; it is returned at most
once and removed from `LinkReceived`, so combining both never navigates twice:

```csharp
private async void Start()
{
    await StrybokSdk.Initialize();
    ResolvedDeepLink launch = await StrybokSdk.GetInitialLinkAsync();
    SceneManager.LoadScene(launch != null && launch.RoutePath.StartsWith("/invite/") ? "Invite" : "Menu");
}
```

The API is `Task`-based and needs no coroutine or GameObject; continuations and handlers run on
Unity's main thread. In a coroutine, wait with
`yield return new WaitUntil(() => task.IsCompleted);`.

### API

| Member                             | Purpose                                                                                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `StrybokSdk.Initialize(options)`   | Start listening; the task completes after the launch link was handled. Calling it again is harmless.      |
| `StrybokSdk.Initialize()`          | The same, configured from the `StrybokSettings` asset.                                                    |
| `StrybokSdk.LinkReceived`          | Event with a `ResolvedDeepLink` (`Url`, `Uri`, `LinkId`, `AppId`, `RoutePath`, `Payload`, `PayloadJson`). |
| `StrybokSdk.LinkFailed`            | Event with a `StrybokException` for a link of yours that could not be resolved.                           |
| `StrybokSdk.GetInitialLinkAsync()` | The launch link, if not delivered through `LinkReceived` yet; otherwise `null`.                           |
| `StrybokSdk.ResolveAsync(url)`     | Resolve a URL explicitly (QR scanner, tests). Does not raise `LinkReceived` or report an app open.        |
| `StrybokSdk.IsStrybokUrl(url)`     | Whether a URL is one of this app's links.                                                                 |
| `StrybokSdk.CreateLinkAsync(...)`  | Create a short link to share, see [below](#create-a-link-to-share).                                       |
| `StrybokSdk.Shutdown()`            | Stop listening and drop queued links.                                                                     |

Options (`StrybokOptions`): `ApiBaseUrl`, `AllowedHosts`, `Timeout` (5 s per attempt),
`MaxRetries` (2; only timeouts, network errors and 5xx are retried — resolving is an idempotent
GET), `RetryDelay` (400 ms, doubling), `DedupeWindow` (2 s), `ReportAppOpens`, `OnError`,
`OnHandlerException`, `ClientKey` (only for creating links), plus `LinkSource` / `HttpTransport` / `Platform` / `CallbackContext` for
tests and custom set-ups.

`StrybokSdk` is a thin static wrapper around `StrybokClient`, a plain C# class without any
`UnityEngine` dependency. Create your own `StrybokClient` when you need several instances or run
outside Unity; you then pass an `ILinkSource` and an `IHttpTransport` yourself
(`UnityLinkSource` and `UnityWebRequestTransport` are public).

### Payload

`link.Payload` is an `IReadOnlyDictionary<string, object>` produced by a small built-in JSON
reader (`JsonUtility` cannot read arbitrary objects). Values are `string`, `bool`, `long` for
integers, `double` for other numbers, `null`, `Dictionary<string, object>` for objects and
`List<object>` for arrays. `link.GetString("key")` returns a string value or `null`.

For a typed payload use the raw JSON with `JsonUtility` (or any JSON library you already ship):

```csharp
[System.Serializable]
private sealed class InvitePayload
{
    public string invitationId;
    public int bonus;
}

InvitePayload payload = JsonUtility.FromJson<InvitePayload>(link.PayloadJson);
```

### Errors

`ResolveAsync` fails with, and `LinkFailed` / `OnError` receive, a `StrybokException`. Its `Url`
is the link that was being handled; `ToString()` leaves the URL out because a query can hold
private data.

| Type                                                  | Meaning                                                                                                                                              |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `UnsupportedLinkException`                            | Not `https`, host not in `AllowedHosts`, or not a single-slug path. Raised before any network call. Ignored silently for links arriving from the OS. |
| `LinkNotFoundException`                               | Unknown, disabled or expired link (404).                                                                                                             |
| `StrybokTimeoutException` / `StrybokNetworkException` | No answer after all retries (e.g. offline). The game keeps working; the link is simply not opened.                                                   |
| `StrybokServerException`                              | Unexpected status or malformed response; `StatusCode` tells which.                                                                                   |

When neither `LinkFailed` nor `OnError` is set, failures are logged as warnings. An exception
thrown by one of your own handlers is logged with `Debug.LogException` and does not stop other
handlers or later links. After `Shutdown`, `ResolveAsync` on the old client fails with
`ObjectDisposedException`.

### Create a link to share

`CreateLinkAsync` asks the service for a short link to a screen of your game — the "Share"
button. It is new in 0.2.0.

1. In the dashboard, open the application, switch on **Links from the app** and list the route
   patterns the app may create links for (for example `/product/*`). Routes outside the list are
   refused.
2. Copy the **publishable client key** (`pk_…`) into **Client Key** of the settings asset, or
   set `ClientKey` in `StrybokOptions`. The key is safe to ship inside the game: it can only
   create links for those patterns. **Never put a secret API key (`lb_…`) in a game**; it is not
   accepted here.

```csharp
try
{
    CreatedLink link = await StrybokSdk.CreateLinkAsync(
        "/product/42",
        payload: new Dictionary<string, object> { { "productId", 42 } },
        title: "Red shoes");
    Share(link.Url); // https://my-app.strybok.link/Ab3xK9
}
catch (StrybokRateLimitedException error)
{
    ShowTryAgainLater(error.RetryAfter); // may be null
}
catch (StrybokException error)
{
    Debug.LogWarning("Could not create a link: " + error);
}
```

Only the route is required; `payload`, `title`, `description` and `imageUrl` are optional and
left out of the request when not given. `CreateLinkWithPayloadJsonAsync` takes the payload as
JSON text instead (`JsonUtility.ToJson(myPayload)`); it must be a JSON object. Asking again for
the same content returns the same link, so repeated taps do not pile up links.

It only runs when you call it and is independent of receiving links: it works before
`Initialize` (then from the settings asset, which does not need a link host for this), never
raises `LinkReceived` or `LinkFailed`, and failures come back from the awaited task:

| Type                                                                             | Meaning                                                                                                              | Retry later?            |
| -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| `StrybokUnauthorizedException`                                                   | No `ClientKey` set (no network call), or the key was refused: wrong or replaced key, or the feature is switched off. | no                      |
| `RouteNotAllowedException`                                                       | `RoutePath` is not covered by the route patterns in the dashboard.                                                   | no                      |
| `StrybokValidationException`                                                     | Bad route or payload (no network call), or rejected by the service; `Issues` lists what it found.                    | no                      |
| `StrybokRateLimitedException`                                                    | Too many requests, or the app's daily limit of new links is reached.                                                 | yes, after `RetryAfter` |
| `StrybokNetworkException` / `StrybokTimeoutException` / `StrybokServerException` | No usable answer after the retries (`Timeout`, `MaxRetries`, `RetryDelay` as for resolving).                         | yes                     |

Errors never contain the client key or the payload. Do not put personal data in a route or
payload: whoever receives the link can read them.

## 7. Treat routes as untrusted input

`RoutePath` and `Payload` are data from the network, chosen by whoever created the link. Match
the route against screens your game knows before navigating (the `switch` above), validate ids
before using them, and never pass payload contents to `SceneManager.LoadScene`,
`Application.OpenURL`, `Resources.Load` or similar directly.

If a link needs a signed-in player or finished loading, remember the route and open it once the
game is ready instead of dropping it.

## 8. Analytics and privacy

After a link is delivered (or queued) the SDK sends one best-effort `app_open` event
(`POST /v1/public/events`) containing a random event id, the link id, the platform
(`ios` / `android`, `unknown` in the Editor and on other platforms) and the SDK version — no
device identifiers. It never delays navigation and all failures are ignored. Disable it with
**Report App Opens** / `ReportAppOpens = false`.

The dashboard shows these as **App opens**, separate from browser fallbacks counted by the
server. The number is self-reported by the app and is not a count of unique people.

Only the link's host and slug are sent to `resolve`; query parameters and fragments stay on the
device.

## 9. Testing your integration

In the Editor `Application.absoluteURL` is empty and no links arrive, so simulate them with
`Strybok.Testing.FakeLinkSource`:

```csharp
var source = new FakeLinkSource(initialLink: "https://my-app.strybok.link/invite777");
StrybokSdk.Initialize(new StrybokOptions
{
    ApiBaseUrl = "https://api.strybok.com",
    AllowedHosts = { "my-app.strybok.link" },
    LinkSource = source,
});
source.Emit("https://my-app.strybok.link/other"); // a link arriving while the game runs
```

For unit tests without a network, also pass your own `IHttpTransport`:

```csharp
private sealed class FakeTransport : IHttpTransport
{
    public Task<StrybokHttpResponse> SendAsync(StrybokHttpRequest request, CancellationToken cancellationToken)
    {
        return Task.FromResult(new StrybokHttpResponse(
            200,
            "{\"linkId\":\"l1\",\"appId\":\"a1\",\"routePath\":\"/invite/abc123\",\"payload\":{}}"));
    }
}
```

The package's own tests (`Tests/Editor`) show complete examples. To run them, add
`"testables": ["com.strybok.sdk"]` to `Packages/manifest.json` and open
_Window > General > Test Runner > EditMode_.

On devices, send a link the way the native guides describe: tap it in Notes or Messages on iOS;
`adb shell am start -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d "https://my-app.strybok.link/<slug>"`
on Android.

## Troubleshooting

| Symptom                                                     | Check                                                                                                                                                                       |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Link opens the browser/store instead of the game            | Native association is not verified: see the [Android](./android.md) and [iOS](./ios.md) guides. The SDK is not involved yet at this point.                                  |
| No entitlement / intent filter in the built project         | `StrybokSettings.asset` is not in a `Resources` folder, **Link Hosts** is empty, or the toggle is off. Look for `[Strybok]` warnings in the build log.                      |
| iOS build fails to sign after adding the entitlement        | The App ID / provisioning profile lacks the Associated Domains capability.                                                                                                  |
| Game opens but nothing happens                              | Host missing from `AllowedHosts` / **Link Hosts**; `LinkFailed` output or `[Strybok]` warnings in the log; API reachable from the device? `Initialize` called?              |
| Works when the game is running, not on cold start           | Subscribe to `LinkReceived` (or call `GetInitialLinkAsync`) at some point after start; the launch link waits in the queue until then. Call `Initialize` in the first scene. |
| Screen opens twice                                          | You navigate from both `GetInitialLinkAsync()` and your own `Application.deepLinkActivated` handler. Use the SDK events only.                                               |
| Handler never runs after a scene change                     | The subscribing object was destroyed and unsubscribed. Subscribe from a `DontDestroyOnLoad` object.                                                                         |
| `Initialize was called again` warning                       | A Strybok Behaviour and your own code both initialize. Keep one.                                                                                                            |
| Works on tap in Notes/Messages, not from Instagram/Telegram | In-app browsers often do not hand links to apps. Users get the fallback page with a manual button.                                                                          |
