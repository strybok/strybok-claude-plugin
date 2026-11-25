# Strybok plugin for Claude Code

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](./LICENSE)

A [Claude Code](https://claude.com/claude-code) plugin with one skill, `/strybok:integrate`, that
adds [Strybok](https://strybok.com) deep links to your app for you instead of you copying
snippets from the guides: it detects the platform, installs the SDK, does the native
configuration for iOS Universal Links and Android App Links, routes each link into the
navigation your app already has, optionally adds share-link creation, and checks what can be
checked from a terminal.

Supported projects: Flutter, React Native and Expo, Capacitor and Ionic, native iOS (Swift),
native Android (Kotlin / Java), Kotlin Multiplatform, and, as experimental SDKs, Unity, .NET MAUI
and Unreal Engine.

> **Status:** version 0.2.0. The skill has been checked against the format rules and against
> Strybok's own example app, but it has **not yet been exercised on third-party projects**.
> Review the plan it shows and the diff it produces. No Strybok SDK has been verified on physical
> devices yet. Deferred deep linking (restoring a link after a store install) is optional, off
> by default and not verified on a device; the skill only adds it when you ask, and on iOS there
> is no automatic way.

## Install

In Claude Code:

```text
/plugin marketplace add strybok/strybok-claude-plugin
/plugin install strybok@strybok
```

Or from a shell:

```sh
claude plugin marketplace add strybok/strybok-claude-plugin
claude plugin install strybok@strybok
```

Without the plugin system, copy the skill folder into your personal or project skills (the
command is then `/integrate`):

```sh
git clone https://github.com/strybok/strybok-claude-plugin
cp -R strybok-claude-plugin/skills/integrate ~/.claude/skills/integrate          # all projects
cp -R strybok-claude-plugin/skills/integrate <project>/.claude/skills/integrate  # one project
```

## Use

Open your app's project in Claude Code and run:

```text
/strybok:integrate
```

You can pass what you already know, for example `/strybok:integrate ios my-app.strybok.link`.
Claude also picks the skill up by itself when you ask for something like "add Strybok deep
links to this app" or "replace Firebase Dynamic Links with Strybok".

Have these ready (the skill asks for what it cannot find in the project):

- the link hostname of your application, from the **Setup** tab in the Strybok dashboard
  (`<name>.strybok.link`);
- for iOS: the Apple Team ID and the bundle identifier;
- for Android: the package name and the SHA-256 fingerprints of your signing certificates
  (for a Play release, the App signing key certificate from Play Console).

## What it changes in your project

Always after showing a plan and asking you first:

- **Dependencies:** adds the Strybok SDK with your project's own package manager (`pubspec.yaml`,
  `package.json`, `Package.swift`, Gradle files or the version catalog, `.csproj`,
  `Packages/manifest.json`, or a plugin folder for Unreal).
- **iOS configuration:** the Associated Domains entitlement (`applinks:<your link hostname>`).
  If the app has no entitlements file, it creates one and references it from the app target's
  build settings in `project.pbxproj`. For Expo it edits `app.json` / `app.config.js` instead.
- **Android configuration:** an `autoVerify` intent filter for your link hostname in
  `AndroidManifest.xml` (or the Expo / MAUI equivalent).
- **Flutter only:** the two flags that switch off Flutter's built-in deep link handling.
- **App code:** creates the SDK once in your app's entry point and adds one handler that maps
  each link's route onto routes your app already has, including keeping a link until after
  sign-in if your app has an auth gate. Optionally a `createLink` call for a Share button.
- **Tests:** a small test for the route mapping, when your project already has tests.

It also runs your project's analyzer, build and tests, and fetches the two association files of
your link hostname (`/.well-known/apple-app-site-association`, `/.well-known/assetlinks.json`)
to compare them with your project.

## What it never does

- It never asks you to paste a secret Strybok API key (`lb_…`) and never writes one into your
  project. The only key that goes into app code is the publishable client key (`pk_…`), and
  only if you ask for link creation; that key is public by design.
- It never touches signing keys, keystores, certificates or provisioning profiles.
- It never commits, pushes, or changes anything in your Strybok account unless you ask.
- It never navigates to a route your app does not define: link routes are treated as untrusted
  input.
- It does not claim that links work on a device. iOS and Android decide that on the device, so
  the skill ends with a checklist of what you still have to do by hand: enable Associated
  Domains for the App ID in the Apple Developer portal, enter your identifiers and signing
  fingerprints in the Strybok dashboard, and test on a physical device.

If you give it the _name_ of an environment variable that holds a secret API key with the
`apps:read` scope, it may run one read-only request, `GET /v1/apps/<appId>/integration`, to
fetch your link hostname and setup status. The key's value is never printed or stored. This is
optional; without it the skill simply asks you for the values.

Plugins run with your privileges and are not vetted by Anthropic. The whole skill is two kinds
of plain-text files you can read before installing:

- [`skills/integrate/SKILL.md`](./skills/integrate/SKILL.md): the instructions Claude follows;
- [`skills/integrate/reference/`](./skills/integrate/reference/): the Strybok integration guides
  it reads for your platform, the same ones the SDK repositories publish.

There are no scripts, hooks, MCP servers or network calls other than the ones named above.

## About this repository

This repository is generated from Strybok's main repository; do not edit it by hand. The guides
under `skills/integrate/reference/` are copied in when it is generated, so in the source tree
that folder does not exist. Changes are listed in the [changelog](./CHANGELOG.md).

## Licence

Apache 2.0, see [LICENSE](./LICENSE).
