---
name: integrate
description: Integrates Strybok deep links into an existing app project - detects the platform (Flutter, React Native, Expo, Capacitor/Ionic, native iOS, native Android, Kotlin Multiplatform, Unity, .NET MAUI, Unreal), installs the Strybok SDK, configures iOS Universal Links (Associated Domains) and Android App Links (autoVerify intent filter), routes each delivered link into the app's existing navigation, optionally adds share-link creation, then verifies. Use when the user asks to add or set up Strybok, to integrate deep links, Universal Links or App Links through Strybok or a strybok.link host, to replace Firebase Dynamic Links with Strybok, or to create share links with Strybok (createLink).
argument-hint: '[platform or app directory] [link host, e.g. my-app.strybok.link]'
allowed-tools: Read, Grep, Glob
---

# Integrate Strybok deep links

Strybok gives an app a managed link host (`<name>.strybok.link`). A link on that host opens the
app through iOS Universal Links / Android App Links; the SDK asks the Strybok API which in-app
route the link stands for and hands the app each link exactly once. Your job is to wire that
into a project you have not seen before, with small edits the user can review.

Arguments, if any: `$ARGUMENTS` (a platform, an app directory, a link host).

Work through the steps in order. Do not skip step 0 or the plan in step 3.

## Rules that always apply

- **Ask before changing the project.** Show a short plan (files to edit, dependency to add,
  commands to run) and wait for the user's approval before you add a dependency or edit a native
  project file (`AndroidManifest.xml`, `*.entitlements`, `Info.plist`, `project.pbxproj`, Gradle
  files, `app.json`, `*.csproj`, `Packages/manifest.json`, `*.ini`, `*.uproject`).
- **Minimal, reviewable edits.** Add to the project's existing files and patterns; do not
  reformat, restructure or "clean up" anything else. Keep existing entitlements, intent filters
  and deep link handlers that are unrelated to Strybok.
- **Never touch signing material:** keystores, `key.properties`, certificates, provisioning
  profiles, `google-services.json`, signing settings in Xcode or Gradle. You may _read_ a
  fingerprint with `keytool` only when the user asks you to.
- **Never commit, push, or change the Strybok dashboard/API** (`PATCH`, `POST`) unless the user
  asks for exactly that.
- **Secrets.** A secret API key starts with `lb_`; it can manage every app and link of the
  organization. Never ask the user to paste one into the chat, never print it, never write it
  into the project, never pass it to an SDK. The only key that may appear in app code is the
  publishable client key, which starts with `pk_` (see "Optional: creating links").
- **Never invent API names.** Take every class, method, option and config key from the reference
  guide for the platform. After installing, if the guide and the installed package disagree,
  trust the installed package (its type definitions, public headers, README and CHANGELOG) and
  tell the user about the difference.
- **Routes are untrusted input.** `routePath` and `payload` come from the network. Navigate only
  to routes the app already defines; never pass `routePath` straight to the router, never
  evaluate or execute payload contents.
- **Be honest about what is verified.** Nothing you can run in a terminal proves that iOS or
  Android opens the app for a link. Say so in the summary.

## Step 0: gather the inputs

You need these values. Find what you can in the project, then ask the user for the rest in one
message. Do not guess any of them.

| Value                                                    | Where it comes from                                                                                                                                           |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Link host, e.g. `my-app.strybok.link`                    | Strybok dashboard: the application's **Setup** tab, **Link hostname**. Ask the user.                                                                          |
| Custom domain, e.g. `go.example.com` (only if any)       | Setup tab, **Custom domain**. Use it only when its status is `active`; ask the user whether it is.                                                            |
| API base URL                                             | `https://api.strybok.com` unless the user names another one.                                                                                                  |
| iOS bundle identifier                                    | Project: `PRODUCT_BUNDLE_IDENTIFIER` in `project.pbxproj`, `ios.bundleIdentifier` in `app.json`, `ApplicationId` in a MAUI `.csproj`, Unity Player Settings.  |
| Apple Team ID (10 characters)                            | Project: `DEVELOPMENT_TEAM` in `project.pbxproj` if set; otherwise ask.                                                                                       |
| Android package name (application ID)                    | Project: `applicationId` in the app module's Gradle file, `android.package` in `app.json`.                                                                    |
| SHA-256 fingerprints of the Android signing certificates | The user. For a Play release it is the **App signing key certificate** from Play Console, not the upload key. The Android guide lists the `keytool` commands. |
| Which platforms the app ships on                         | Project (is there an `ios/` and an `android/` part?) and the user.                                                                                            |

The identifiers and fingerprints are not put into the app by you: the user enters them in the
dashboard (**Setup** tab, **1. App identifiers**: Apple Team ID, Bundle ID, Package name, SHA-256
certificate fingerprints), and Strybok then serves the association files. You need them to check
in step 5 that the dashboard and the project agree.

If the user has no Strybok application yet, tell them to create one in the dashboard first and
come back with the link host; you can prepare everything else meanwhile.

### Optional: read the setup data from the API

Only if the user says a secret API key (`lb_…`, scope `apps:read`) is already in an environment
variable, tells you the variable's name, and gives you the application id (it is in the
dashboard URL of the app, `/apps/<appId>/setup`). Reference the variable by name so the value
never appears in the conversation:

```sh
curl -fsS -H "Authorization: Bearer $NAME_THE_USER_GAVE" \
  "https://api.strybok.com/v1/apps/<appId>/integration"
```

The answer contains:

- `hostname`: the link host;
- `allowedHosts`: the link host plus every **active** custom domain; `customDomains` lists all
  of them with their status, `linkHostname` the host new link URLs use;
- `ios` and `android`: each with `status` (`configured` or `incomplete`), `missing` (which
  identifiers are not set yet: `iosTeamId`, `iosBundleId`, `androidPackageName`,
  `androidSha256Fingerprints`), `associationUrl` and `associationFile`;
- `snippets`: `iosAssociatedDomainsEntitlement`, `androidManifestIntentFilter`,
  `androidVerifyCommands`, and for Flutter `iosInfoPlistFlutter` and `androidManifestFlutter`;
- `parameterLinks`: `{ "payloadKeys": [...] }` while the owner has switched on links without an
  API call (path links, Firebase long links), otherwise `null`: the value of the SDK option of
  the same name (see "Optional: links without an API call" below);
- `osVerification`: always `not_checked`. `configured` only means the association file can be
  generated; it never means the OS verified it.

Do not echo the variable, do not run the command with shell tracing, and do not store the answer
in the project. Without such a variable, simply ask the user for the values.

## Step 1: detect the platform

Look at the project files (do not ask what you can read). Check in this order: a framework
project also contains native `ios/` and `android/` folders, and those do **not** make it a
native project.

| Evidence in the project                                                                                                                      | Platform             | Guide to read                                                          |
| -------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- | ---------------------------------------------------------------------- |
| `pubspec.yaml` with a `flutter:` SDK dependency                                                                                              | Flutter              | [reference/flutter.md](reference/flutter.md)                           |
| `package.json` depending on `expo`                                                                                                           | Expo (React Native)  | [reference/react-native.md](reference/react-native.md)                 |
| `package.json` depending on `react-native`, no `expo`                                                                                        | Bare React Native    | [reference/react-native.md](reference/react-native.md)                 |
| `package.json` depending on `@capacitor/core` (with or without Ionic)                                                                        | Capacitor            | [reference/capacitor.md](reference/capacitor.md)                       |
| Another JavaScript runtime (Cordova `config.xml`, NativeScript, a custom WebView shell)                                                      | JavaScript core      | [reference/javascript.md](reference/javascript.md)                     |
| A Gradle module applying the Kotlin Multiplatform plugin (`kotlin("multiplatform")` / `org.jetbrains.kotlin.multiplatform`) with iOS targets | Kotlin Multiplatform | [reference/kotlin-multiplatform.md](reference/kotlin-multiplatform.md) |
| `ProjectSettings/ProjectVersion.txt` and `Packages/manifest.json`                                                                            | Unity                | [reference/unity.md](reference/unity.md)                               |
| A `*.uproject` file                                                                                                                          | Unreal Engine        | [reference/unreal.md](reference/unreal.md)                             |
| A `.csproj` with `<UseMaui>true</UseMaui>` (or a `net*-ios` / `net*-android` target framework without MAUI)                                  | .NET MAUI / .NET     | [reference/dotnet-maui.md](reference/dotnet-maui.md)                   |
| A `*.xcodeproj`, `*.xcworkspace` or app `Package.swift`, and none of the above                                                               | Native iOS (Swift)   | [reference/ios-sdk.md](reference/ios-sdk.md)                           |
| A Gradle module applying `com.android.application`, and none of the above                                                                    | Native Android       | [reference/android-sdk.md](reference/android-sdk.md)                   |

Then:

- **Several apps or a monorepo** (more than one match, or several app modules): list what you
  found and ask which app to integrate. Do one app at a time.
- **A native iOS app and a native Android app side by side** are two integrations: do both, one
  after the other, with the same link host.
- **Nothing matches** (for example a web-only project): say so and stop. Strybok SDKs are for
  mobile apps; do not improvise an integration.
- **Strybok already present** (the dependency or the link host is already in the project): read
  what is there, report it, and only fill the gaps.
- State your conclusion in one line before going on, e.g. "This is an Expo app
  (`apps/mobile`), iOS and Android; I will follow the React Native guide."

## Step 2: read the matching guide, and only that one

Read the guide from the table for the detected platform, completely, before planning. It is the
source of truth for the package name, the API, the native configuration and the known limits.
Do not load guides for other platforms. Read these shared guides when the platform guide points
to them:

- [reference/ios.md](reference/ios.md): Associated Domains, checking the association file,
  verifying on a device. Its Xcode steps name the `Runner` target because it was written for
  Flutter; use the app's real target.
- [reference/android.md](reference/android.md): which fingerprints, the intent filter, `adb`
  verification commands.
- [reference/link-creation-security.md](reference/link-creation-security.md): the risk of the
  public client key, the built-in limits and the optional attestation (App Attest, Play
  Integrity) with its limits. Read it before adding `createLink`.
- [reference/sdk-contract.md](reference/sdk-contract.md): the behaviour all SDKs share (which
  URLs are accepted, delivery exactly once, errors, `createLink`, campaign values, deferred
  links). Read it when something is unclear or the user asks how it works.
- [reference/manual-test-plan.md](reference/manual-test-plan.md): the device scenarios to hand
  to the user at the end.

The guides use `my-app.strybok.link` (in places `…links.example.com`) and `https://api.strybok.com`
(in places `https://api.example.com`) as placeholders: always substitute the user's real link
host and API base URL.

### Maturity: tell the user before you start

Deferred deep linking (restoring a link that was tapped before the app was installed) is
**optional, off by default and not verified on any device or against any real store**. A normal
integration does not include it. If the user is replacing Firebase Dynamic Links and relies on
it, say this now:

- **Android:** possible through the store's install referrer (Google Play; Huawei AppGallery and
  Samsung Galaxy Store are unverified), only for installs from the store page the link opened.
- **iOS:** nothing is automatic. The only ways are a link the person copies before the App Store
  and pastes in the app, or opt-in probabilistic matching, which Apple's rules forbid.
- It needs an SDK version that has the `deferredLinks` option (the last column below). If the
  installed version has no such option, it is not available: do not improvise it.

No SDK has been verified on physical devices yet. On top of that:

| Platform             | Status to pass on                                                                                                                                                                              | Link creation (`createLink`) | Deferred links (optional, unverified)                     |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- | --------------------------------------------------------- |
| Flutter              | Early release, on pub.dev, unit and widget tests.                                                                                                                                              | yes                          | from 0.3.0; the app supplies referrer, pasted text, store |
| Native iOS           | Unit tests on macOS and the iOS Simulator.                                                                                                                                                     | yes                          | from 0.4.0: pasted link, opt-in matching                  |
| Native Android       | Early release, on Maven Central, JVM unit tests.                                                                                                                                               | yes                          | from 0.4.0, store referrer sources included               |
| Kotlin Multiplatform | Early release, on Maven Central; not tried inside a complete Compose Multiplatform app.                                                                                                        | yes                          | from 0.4.0, both platforms                                |
| React Native / Expo  | Unit tests with a fake `Linking` module; never run inside a real React Native or Expo app. The Expo Router snippet is a sketch.                                                                | yes                          | from 0.3.0; the app supplies referrer, pasted text, store |
| Capacitor            | Unit tests with fake plugins; never run inside a real Capacitor app.                                                                                                                           | yes                          | from 0.3.0; the app supplies referrer, pasted text, store |
| JavaScript core      | Unit tests; you must supply the `LinkSource` for the runtime.                                                                                                                                  | yes                          | from 0.3.0; the app supplies referrer, pasted text, store |
| **Unity**            | **Experimental.** Edit Mode tests only; not run on a device or simulator, not run on Unity 2021.3 / 2022.3; installed from OpenUPM or a git URL.                                               | yes (0.2.0)                  | from 0.3.0; the app supplies the referrer and pasted text |
| **.NET MAUI**        | **Experimental.** The core library is tested; the MAUI glue has only been compiled, never run inside a MAUI app, and the guide's code samples were not compiled.                               | yes (0.2.0)                  | from 0.3.0; the app supplies the referrer and pasted text |
| **Unreal Engine**    | **Experimental.** Never compiled with Unreal Build Tool and never run on a device; only the engine-independent core is tested. Expect compile errors; the guide ends with a "Known gaps" list. | yes (0.2.0), uncompiled      | from 0.3.0, uncompiled; the game supplies both from C++   |

A version named here may not be published yet: check what the registry really offers (step 4a).
Attestation of `createLink` (App Attest, Play Integrity) exists only in the native iOS, native
Android and Kotlin Multiplatform SDKs, from 0.3.0.

For the three experimental SDKs, get an explicit "go ahead" from the user after telling them
this.

## Step 3: plan, then ask

Before the first edit, show a plan of at most a dozen lines:

1. the SDK package and the version you will add (step 4a);
2. each file you will create or edit, with one line on what changes;
3. where the SDK will be created and where links will be routed (step 4c, 4d), naming the
   router you found and the routes you intend to accept;
4. the commands you will run;
5. what the user will have to do by hand afterwards.

Wait for approval. If the user changes the plan, follow the user.

## Step 4: integrate

### 4a. Install the SDK with the project's own tools

Use the package manager the project already uses, and the current published version rather
than a number copied from a guide. Look the version up:

| Platform             | Package                                        | Find the current version                                                                  | Add it                                                                                                                  |
| -------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Flutter              | `strybok`                                      | `flutter pub add` picks it                                                                | `flutter pub add strybok`                                                                                               |
| React Native / Expo  | `@strybok/react-native`                        | `npm view @strybok/react-native version`                                                  | The project's manager, chosen by its lockfile: `npm install`, `yarn add`, `pnpm add` or `bun add`                       |
| Capacitor            | `@strybok/capacitor` (and `@capacitor/app`)    | `npm view @strybok/capacitor version`                                                     | As above, then `npx cap sync`                                                                                           |
| JavaScript core      | `@strybok/js`                                  | `npm view @strybok/js version`                                                            | As above                                                                                                                |
| Native iOS           | Swift package `github.com/strybok/strybok-ios` | `git ls-remote --tags https://github.com/strybok/strybok-ios`                             | `Package.swift`, XcodeGen/Tuist manifest, or Xcode (see below)                                                          |
| Native Android       | `com.strybok:strybok-android`                  | `curl -fsS https://repo1.maven.org/maven2/com/strybok/strybok-android/maven-metadata.xml` | `implementation(...)` in the app module; use the version catalog (`gradle/libs.versions.toml`) if the project has one   |
| Kotlin Multiplatform | `com.strybok:strybok-kmp`                      | `curl -fsS https://repo1.maven.org/maven2/com/strybok/strybok-kmp/maven-metadata.xml`     | `api(...)` in `commonMain` **and** `export(...)` in the iOS framework, as the guide shows                               |
| .NET MAUI            | `Strybok`, `Strybok.Maui`                      | `dotnet add package` picks it                                                             | `dotnet add <project> package Strybok` and `… package Strybok.Maui`                                                     |
| Unity                | `com.strybok.sdk` (git URL)                    | `git ls-remote --tags https://github.com/strybok/strybok-unity`                           | `Packages/manifest.json`: `"com.strybok.sdk": "https://github.com/strybok/strybok-unity.git#<tag>"`                     |
| Unreal Engine        | plugin `Strybok`                               | `git ls-remote --tags https://github.com/strybok/strybok-unreal`                          | Clone (or add as a submodule) into `Plugins/Strybok`, enable it in the `.uproject`, add the module to `<Game>.Build.cs` |

- If a lookup fails (offline, registry unreachable), use the version the guide shows and tell
  the user it was not checked.
- **Xcode projects without a manifest:** adding a Swift package means editing `project.pbxproj`
  by hand, which is easy to get wrong. Default to giving the user the Xcode steps from the
  guide (_File → Add Package Dependencies…_) and continue with the code; edit the `pbxproj`
  yourself only if the user asks, and then build to prove the project still opens.
- After installing, open the installed package's own README or type definitions once and check
  the names you are about to use (see "Never invent API names").

### 4b. Native configuration

Follow the platform guide's native setup section exactly. The points that most often go wrong:

**iOS: Associated Domains entitlement.** The app target needs
`com.apple.developer.associated-domains` containing `applinks:<link host>`: no `https://`, no
path, the exact host.

- Find the target's entitlements file: the `CODE_SIGN_ENTITLEMENTS` build setting in
  `project.pbxproj`. If it exists, add the key to it, or add the `applinks:` entry to the
  existing array. Keep every other entitlement and every other domain.
- **No entitlements file yet:** create `<AppFolder>/<Target>.entitlements` (a plist with the
  key above), and make the app target use it by setting
  `CODE_SIGN_ENTITLEMENTS = <AppFolder>/<Target>.entitlements;` in **every** build
  configuration of the app target (Debug, Release, Profile, …), not of the project and not of
  test targets. A file that no build setting references is silently ignored. This is the one
  `pbxproj` edit that is usually needed; show it in the plan. The file does not need to be in a
  build phase.
- Do not add `?mode=developer` unless the user asks for a development-only setup; it must not
  ship.
- Platform specifics: Flutter uses the `Runner` target (`ios/Runner/Runner.entitlements`);
  Expo sets `ios.associatedDomains` in `app.json` / `app.config.js` instead of editing native
  files; Capacitor uses the `App` target in `ios/App`; MAUI uses
  `Platforms/iOS/Entitlements.plist`; Unity adds the entitlement in a build step driven by its
  settings asset; Unreal needs a premade entitlements file and an `.ini` entry. Each guide has
  the details.
- Bare React Native must also forward Universal Links to `RCTLinkingManager` in the
  `AppDelegate`; the guide has the code. Native iOS apps forward what the system gives them to
  the SDK (`handle…`), as the guide shows for UIKit and SwiftUI.

**Custom domain.** If the application has an **active** custom domain, every place above and
below gets it next to the link host: a second `applinks:` entry, a second `<data>` host in the
same intent filter, and both hosts in the SDK's `allowedHosts`. Never declare a custom domain
that is not active yet: on Android 11 and lower one host that fails verification turns App Links
off for all hosts of the app. Tell the user to make the domain primary in the dashboard only after
this app version is released.

**Android: App Links intent filter.** Inside the launcher `<activity>` of the app module's
`AndroidManifest.xml`, add an `<intent-filter android:autoVerify="true">` with action `VIEW`,
categories `DEFAULT` and `BROWSABLE`, and `<data android:scheme="https" android:host="<link host>" />`.
Declare the whole host, no path. Leave the existing `MAIN`/`LAUNCHER` filter and other filters
alone. Check the activity's `launchMode` against the guide (`singleTop` or `singleTask`) so a
link reaches the running activity instead of starting a second one. Expo declares the same
thing as `android.intentFilters` in `app.json`; MAUI as an `[IntentFilter]` attribute on
`MainActivity`; Unity and Unreal generate it at build time.

**Flutter: disable the built-in deep link handler,** or every link is handled twice:

- `android/app/src/main/AndroidManifest.xml`, inside the main `<activity>`:
  `<meta-data android:name="flutter_deeplinking_enabled" android:value="false" />`
- `ios/Runner/Info.plist`: `<key>FlutterDeepLinkingEnabled</key><false/>`

**Expo:** there is no Strybok config plugin to add; the SDK is pure JavaScript. The `app.json`
keys change the native projects, so the app must be rebuilt (`npx expo prebuild` or a new EAS
build), and Expo Go cannot be used to test links. If the project has an `app.config.js` /
`app.config.ts`, put the keys there instead of creating an `app.json`.

**Editor-only steps.** Unity's settings asset is created from _Tools > Strybok > Settings_ in
the Unity Editor, and some Unreal settings live in the editor's Project Settings. Do not
hand-write Unity `.asset` or `.meta` files; list these as steps for the user. Unreal's settings
are plain `.ini` entries (`Config/DefaultGame.ini`, `Config/DefaultEngine.ini`) that you may
edit as the guide shows.

### 4c. Create the SDK once, in the app's real entry point

Find where the app really starts and where its long-lived objects are created (`main()` and the
root widget; the root component or the navigation container; the `App` struct or the scene
delegate; the `Application` class and the launcher activity; `MauiProgram.cs`; …) and create
**one** instance there, the way the guide shows, with:

- the API base URL and the link host from step 0. Follow the project's existing way of handling
  configuration (an env/config module, `--dart-define`, build config fields) if it has one;
  otherwise plain constants are fine: neither value is a secret;
- an error callback (`onError` or the platform's equivalent) that goes to the project's own
  logger, not to the user interface;
- `parameterLinks` only when the owner uses links without an API call (below).

Do not create an instance per screen, and do not add a second listener for the same links with
another deep link library: that navigates twice. If the app already handles other deep links
(a custom scheme, another provider), leave that code working and let Strybok handle only its
own host; the SDK ignores URLs of other hosts.

### 4d. Route each link into the app's existing navigation

Find the router the app already uses and its list of routes: `go_router` / Navigator 2 /
`auto_route` (Flutter); React Navigation or `expo-router` (React Native); the framework's
router in a Capacitor app (Angular, React Router, Vue Router); `NavigationStack` / a
coordinator / `UINavigationController` (iOS); Jetpack Navigation or Navigation Compose
(Android); Shell (MAUI). Use what is there; never add a navigation library.

Then, in the single link listener:

1. **Match `routePath` against routes the app already has**, with an explicit allowlist or
   pattern match written for this app (for example `/invite/<code>`, `/product/<id>`). Ask the
   user which screens links should open if it is not obvious; they must use the same paths as
   `routePath` when creating links in the dashboard.
2. **Navigate only on a match.** For anything else, do nothing visible (log it) or open the
   home screen, whichever the user prefers. Never call the router with an unmatched path.
3. **Use `payload` as data only,** after checking the types you read from it.
4. **Authentication gate.** If the app has sign-in and the target screen needs a signed-in
   user, do not drop the link: keep the matched route as a pending route in the app's existing
   session/auth state and open it right after sign-in succeeds, then clear it. Reuse the app's
   own redirect mechanism (a router redirect, an auth listener). Most guides have a "Links
   that arrive before sign-in" section; the Flutter guide links to a complete example.
5. **Expo Router** resolves incoming URLs itself; use the `+native-intent` approach from the
   guide and do not also navigate from the listener. Tell the user that snippet is unverified.

Keep the handler small and in one place, next to the app's other navigation setup.

## Optional: creating links from the app

Do this only when the user asks for share links or `createLink`. It is available in every
SDK from version 0.2.0. In the three experimental SDKs (Unity, .NET, Unreal) it is as unproven
as the rest of them; the Unreal module, including this call, has never been compiled.

1. The user must switch it on in the dashboard: the application's **Setup** tab, section
   **Links from the app**: tick **Allow the app to create links** and list the **Allowed
   routes** (patterns such as `/product/*`; `*` is one path segment, a final `/**` is one or
   more). The dashboard then shows the **Client key**, which starts with `pk_`.
2. Pass that key as the SDK's `clientKey` option and call `createLink` with a route from the
   allowed patterns, from the user's Share action. Read the guide's "Create a link to share"
   section for the exact call and the typed errors, and handle them: a rate limit or a network
   failure means "try again later"; unauthorized or route-not-allowed means the key or the
   dashboard settings are wrong.
3. Say this plainly to the user: the client key is **public**. It ships inside the app and
   anyone can extract it. That is acceptable only because it can do one thing, create links for
   the allowed routes, within rate limits and a daily cap; a wide pattern such as `/**` removes
   most of that protection. It is fine to commit it like any other public configuration. A
   secret `lb_…` key is never accepted by `createLink` and must never be used here.
4. Do not put personal data in a route or payload: whoever receives the link can read them.

## Optional: campaign values and deferred links

Do either only when the user asks. The platform guide is the source of every name; if the
installed SDK version lacks the API, say so and stop.

**Campaign values (UTM).** From SDK 0.3.0 (native iOS, native Android and Kotlin Multiplatform:
0.4.0) every delivered link has a `campaign` object (source, medium, campaign, term, content)
taken from the link's defaults and the `utm_*` parameters of the tapped URL; the guide shows how
to read it. They are labels chosen by whoever shared the link: use them for analytics, never
for access, discounts or anything that matters.

**Deferred links.** Read the guide's "Deferred links" section completely, and the sections on
links tapped before the app is installed in [reference/android.md](reference/android.md) and
[reference/ios.md](reference/ios.md), before planning. Then:

1. Repeat the limits from step 2 and get an explicit "go ahead". Nothing of this has run on a
   device; never write that it works.
2. The owner switches it on per application in the dashboard (the guides name the settings);
   you do not change the dashboard.
3. **Android:** set the SDK's `deferredLinks` option and the referrer sources the guide shows,
   and add the store's install-referrer library the guide names. SDKs without native code
   (Flutter, React Native, Capacitor, JavaScript, Unity, .NET, Unreal) need the app to supply the
   referrer string, and some a persistent store: use what the guide describes, and do not write
   native bridge code unless the user asks for it.
4. **iOS:** by default only the pasted-link flow: a paste button the user designs into
   onboarding, passing the text to `redeemPastedLink`. Never read the clipboard yourself.
5. **Probabilistic matching (iOS): never enable it on your own initiative.** Only on an explicit
   request, after telling the user that Apple's rules forbid it, that the App Review and legal
   risk is theirs, that it can hand one person another person's link, and that the service they
   use may not offer it. It also needs a per-link permission in the dashboard.
6. If the app must ask for consent before reading the install referrer, leave the option off
   and call the guide's `enableDeferredLinks` after consent. Tell the user that their privacy
   policy and store data forms are affected; the guides say how.
7. Treat a deferred link like any other in the route handler (step 4d); it is marked
   `isDeferred`. Grant nothing on a link marked `isProbabilisticMatch`.

## Optional: links without an API call

Do this only when the owner uses path links or Firebase long links (the dashboard's **Setup** tab,
section **Links without an API call**; `parameterLinks` in the integration answer is not `null`),
for example when the user is moving off Firebase Dynamic Links. The owner configures the routes
and the payload keys in the dashboard; you do not change the dashboard.

1. Check that the installed SDK has the option: native iOS, native Android and Kotlin
   Multiplatform from 0.5.0, every other SDK from 0.4.0. Older versions open the app but drop
   these links (only one-segment links such as `/promo` arrive, without payload): say so.
2. Pass the owner's payload keys, in the dashboard's order, as the SDK's `parameterLinks` option,
   exactly as the guide's "Links without an API call" section shows. The SDK sends Strybok only
   those query keys; never add other keys, and never keys that could carry personal data.
3. The links arrive through the same listener as every link, with `routePath` (the route the
   server read, for example `/product/42`) and a `payload` of strings: handle them in step 4d's
   route table like any other route, treating the values as untrusted input.
4. Tell the user that the keys are compiled into the app: when the owner changes them, the app
   needs a release before new keys arrive. Deferred links are not offered for these links.

**Fallbacks by country** (sending visitors from one country to a different store or page) are a
dashboard setting (**Setup** tab, **Fallbacks by country**, also available per link). They act
in the browser before the app opens; there is no SDK code for them. If the user asks, point
them to the dashboard.

## Step 5: verify what can be verified

1. **Static checks and tests: the project's own commands.** Prefer the scripts the project
   defines (`package.json` scripts, a Makefile, CI workflow steps). Typical ones:

   | Platform             | Commands                                                                                                                           |
   | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
   | Flutter              | `flutter analyze`, `flutter test`, and if the user agrees `flutter build apk --debug`                                              |
   | React Native / Expo  | the project's type-check, lint and test scripts (often `npx tsc --noEmit`, `npm run lint`, `npm test`)                             |
   | Capacitor            | the web app's type-check/lint/test/build scripts, then `npx cap sync`                                                              |
   | Native iOS           | `swift build` for a package; otherwise `xcodebuild -list`, then `xcodebuild build` for the app scheme with a simulator destination |
   | Native Android / KMP | `./gradlew :<app module>:assembleDebug`, the module's `lint` and unit `test` tasks                                                 |
   | .NET MAUI            | `dotnet build` for the app project                                                                                                 |
   | Unity / Unreal       | nothing reliable from a terminal: the user builds in the editor                                                                    |

   Report failures as they are. If something failed before your change too, say so rather than
   fixing unrelated code.

2. **A test for the route mapping,** if the project has tests: one that feeds a known and an
   unknown `routePath` to your handler. The guides' "Testing your integration" sections show
   the SDK's fake link sources.

3. **Association files** (needs network; skip and say so if you have none):

   ```sh
   curl -fsS -i "https://<link host>/.well-known/apple-app-site-association"
   curl -fsS -i "https://<link host>/.well-known/assetlinks.json"
   ```

   Expect `200`, `Content-Type: application/json`, no redirect. The first must contain
   `<TEAM ID>.<bundle id>` in `appIDs`, with the bundle id the project really builds with; the
   second must contain the project's package name and the fingerprint(s) the user gave. An
   empty `details` array or `[]` means the identifiers are still missing in the dashboard. A
   mismatch between these files and the project is the most common reason links open the
   browser: report exactly what differs.

4. **Re-read your own diff** (`git diff` if the project uses git): only intended files changed,
   the link host is spelled identically everywhere (entitlement, manifest, SDK configuration),
   and no `lb_` key is anywhere in it.

**What you cannot verify.** Whether iOS or Android actually hands a link to the app is decided
on the device, after the OS fetched the association file (iOS goes through Apple's CDN, which
can lag by hours). A passing build, passing tests and correct association files do not prove
it. Never write "deep links work"; write what was checked.

## Step 6: summarize

End with a short report:

- **Platform and SDK version** integrated.
- **Files changed**, one line each.
- **Checks run** and their results; checks you could not run and why.
- **Left for the user**, as a checklist. Include what applies:
  - enter or confirm the identifiers in the Strybok dashboard (**Setup** tab), including the
    SHA-256 fingerprint of the Play App Signing key and of any debug key used for testing;
  - enable the **Associated Domains** capability for the App ID in the Apple Developer portal
    (Xcode's automatic signing does it; with manual signing regenerate the provisioning
    profile);
  - any Xcode, Unity Editor or Unreal Editor step you did not perform;
  - rebuild and reinstall the app (native configuration changed);
  - create a test link in the dashboard whose route is one the handler accepts;
  - test on a **physical device**: tap the link in Notes or Messages (not the browser's address
    bar), with the app closed, in the background and in the foreground, and signed out if the
    app has sign-in; on Android check `adb shell pm get-app-links <package>` shows `verified`.
    Offer the scenarios from [reference/manual-test-plan.md](reference/manual-test-plan.md).
- **Limits:** deferred deep linking is off unless it was asked for, and unverified on devices
  either way (iOS has no automatic way); the SDK's maturity note from step 2; any place where
  the guide and the installed SDK disagreed.

Do not commit. Offer to, if the user wants.
