# Android SDK integration (Kotlin / Java)

The `com.strybok.sdk` library receives verified HTTPS links (Android App Links) from your
Activity, asks the Strybok API which in-app route a link stands for, and hands your app each
link exactly once.

> **Status:** early release (0.5.x), published on Maven Central. Covered by JVM unit tests, and
> a sample app using it compiles (Kotlin and Java, including an R8 release build). Not yet
> verified on a physical device or emulator (see the [manual test plan](./manual-test-plan.md)).
>
> Links without an API call (path links and Firebase long links) are **optional and off by
> default** (`StrybokConfig.parameterLinks`, new in 0.5.0) and unit-tested only; see
> [Links without an API call](#11-links-without-an-api-call-optional).
>
> Restoring a link that was tapped before the app was installed ("deferred links") is
> **optional and off by default**, and has not run on a device or against a real store; see
> [Deferred links](#10-deferred-links-optional).

Building a Flutter app? Use the [Flutter SDK](./flutter.md) instead. The behaviour shared by all
SDKs is described in the [SDK contract](./sdk-contract.md).

## 1. Prerequisites

1. Create an application in Strybok with your Android package name and certificate fingerprints.
2. Read its setup data: `GET /v1/apps/:appId/integration` returns the managed hostname
   (for example `my-app.strybok.link`) and ready-to-copy snippets.
   If the application has an active [custom domain](./ios.md#custom-domain-optional), the same answer lists it in `allowedHosts`: declare it in the entitlement and the intent filter next to the managed hostname and pass both to `allowedHosts`.
3. Complete the [Android App Links setup](./android.md) so that
   `https://my-app.strybok.link/.well-known/assetlinks.json` lists your app. Without it Android
   never delivers links to the app, and the SDK has nothing to receive.

Requirements: `minSdk` 21, Kotlin coroutines (`kotlinx-coroutines-core` comes with the library).

## 2. Add the library

The library is on Maven Central. In `app/build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.strybok:strybok-android:0.5.0")
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
| `Strybok(context, config)`               | The same, for an instance that can restore a link after installation. See [Deferred links](#10-deferred-links-optional).                                |
| `start(intent, savedInstanceState)`      | Call from `onCreate`. Returns `true` when the intent carries one of your links.                                                                         |
| `start(intent)`                          | Same without the re-creation guard; only for intents known to be fresh.                                                                                 |
| `handleIntent(intent)`                   | Call from `onNewIntent`. Returns `true` when the intent carries one of your links.                                                                      |
| `startWithUrl(url)` / `handleUrl(url)`   | The same with plain strings (tests, custom sources such as a push notification).                                                                        |
| `links`                                  | `Flow<ResolvedDeepLink>` (`url`, `linkId`, `appId`, `routePath`, `payload`, `campaign`, `isDeferred`). Completes on `stop`.                             |
| `addLinkListener` / `removeLinkListener` | Listener alternative to `links`.                                                                                                                        |
| `getInitialLink()`                       | Suspends; the launch link, if not delivered yet. Also `getInitialLink(callback)`.                                                                       |
| `awaitStarted()`                         | Suspends until the launch link was resolved or has failed.                                                                                              |
| `resolve(url)`                           | Suspends; resolve a URL explicitly (QR scanner). Does not emit on `links` or report an app open. Also `resolve(uri: Uri)` and `resolve(url, callback)`. |
| `isStrybokUrl(url)`                      | Whether a URL is one of this app's links (with `parameterLinks` also path links and Firebase long links).                                               |
| `createLink(routePath, …)`               | Suspends; makes a short link to a screen of the app. See [Create a link to share](#9-create-a-link-to-share).                                           |
| `enableDeferredLinks()`                  | Switch deferred links on later, for example after a consent screen; the check then runs once.                                                           |
| `deferredLinkStatus`                     | Where the deferred-link check stands. Also `deferredLinkStatuses` (a `StateFlow`) and `addDeferredLinkStatusListener`.                                  |
| `stop()`                                 | Stop for good; nothing is delivered afterwards. Not needed for a process-wide instance.                                                                 |
| `Strybok.SDK_VERSION`                    | `"0.5.0"`.                                                                                                                                              |

`StrybokConfig` options: `timeoutMillis` (5000 per attempt), `maxRetries` (2; only timeouts,
network errors and 5xx are retried — resolving is an idempotent GET), `retryDelayMillis` (400,
doubles per retry), `dedupeWindowMillis` (2000), `reportAppOpens` (`true`), `onError`,
`clientKey` (only for `createLink`), `attestation` and `integrityTokenProvider` (only for
`createLink`, see [Extra protection](#extra-protection)), `deferredLinks` and
`installReferrerSources` (see [Deferred links](#10-deferred-links-optional)), `parameterLinks`
(see [Links without an API call](#11-links-without-an-api-call-optional)), plus `transport` and
`callbackExecutor` for tests and custom HTTP stacks.

### Errors

`resolve` throws, and `onError` receives, a `StrybokException`. The hierarchy is sealed, so a
`when` is exhaustive; every error has `url`, the link being handled. The same hierarchy holds the
errors that only `createLink` throws (see [Create a link to share](#9-create-a-link-to-share));
they never reach `onError`.

| Type                                                  | Meaning                                                                                                                                                                                                      |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `UnsupportedLinkException`                            | Not `https`, host not in `allowedHosts`, or not a single-slug path (with `parameterLinks`: neither a route path nor a long link). Thrown by `resolve` before any network call. Ignored silently for intents. |
| `LinkNotFoundException`                               | Unknown, disabled or expired link (404).                                                                                                                                                                     |
| `StrybokTimeoutException` / `StrybokNetworkException` | No answer after all retries (e.g. offline). The app keeps working; the link is simply not opened. `cause` holds the transport error.                                                                         |
| `StrybokServerException`                              | Unexpected status or malformed response; see `statusCode`.                                                                                                                                                   |

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

Only the link's host and slug are sent to `resolve`, plus the five `utm_*` campaign parameters of
the tapped URL when they are valid (see below); every other query parameter and the fragment
stay on the device. With `parameterLinks`, `resolve` also gets the path of a path link and the
values of the payload keys you configured, or for a Firebase long link the deep link's scheme,
host and path, its values of those keys and the `apn` and `ibi` parameters (see
[Links without an API call](#11-links-without-an-api-call-optional)); do not configure keys that
carry personal data. `createLink` sends the client key and what you pass to it (route, payload, title,
description, image URL), and only when your app calls it. With `deferredLinks` switched on, the
SDK also reads the store's install referrer and sends the code found in it, your package name
and the SDK version, at most three times in the life of an installation (see
[Deferred links](#10-deferred-links-optional)).

### Campaign values

A resolved link has `campaign` (`LinkCampaign`): the `utm_source`, `utm_medium`, `utm_campaign`,
`utm_term` and `utm_content` of this open as `source`, `medium`, `campaign`, `term` and
`content`. It is never `null`; a property without a value is `null`, and `campaign.isEmpty` is
`true` when there is none at all.

```kotlin
strybok.links.collect { link ->
    link.campaign.source?.let { analytics.setUserProperty("link_source", it) } // a label, nothing more
    navigate(link.routePath)
}
```

```java
String source = link.getCampaign().getSource(); // null when not set
```

- **Where the values come from.** The link's defaults — set in the dashboard (link form →
  "Campaign (optional)") or through the management API — with the `utm_*` parameters of the
  tapped URL on top, key by key. `https://my-app.strybok.link/invite777?utm_source=instagram`
  opens with source `instagram` whatever the default is; a link tapped without parameters gives
  the defaults. A URL can replace a default but cannot clear it.
- **Untrusted labels.** Whoever shares a link can write any `utm_*` value into its URL. Use the
  values as labels for your own analytics; never decide access, prices, rewards or navigation by
  them.
- **What is accepted.** Only these five lower-case names, and only values of 1–64 characters
  out of `A–Z a–z 0–9 . _ ~ -`. Anything else (`spring sale`, an e-mail address, non-Latin
  letters) is dropped silently, as if the parameter were absent — by the SDK and again by the
  server.
- **What is sent.** With the resolve request the SDK sends those five parameters when they are
  valid, and nothing else of the query and never the fragment. With the `app_open` event it
  reports the values exactly as resolve returned them. The dashboard's link analytics then
  shows a "Campaigns" breakdown by source, medium and campaign, with app opens and browser opens
  counted separately (term and content are not broken down).
- **Versions.** New in 0.4.0. Older versions send no query: their links carry no `campaign` and
  their app opens are counted under the link's defaults.

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
    implementation("com.strybok:strybok-android:0.5.0")
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

## 10. Deferred links (optional)

A person without your app taps a link, is sent to your store page, installs the app and opens
it. With deferred links switched on, the SDK delivers that link about a second after the first
start, so the app can open the right screen instead of its home screen. It is **off by
default**, in the dashboard and in the SDK.

What the service does for this, where the code is added to your store URLs and everything it
cannot do is in the
[Android setup guide](./android.md#links-tapped-before-the-app-is-installed-deferred-links); the
exact behaviour of the SDK is in the
[SDK contract](./sdk-contract.md#deferred-links-optional-android).

> **Nothing here has run on a device or against a real store.** The whole flow is covered by JVM
> unit tests with fake sources, a fake store and a fake service. The Google Play source is
> written from Google's documentation. **The Samsung Galaxy Store and Huawei AppGallery sources
> are unverified**: they compile (Samsung) or follow Huawei's published description, and whether
> those stores hand our value to the app at all has never been tried.

**Set-up**

1. Dashboard → your application → **When a link opens in a browser** → switch on **Restore the
   link after install (Android only)**. The Android package name must be set. Do this when an
   app version with step 3 is in the store; before that, codes are handed out for nothing.
2. Add the library of each store you publish in. The SDK compiles against them but does not
   bring them along, so an app that does not use a store adds nothing:

   | Source                             | Store                | Add to the app                                                                                                      | Standing                                                                                                                                                                                      |
   | ---------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | `PlayInstallReferrerSource`        | Google Play          | `implementation("com.android.installreferrer:installreferrer:2.2")`                                                 | Written from Google's documentation. Not device-tested                                                                                                                                        |
   | `GalaxyStoreInstallReferrerSource` | Samsung Galaxy Store | `implementation("store.galaxy.samsung.installreferrer:samsung_galaxystore_install_referrer:4.0.0")` (Maven Central) | **Unverified.** Written from the classes of Samsung's library; Galaxy Store 4.5.20.1 or later. Whether the referrer of an `https` store link reaches the app is known from third parties only |
   | `AppGalleryInstallReferrerSource`  | Huawei AppGallery    | No library. Possibly the two manifest entries below — **unverified**                                                | **Unverified.** Written from Huawei's AppGallery Connect pages; AppGallery 10.5.0.300 or later. Huawei asks for the user's consent first                                                      |

   A source whose library is missing answers "unavailable" and the next one is asked; nothing
   throws.

3. Create the instance **with a `Context`**, switch the option on and list the sources in the
   order to ask them:

```kotlin
val strybok = Strybok(
    applicationContext,
    StrybokConfig(
        apiBaseUrl = "https://api.strybok.com",
        allowedHosts = setOf("my-app.strybok.link"),
        deferredLinks = true,
        installReferrerSources = listOf(
            PlayInstallReferrerSource(applicationContext),
            GalaxyStoreInstallReferrerSource(applicationContext),
            AppGalleryInstallReferrerSource(applicationContext),
        ),
    ),
)
```

Java: `new Strybok(context, new StrybokConfig.Builder(…).deferredLinks(true).installReferrerSources(Arrays.asList(new PlayInstallReferrerSource(context))).build())`.

A source whose store did not install the app returns no code and the next one is asked, so the
list above is fine for an app published in all three stores. `Strybok(config)` without a context
refuses `deferredLinks = true` with an `IllegalArgumentException`.

**Receiving the link.** Nothing changes in your code: the link arrives on `links` (or your
listener) like any other, with `isDeferred == true`.

```kotlin
strybok.links.collect { link ->
    if (link.isDeferred) analytics.label("opened_after_install")
    navigate(link.routePath)
}
```

- It arrives **after** the app has started — typically within a second, later when the store or
  the network is slow. `start`, the launch link, `getInitialLink()` and `createLink` never wait
  for it, and `getInitialLink()` never returns it. Be ready to navigate from a screen that is
  already showing.
- If a normal link of your app arrives first — the launch link, or one tapped while the check
  is running — the normal link wins and the deferred one is dropped: one launch, one
  navigation.
- No `app_open` event is sent for it; the service counted the exchange itself.

**Asking for consent first.** Reading the store's install referrer is access to information
stored on the device, and Huawei requires the user's consent for its referrer. Whether your app
must ask is for you to decide. If it does, leave `deferredLinks` off and call
`strybok.enableDeferredLinks()` once the person agreed: the check then runs once. Until then the
SDK reads, sends and stores nothing.

**What happens, and how often.** The check runs at most once per process, after `start` has
done everything else. It asks the sources in order; when a referrer carries a code
(`strybok_click=dc_…`), the SDK sends that code, your package name and the SDK version to
`POST /v1/public/deferred-links/redeem` — nothing about the device and nothing else from the
referrer — and delivers the link it gets back. The referrer and the code are never logged or
stored. The SDK keeps two values in the app's private no-backup storage (so a restored backup
does not look like an old installation): how many launches tried, and whether the check is
finished.

- A launch that cannot finish (the store or the network is not reachable) leaves the check for
  the next launch; after **three** launches it gives up.
- A launch that crashes during the check has used one of the three. A link is delivered **at
  most once**: it can be lost, it is never delivered twice.
- An app that was **updated** since it was installed is never asked: a person who merely updates
  to a version with this feature does not get an old link.
- A source may take 10 seconds; after that it is abandoned for this launch.

Nothing of this is reported to `onError` and nothing throws. For development and QA, read
`strybok.deferredLinkStatus` (or collect `deferredLinkStatuses`, or use
`addDeferredLinkStatusListener`):

| Status             | Meaning                                                                                                  |
| ------------------ | -------------------------------------------------------------------------------------------------------- |
| `DISABLED`         | `deferredLinks` is off.                                                                                  |
| `CHECKING`         | The check is running in this launch (or waits for `start`).                                              |
| `PENDING`          | This launch could not finish the check; a later launch tries again.                                      |
| `NOT_FIRST_LAUNCH` | The check was finished in an earlier launch, or the app was updated since it was installed. Final.       |
| `UNAVAILABLE`      | No source is configured, or none can ever answer on this device (no such store, library missing). Final. |
| `NO_REFERRER`      | The store returned a referrer without a code of ours: the usual result of an ordinary install. Final.    |
| `NOT_FOUND`        | The service refused the code: unknown, older than 24 hours, already used, or another app's. Final.       |
| `SUPERSEDED`       | A normal link of your app arrived first. Final.                                                          |
| `DELIVERED`        | The link was handed to `links`. Final.                                                                   |
| `FAILED`           | Three launches could not finish the check, or the answer of the service was unusable. Final.             |

`isFinal` says whether this installation will ever check again. To run the check again while
developing, clear the app's data.

**Huawei AppGallery: manifest entries that may be needed.** AppGallery hands the referrer over
through a content provider. Huawei's page does not say whether an app that targets Android 11
or later must declare that it wants to see the provider, or hold a permission to read it, and
this has not been tried. The SDK therefore adds **nothing** to your manifest. If
`AppGalleryInstallReferrerSource` answers "unavailable" on a device that has AppGallery, try:

```xml
<manifest>
    <queries>
        <provider android:authorities="com.huawei.appmarket.commondata" />
    </queries>
    <uses-permission android:name="com.huawei.appmarket.service.commondata.permission.GET_COMMON_DATA" />
</manifest>
```

**Another store, or your own code.** `InstallReferrerSource` is a one-method interface:
`getInstallReferrer()` is called on a background thread and returns
`InstallReferrerResult.referrer(string)`, `InstallReferrerResult.UNAVAILABLE_FOR_NOW` (try again
in a later launch) or `InstallReferrerResult.UNAVAILABLE` (asking again is pointless). It must
not throw and must close what it opened. An app that already reads the Play referrer for
another SDK can pass the string on instead of connecting twice.

**In tests**, pass a source of your own and answer the request in your `transport`:

```kotlin
val config = StrybokConfig(
    apiBaseUrl = "https://api.example.test",
    allowedHosts = setOf("my-app.strybok.link"),
    deferredLinks = true,
    installReferrerSources = listOf(
        InstallReferrerSource { InstallReferrerResult.referrer("strybok_click=dc_0123456789abcdefghijABCDEFGHIJ01") },
    ),
    transport = { request ->
        // POST …/v1/public/deferred-links/redeem
        StrybokHttpResponse(
            200,
            """{"linkId":"l1","appId":"a1","routePath":"/promo/42","url":"https://my-app.strybok.link/promo42","deferred":true}""",
        )
    },
    callbackExecutor = { it.run() },
)
```

The state lives in the app's storage, so this needs an instrumented test (or Robolectric) for
`Strybok(context, config)`.

**The limits, plainly:**

- It works **only when the app is installed from the store page the link sent the person to**
  (the store _app_, not the store's website, not after a search, not from an APK), **within 24
  hours** of the tap, **once**.
- Only on Android, and only for the stores above. iOS has no install referrer; what exists
  there instead is in the [iOS guide](./ios.md#links-tapped-before-the-app-is-installed), and
  this SDK has no part in it.
- **Huawei AppGallery and Samsung Galaxy Store have never been verified**, not even that the
  stores pass the value on. Google Play is written from Google's documentation.
- **None of it has run on a device.** The calls into Google's and Samsung's libraries compile
  against them and are otherwise untested; the AppGallery query is tested against a fake
  cursor.

## 11. Links without an API call (optional)

The owner of the application can let addresses of its hosts open a route of the app without
creating a link first (dashboard: **Setup → Links without an API call**, off by default):
**path links** such as `https://my-app.strybok.link/product/42?id=7`, and **Firebase long links**
`https://my-app.strybok.link/?link=https://example.com/product/42&apn=…`. The intent filter of
section 3 declares the whole host, so Android hands these links to the app like any other. The
SDK delivers them only with `StrybokConfig.parameterLinks`, new in 0.5.0. Without it, and in
earlier versions, a path of several segments and a long link are ignored without a request, and
a one-segment path is delivered without its payload.

Copy the keys from `parameterLinks` of the integration guide (`GET /v1/apps/:appId/integration`,
shown on the dashboard's setup screen). It is `null` while both kinds are off; then leave the
option out.

```kotlin
val strybok = Strybok(
    StrybokConfig(
        apiBaseUrl = "https://api.strybok.com",
        allowedHosts = setOf("my-app.strybok.link"),
        parameterLinks = ParameterLinks(payloadKeys = listOf("id", "lang")),
    ),
)

strybok.links.collect { link ->
    // Tapped: https://my-app.strybok.link/product/42?id=7&lang=uk
    // link.routePath == "/product/42" and link.payload == mapOf("id" to "7", "lang" to "uk"),
    // as the service read them.
    navigate(link.routePath)
}
```

Java: `new StrybokConfig.Builder(…).parameterLinks(new ParameterLinks(Arrays.asList("id", "lang"))).build()`.

- **What arrives.** A link like any other, on `links`, a listener or from `getInitialLink()`,
  with the route and payload the service read — never the tapped path. Treat them as untrusted,
  as for every link. The `app_open` event carries the `linkId` the service returned.
- **What is sent.** The path exactly as received, the first value of each payload key (kept when
  it is 1–256 characters without control characters) and the `utm_*` values; for a long link the
  deep link's scheme, host and path with its own values of the payload keys, plus `apn` and
  `ibi`. Nothing else of the query — not `afl`, `ifl`, `ofl`, `st`, `sd`, `si` or `isi` — and
  never the fragment. The SDK reads the URL string as the intent carried it; it never decodes or
  normalises the path.
- **Which addresses count.** Any address of `allowedHosts` whose path is a valid route (no `//`,
  no `.` or `..` segment, not `/favicon.ico`, `/robots.txt` or anything under `/.well-known/`) is
  resolved. Whether a route is allowed is the service's decision, so `onError` gets a
  `LinkNotFoundException` for an address that is no link, a typo for example.
- **The keys are compiled in.** Installed builds keep sending the keys they know, and the service
  ignores keys it no longer allows: a key the owner adds reaches users with your next release.
  An invalid list — more than 20 keys, a key twice, a key that is not 1–40 characters of
  `A–Z a–z 0–9 _ . -`, starts with `utm_` or is `st`, `sd`, `si` or `strybok_click` — makes
  `StrybokConfig` throw an `IllegalArgumentException`.
- **Deferred links do not change.** They exist only for links created in advance, so a restored
  link still has a single slug.

The exact rules are in the
[SDK contract](./sdk-contract.md#links-without-an-api-call). This is covered by JVM unit tests
against the shared test vectors of the contract only; it has not run on a device, an emulator or
against the production API.

## Troubleshooting

| Symptom                                                | Check                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Link opens the browser/store instead of the app        | The App Link is not verified: see the [Android setup](./android.md) (`adb shell pm get-app-links <package>`). The SDK is not involved yet at this point.                                                                                                                                                                                                                                                                              |
| App opens but nothing happens                          | `start`/`handleIntent` returned `false`? Then the host is missing from `allowedHosts` or the intent is not a `VIEW` intent. Otherwise look at `onError`; is the API reachable from the device? Is anything collecting `links`?                                                                                                                                                                                                        |
| Works on cold start, not while the app is running      | `onNewIntent` does not call `handleIntent`, or the Activity is not `singleTop`/`singleTask` and a second instance was created.                                                                                                                                                                                                                                                                                                        |
| Screen opens again after rotation or from recents      | `start(intent)` is used without `savedInstanceState`, or the app also navigates from `intent.data` itself or through a Navigation `navDeepLink` for the same host.                                                                                                                                                                                                                                                                    |
| Screen opens twice                                     | Two collectors/listeners are attached at the same time (each one receives every link), or both `getInitialLink()` and your own `intent.data` handling navigate.                                                                                                                                                                                                                                                                       |
| `onError` gets `StrybokNetworkException` in debug      | No `INTERNET` permission, or `apiBaseUrl` is `http://` and cleartext traffic is blocked by the network security config.                                                                                                                                                                                                                                                                                                               |
| `RuntimeException: Method ... not mocked` in tests     | Set `callbackExecutor` and add the `org.json` test dependency (see Testing).                                                                                                                                                                                                                                                                                                                                                          |
| No link after installing from the store                | Read `deferredLinkStatus`: `DISABLED` — the option is off; `UNAVAILABLE` — the store's library is missing or no source is listed; `NO_REFERRER` — the install did not come from the page the link opened; `NOT_FOUND` — the code is older than 24 hours, was used, or the dashboard switch is off; `NOT_FIRST_LAUNCH` — the check already ran (clear the app's data to try again). See [Deferred links](#10-deferred-links-optional). |
| Works from Notes/Messages, not from Instagram/Telegram | In-app browsers often do not hand links to apps. Users get the fallback page with a manual button.                                                                                                                                                                                                                                                                                                                                    |
