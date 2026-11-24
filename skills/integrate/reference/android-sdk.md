# Android SDK integration (Kotlin / Java)

The `com.strybok.sdk` library receives verified HTTPS links (Android App Links) from your
Activity, asks the Strybok API which in-app route a link stands for, and hands your app each
link exactly once.

> **Status:** early release (0.2.x), published on Maven Central. Covered by JVM unit tests, and
> a sample app using it compiles (Kotlin and Java, including an R8 release build). Not yet
> verified on a physical device or emulator (see the [manual test plan](./manual-test-plan.md)).
>
> Deferred deep linking is **not supported**: a link tapped before the app is installed is not
> restored after installation.

Building a Flutter app? Use the [Flutter SDK](./flutter.md) instead. The behaviour shared by all
SDKs is described in the [SDK contract](./sdk-contract.md).

## 1. Prerequisites

1. Create an application in Strybok with your Android package name and certificate fingerprints.
2. Read its setup data: `GET /v1/apps/:appId/integration` returns the managed hostname
   (for example `my-app.strybok.link`) and ready-to-copy snippets.
3. Complete the [Android App Links setup](./android.md) so that
   `https://my-app.strybok.link/.well-known/assetlinks.json` lists your app. Without it Android
   never delivers links to the app, and the SDK has nothing to receive.

Requirements: `minSdk` 21, Kotlin coroutines (`kotlinx-coroutines-core` comes with the library).

## 2. Add the library

The library is on Maven Central. In `app/build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.strybok:strybok-android:0.3.0")
}
```

`mavenCentral()` must be among your repositories (it is in a project created by Android Studio).
The library is built with Gradle 9.1 and Android Gradle plugin 9.0.1 (built-in Kotlin); the
source is at [github.com/strybok/strybok-android](https://github.com/strybok/strybok-android).

No ProGuard/R8 rules are needed.

## 3. Declare the link in the manifest

`app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />

<application ...>
    <activity
        android:name=".MainActivity"
        android:exported="true"
        android:launchMode="singleTop">

        <!-- your existing MAIN/LAUNCHER filter stays as it is -->

        <intent-filter android:autoVerify="true">
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
            <category android:name="android.intent.category.BROWSABLE" />
            <data android:scheme="https" android:host="my-app.strybok.link" />
        </intent-filter>
    </activity>
</application>
```

Declare the whole host (no path filter); the slug is resolved at runtime. `singleTop` (or
`singleTask`) makes Android hand a link to the running Activity through `onNewIntent` instead of
stacking a second copy of it.

## 4. Use it

Create **one instance for the whole process**, for example in your `Application`:

```kotlin
class App : Application() {
    lateinit var strybok: Strybok
        private set

    override fun onCreate() {
        super.onCreate()
        strybok = Strybok(
            StrybokConfig(
                apiBaseUrl = "https://api.strybok.com",
                allowedHosts = setOf("my-app.strybok.link"),
                onError = { error -> Log.w("Strybok", "Link failed", error) },
            ),
        )
    }
}
```

Android has no global stream of incoming URLs: the system gives links to an Activity as intents.
Forward them from `onCreate` and `onNewIntent`, and collect `links`:

```kotlin
class MainActivity : AppCompatActivity() {
    private val strybok get() = (application as App).strybok

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        strybok.start(intent, savedInstanceState)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                strybok.links.collect { link ->
                    // link.routePath == "/invite/abc123"
                    // link.payload["invitationId"] == "abc123"
                    openRoute(link)
                }
            }
        }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        setIntent(intent)
        strybok.handleIntent(intent)
    }
}
```

One collector covers every case:

| Situation                                 | What happens                                                                                          |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Cold start** (app launched by the link) | `start()` resolves the launch link. It is kept until something collects `links`, then delivered once. |
| **Background** (app resumed by the link)  | `onNewIntent` → `handleIntent` → delivered on `links`.                                                |
| **Foreground**                            | `onNewIntent` → `handleIntent` → delivered on `links`.                                                |

A link that finishes resolving while nothing is collecting (the collector above is cancelled while
the Activity is stopped) is queued and delivered to the next collector. Nothing is lost, nothing
is delivered twice.

### Configuration changes and re-created activities

Android re-creates an Activity after a rotation, and after the process was killed in the
background, **with the intent it was originally launched with**. The SDK must not navigate to
that old link again:

- Always pass `savedInstanceState`: `strybok.start(intent, savedInstanceState)`. When it is not
  `null` the Activity is being re-created and the intent is ignored.
- Intents re-delivered from the recents screen (`FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY`) are
  ignored by `start` and `handleIntent`.
- Keep the `Strybok` instance in the `Application` (or another process-wide holder), not in the
  Activity. A link that is still resolving during a rotation then survives it and is delivered to
  the new Activity's collector.
- Calling `start` again is harmless: the first call defines the launch link, later calls behave
  like `handleIntent`. This is what makes a second `onCreate` work when the user left with Back
  and opened the app through another link.
- The identical URL arriving twice within `dedupeWindowMillis` (2 s by default) is dropped. The
  same link tapped again later is a new delivery.

Prefer to read the launch link yourself, for example to pick the start destination?
`strybok.getInitialLink()` suspends until the launch link is resolved (or has failed), returns it
at most once and removes it from `links`, so combining both never navigates twice.

### Jetpack Compose with Navigation

```kotlin
private val inviteRoute = Regex("^/invite/([A-Za-z0-9_-]{1,64})$")

@Composable
fun AppNavHost(strybok: Strybok, navController: NavHostController = rememberNavController()) {
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(strybok, navController, lifecycleOwner) {
        lifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            strybok.links.collect { link ->
                // Map the untrusted route onto destinations the app knows.
                val invite = inviteRoute.matchEntire(link.routePath)
                if (invite != null) {
                    navController.navigate("invite/${invite.groupValues[1]}") { launchSingleTop = true }
                }
            }
        }
    }

    NavHost(navController = navController, startDestination = "home") {
        composable("home") { HomeScreen() }
        composable("invite/{code}") { entry ->
            InviteScreen(code = entry.arguments?.getString("code").orEmpty())
        }
    }
}

class MainActivity : ComponentActivity() {
    private val strybok get() = (application as App).strybok

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        strybok.start(intent, savedInstanceState)
        setContent { AppNavHost(strybok) }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        setIntent(intent)
        strybok.handleIntent(intent)
    }
}
```

Do not also declare a `navDeepLink { uriPattern = "https://my-app.strybok.link/..." }` for the
same host: Navigation would handle the intent as well, and the screen would open twice.

This Compose snippet is illustrative; it was not compiled as part of the SDK build.

### Java

```java
public class MainActivity extends AppCompatActivity {
    private final StrybokLinkListener listener = this::openRoute;

    private Strybok strybok() {
        return ((App) getApplication()).getStrybok();
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        strybok().start(getIntent(), savedInstanceState);
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        setIntent(intent);
        strybok().handleIntent(intent);
    }

    @Override
    protected void onStart() {
        super.onStart();
        strybok().addLinkListener(listener); // queued links arrive right away
    }

    @Override
    protected void onStop() {
        strybok().removeLinkListener(listener);
        super.onStop();
    }

    private void openRoute(ResolvedDeepLink link) {
        if (link.getRoutePath().startsWith("/invite/")) { /* navigate */ }
    }
}
```

Create the instance with the builder:

```java
Strybok strybok = new Strybok(
    new StrybokConfig.Builder("https://api.strybok.com", Collections.singleton("my-app.strybok.link"))
        .onError(error -> Log.w("Strybok", "Link failed", error))
        .build());
```

Listeners and callbacks run on the main thread unless you set `callbackExecutor`.

### API

| Member                                   | Purpose                                                                                                                                                 |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Strybok(config)`                        | Create the instance; does no work until `start`.                                                                                                        |
| `start(intent, savedInstanceState)`      | Call from `onCreate`. Returns `true` when the intent carries one of your links.                                                                         |
| `start(intent)`                          | Same without the re-creation guard; only for intents known to be fresh.                                                                                 |
| `handleIntent(intent)`                   | Call from `onNewIntent`. Returns `true` when the intent carries one of your links.                                                                      |
| `startWithUrl(url)` / `handleUrl(url)`   | The same with plain strings (tests, custom sources such as a push notification).                                                                        |
| `links`                                  | `Flow<ResolvedDeepLink>` (`url`, `linkId`, `appId`, `routePath`, `payload`). Completes on `stop`.                                                       |
| `addLinkListener` / `removeLinkListener` | Listener alternative to `links`.                                                                                                                        |
| `getInitialLink()`                       | Suspends; the launch link, if not delivered yet. Also `getInitialLink(callback)`.                                                                       |
| `awaitStarted()`                         | Suspends until the launch link was resolved or has failed.                                                                                              |
| `resolve(url)`                           | Suspends; resolve a URL explicitly (QR scanner). Does not emit on `links` or report an app open. Also `resolve(uri: Uri)` and `resolve(url, callback)`. |
| `isStrybokUrl(url)`                      | Whether a URL is one of this app's links.                                                                                                               |
| `createLink(routePath, …)`               | Suspends; makes a short link to a screen of the app. See [Create a link to share](#9-create-a-link-to-share).                                           |
| `stop()`                                 | Stop for good; nothing is delivered afterwards. Not needed for a process-wide instance.                                                                 |
| `Strybok.SDK_VERSION`                    | `"0.3.0"`.                                                                                                                                              |

`StrybokConfig` options: `timeoutMillis` (5000 per attempt), `maxRetries` (2; only timeouts,
network errors and 5xx are retried — resolving is an idempotent GET), `retryDelayMillis` (400,
doubles per retry), `dedupeWindowMillis` (2000), `reportAppOpens` (`true`), `onError`,
`clientKey` (only for `createLink`), `attestation` and `integrityTokenProvider` (only for
`createLink`, see [Extra protection](#extra-protection)), plus `transport` and `callbackExecutor`
for tests and custom HTTP stacks.

### Errors

`resolve` throws, and `onError` receives, a `StrybokException`. The hierarchy is sealed, so a
`when` is exhaustive; every error has `url`, the link being handled. The same hierarchy holds the
errors that only `createLink` throws (see [Create a link to share](#9-create-a-link-to-share));
they never reach `onError`.

| Type                                                  | Meaning                                                                                                                                        |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `UnsupportedLinkException`                            | Not `https`, host not in `allowedHosts`, or not a single-slug path. Thrown by `resolve` before any network call. Ignored silently for intents. |
| `LinkNotFoundException`                               | Unknown, disabled or expired link (404).                                                                                                       |
| `StrybokTimeoutException` / `StrybokNetworkException` | No answer after all retries (e.g. offline). The app keeps working; the link is simply not opened. `cause` holds the transport error.           |
| `StrybokServerException`                              | Unexpected status or malformed response; see `statusCode`.                                                                                     |

```kotlin
when (error) {
    is LinkNotFoundException -> showMessage("This link is no longer valid")
    is StrybokTimeoutException, is StrybokNetworkException -> showMessage("You seem to be offline")
    is StrybokServerException, is UnsupportedLinkException -> Unit
    else -> Unit // createLink only
}
```

After `stop()`, `resolve` throws `IllegalStateException`; `start`, `handleIntent` and
`handleUrl` do nothing and return `false`.

## 5. Treat routes as untrusted input

`routePath` and `payload` are data from the network, chosen by whoever created the link. Match
the route against routes your app knows before navigating, and never execute payload contents.
`payload` is a read-only `Map<String, Any?>` of plain JSON values (`String`, `Boolean`, numbers,
`List`, `Map`, `null`).

## 6. Links that arrive before sign-in

If a link needs an authenticated user, remember the route when it arrives and open it after
login instead of dropping it. The SDK delivers each link once; holding on to it across your
sign-in flow is up to the app.

## 7. Analytics and privacy

After a link is delivered (or queued) the SDK sends one best-effort `app_open` event
(`POST /v1/public/events`) containing a random event id, the link id, the platform (`android`)
and the SDK version — no device identifiers. It never delays navigation and all failures are
ignored. Disable it with `reportAppOpens = false`.

Only the link's host and slug are sent to `resolve`; query parameters and fragments stay on the
device. `createLink` sends the client key and what you pass to it (route, payload, title,
description, image URL), and only when your app calls it.

## 8. Testing your integration

`startWithUrl` / `handleUrl` take plain strings and `StrybokConfig.transport` replaces the
network, so the flow can be tested on the JVM without a device. Set `callbackExecutor` in JVM
tests: the default one posts to the Android main thread.

```kotlin
@Test
fun opensInvite() = runBlocking {
    val strybok = Strybok(
        StrybokConfig(
            apiBaseUrl = "https://api.strybok.com",
            allowedHosts = setOf("my-app.strybok.link"),
            transport = { request ->
                if (request.method == "GET") {
                    StrybokHttpResponse(200, """{"linkId":"l1","appId":"a1","routePath":"/invite/abc123"}""")
                } else {
                    StrybokHttpResponse(202, "") // app_open event
                }
            },
            callbackExecutor = { it.run() },
        ),
    )
    strybok.startWithUrl("https://my-app.strybok.link/invite777")
    assertEquals("/invite/abc123", strybok.getInitialLink()?.routePath)
    strybok.stop()
}
```

`org.json` is only a stub in local unit tests; add `testImplementation("org.json:json:20250517")`
to tests that run SDK code on the JVM.

On a device or emulator, simulate a tap:

```sh
adb shell am start -a android.intent.action.VIEW -c android.intent.category.BROWSABLE \
  -d "https://my-app.strybok.link/<slug>"
```

Check cold start (app force-stopped first), background, foreground, rotation right after
opening a link, and reopening the app from the recents screen (the link must not open again).

The SDK's own tests: `./gradlew test` in the SDK directory.

## 9. Create a link to share

`createLink` asks Strybok for a short link to a screen of your app — what a "Share" button
needs. It is optional and independent of receiving links.

1. In the dashboard, open the application and enable **Links from the app**.
2. List the **route patterns** the app may create links for, for example `/product/*`. A pattern
   is made of `/`-separated segments: a literal, `*` for exactly one segment, or `**` at the end
   for one or more. A route outside the list is refused.
3. Copy the **publishable key** (`pk_…`) into the configuration as `clientKey`.

The publishable key is not a secret: it is safe to ship inside the app, because it can only
create links for the listed routes of this one application. **Never put a secret API key
(`lb_…`) into an app**; `createLink` does not accept one.

```kotlin
val strybok = Strybok(
    StrybokConfig(
        apiBaseUrl = "https://api.strybok.com",
        allowedHosts = setOf("my-app.strybok.link"),
        clientKey = "pk_…",
    ),
)

// In a coroutine, for example from the click handler of the Share button:
try {
    val link = strybok.createLink(
        routePath = "/product/42",
        payload = mapOf("productId" to 42), // optional
        title = "Red shoes", // optional, as are description and imageUrl
    )
    startActivity(
        Intent.createChooser(
            Intent(Intent.ACTION_SEND).setType("text/plain").putExtra(Intent.EXTRA_TEXT, link.url),
            null,
        ),
    )
} catch (error: StrybokException) {
    when (error) {
        is StrybokRateLimitedException, is StrybokNetworkException,
        is StrybokTimeoutException, is StrybokServerException,
        -> showMessage("Could not create the link, try again later")
        else -> Log.e("Strybok", "Link creation is misconfigured", error)
    }
}
```

Java uses a callback, which runs on the main thread unless `callbackExecutor` is set; pass `null`
for the values you do not need:

```java
// Configuration: new StrybokConfig.Builder(...).clientKey("pk_…").build()
strybok.createLink("/product/42", payload, "Red shoes", null, null, new StrybokCreateLinkCallback() {
    @Override public void onCreated(CreatedLink link) { share(link.getUrl()); }
    @Override public void onError(StrybokException error) { /* see the table below */ }
});
```

The result is a `CreatedLink` with `url` (what the user shares), `slug`, `routePath` and
`linkId`. `payload` is a `Map<String, Any?>` of plain JSON values (`String`, `Boolean`, numbers,
`null`, `List`, `Map` with `String` keys).

| Error                                                                            | Meaning                                                                                                                         | Retrying helps |
| -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| `StrybokUnauthorizedException`                                                   | No `clientKey` configured, or the key is wrong or was replaced, or the feature is switched off. The service does not say which. | no             |
| `RouteNotAllowedException`                                                       | `routePath` is not covered by the route patterns in the dashboard.                                                              | no             |
| `StrybokValidationException`                                                     | The route, payload, title, description or image URL was rejected; see `message` and `issues`.                                   | no             |
| `StrybokRateLimitedException`                                                    | Too many requests, or the app's daily limit of new links is reached; `retryAfterSeconds` when the service says how long.        | later          |
| `StrybokNetworkException` / `StrybokTimeoutException` / `StrybokServerException` | No usable answer after all retries.                                                                                             | later          |

Things to know:

- Creating the same content again returns the same link, so repeated taps on "Share" do not pile
  up links, and the SDK can safely retry timeouts, network errors and 5xx answers (`maxRetries`,
  `retryDelayMillis`, `timeoutMillis`). 4xx answers, including 429, are never retried.
- It runs only when you call it: `start`, incoming links and `resolve` never wait for it. It works
  before `start` and does not use `allowedHosts`. After `stop()` it throws `IllegalStateException`.
- Errors are thrown to the caller (or given to the callback), never to `onError`, and never
  contain the key or the payload.
- Do not put personal data into the route or the payload: whoever receives the link can read
  them.
- In tests, answer the `POST …/v1/public/links` request in your `transport` with
  `StrybokHttpResponse(201, """{"linkId":"l1","url":"https://my-app.strybok.link/abc","slug":"abc","routePath":"/product/42"}""")`.

### Extra protection

The publishable key can be copied out of your app, so anyone can create links for the routes you
allowed. If that matters to you, the owner of the application can ask Strybok to accept link
requests only — or preferably — from a **genuine build of your app on a genuine device**. On
Android the proof is Google **Play Integrity**. What this does and does not protect, and the
dashboard side, are described in
[Keeping link creation from your app safe](./link-creation-security.md); read
[its limits](./link-creation-security.md#the-limits) before you rely on it.

It is off by default, and the SDK does not depend on Google's libraries: an app for a market
without Google Play uses the same artifact and simply does not switch this on.

**Set-up** (details and the reasons are in the security guide):

1. Dashboard → your application: fill in the **package name** and the **SHA-256 fingerprints of
   your signing certificates** (with Play App Signing, the _app signing key_ certificate from
   Play Console → **Test and release** → **App integrity**, not your upload key).
2. [Google Cloud console](https://console.cloud.google.com/): choose or create a project and
   enable the **Google Play Integrity API**. Note the **project number** (a number, not the
   project id).
3. [Play Console](https://play.google.com/console/): your app → **Test and release** → **App
   integrity** → **Play Integrity API** → **link that Cloud project**.
4. Cloud console → **IAM & Admin** → **Service Accounts**: create a service account only for
   this purpose, with only the roles Google's Play Integrity documentation asks for, and create
   a **JSON key** for it.
5. Dashboard → **Setup** → **Links from the app** → **Extra protection (optional)** →
   **Android — Play Integrity** → **Upload Google key**, then delete the downloaded file. Choose
   the mode: **Preferred** first; **Required** only after you have seen your released app pass.
6. In the app, add Google's library yourself and switch attestation on:

```kotlin
// build.gradle.kts of the app
dependencies {
    implementation("com.strybok:strybok-android:0.3.0")
    implementation("com.google.android.play:integrity:1.4.0") // or newer; not pulled in by the SDK
}
```

```kotlin
val strybok = Strybok(
    StrybokConfig(
        apiBaseUrl = "https://api.strybok.com",
        allowedHosts = setOf("my-app.strybok.link"),
        clientKey = "pk_…",
        attestation = true,
        integrityTokenProvider = PlayIntegrityTokenProvider(
            context = applicationContext,
            cloudProjectNumber = 123456789012L, // the Cloud project number from step 2
        ),
    ),
)
```

Java: `new StrybokConfig.Builder(…).clientKey("pk_…").attestation(true).integrityTokenProvider(new PlayIntegrityTokenProvider(context, 123456789012L)).build()`.

7. Test with a build **installed from Google Play** (an internal testing track is enough). A
   build installed from Android Studio is not "recognised by Play" and is refused.

`createLink` is called exactly as before. The first call — and about one call an hour after
that — first asks the service for a one-time challenge, asks Google Play for an integrity token
bound to it and exchanges that for a session token, which is kept in memory only. Nothing of
this happens when the SDK starts or when a link arrives, and never ahead of time.

Two more errors can come out of `createLink`:

| Error                                 | Meaning                                                                                                                                                                                                                                                     | Retrying helps |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| `StrybokAttestationRequiredException` | The mode is **Required** and this request carried no proof: the device or build cannot attest (see the limits), or `attestation` is off. Expected on such devices — tell the user that sharing is not available here rather than show an error.             | no             |
| `StrybokAttestationFailedException`   | The service refused the session token, the SDK attested again, and the fresh token was refused too. `reason` is `invalid`, `expired` or `exhausted` when the service gave one. Should not happen in a correctly set-up release; check the dashboard set-up. | no             |

**When a device cannot attest** — no Google Play services or Play Store, the library is not in
the app, Google's quota is used up, a network failure, or the dashboard set-up is incomplete —
the SDK does not fail on its own. It sends the request without a proof and reports what the
service answers: the link (mode **Off**, or **Preferred** within the small shared allowance),
`StrybokRateLimitedException` (that allowance is used up) or
`StrybokAttestationRequiredException` (mode **Required**). It then waits 60 seconds before it
tries to attest again. Attestation never affects receiving links and never throws anything but
the `StrybokException`s of `createLink`.

**Without Google Play.** `integrityTokenProvider` is a small interface
(`StrybokIntegrityTokenProvider`: given the request hash, return a Play Integrity token or
`null`). `PlayIntegrityTokenProvider` is the ready-made implementation; it only works when the
app itself adds `com.google.android.play:integrity`, and reports "cannot attest" (with one
warning in logcat) when the library is missing. For a build flavour that must not contain
Google libraries, leave the dependency out and pass `integrityTokenProvider = null`: nothing of
Google is referenced or downloaded. An app that already uses Play Integrity can implement the
interface on top of its own code instead; the request must be a **standard** request with the
given `requestHash`.

**The limits, plainly:**

- It needs **Google Play services, the Play Store and a build installed from Google Play**. It
  does not work on devices without Google services — most devices sold in mainland China,
  Huawei devices without Google Mobile Services — or for sideloaded builds and builds from
  other stores. With mode **Required** those users cannot create links at all.
- **Google limits how often you can ask**: 10,000 integrity requests a day per app by default; a
  higher tier has to be requested from Google. The SDK asks about once an hour per running app
  (more often when a token is used up after 30 new links).
- Strybok needs a **Google service-account key** of yours to read the verdicts.
- It raises the cost of abuse; it does not stop a real user of the real app, and a proof can be
  taken from a device an attacker controls.
- **This code has never run against Google's real service or on a physical device.** It is
  covered by unit tests with a fake token provider and a fake service only; the calls into
  Google's library (`PlayIntegrityTokenProvider`) compile against it and are otherwise
  unverified. Treat **Required** as experimental until you have seen your own released build
  pass.

## Troubleshooting

| Symptom                                                | Check                                                                                                                                                                                                                          |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Link opens the browser/store instead of the app        | The App Link is not verified: see the [Android setup](./android.md) (`adb shell pm get-app-links <package>`). The SDK is not involved yet at this point.                                                                       |
| App opens but nothing happens                          | `start`/`handleIntent` returned `false`? Then the host is missing from `allowedHosts` or the intent is not a `VIEW` intent. Otherwise look at `onError`; is the API reachable from the device? Is anything collecting `links`? |
| Works on cold start, not while the app is running      | `onNewIntent` does not call `handleIntent`, or the Activity is not `singleTop`/`singleTask` and a second instance was created.                                                                                                 |
| Screen opens again after rotation or from recents      | `start(intent)` is used without `savedInstanceState`, or the app also navigates from `intent.data` itself or through a Navigation `navDeepLink` for the same host.                                                             |
| Screen opens twice                                     | Two collectors/listeners are attached at the same time (each one receives every link), or both `getInitialLink()` and your own `intent.data` handling navigate.                                                                |
| `onError` gets `StrybokNetworkException` in debug      | No `INTERNET` permission, or `apiBaseUrl` is `http://` and cleartext traffic is blocked by the network security config.                                                                                                        |
| `RuntimeException: Method ... not mocked` in tests     | Set `callbackExecutor` and add the `org.json` test dependency (see Testing).                                                                                                                                                   |
| Works from Notes/Messages, not from Instagram/Telegram | In-app browsers often do not hand links to apps. Users get the fallback page with a manual button.                                                                                                                             |
