# Changelog

## 0.3.0

- Links without an API call: the skill reads `parameterLinks` from the integration answer and,
  when the owner uses path links or Firebase long links, passes the payload keys to the SDK's new
  `parameterLinks` option (native iOS, native Android and Kotlin Multiplatform 0.5.0, the other
  SDKs 0.4.0). Reference guides refreshed to those versions. Unit-tested SDKs only; nothing has
  run on a device.
- Custom domains: the skill asks whether the application has an **active** custom domain (for
  example `go.example.com`) and, if so, declares it next to the link host in the entitlement, the
  intent filter and `allowedHosts`, never one that is not active yet. Reading the setup data from
  the API mentions the new `allowedHosts`, `customDomains` and `linkHostname` fields. Custom
  domains were tried end to end with one real domain; not yet on a device.

## 0.2.0

- Reference guides refreshed to the current SDKs: JavaScript, React Native, Capacitor, Flutter,
  Unity, .NET and Unreal 0.3.0; native iOS, native Android and Kotlin Multiplatform 0.4.0. Some
  of these versions may not be published yet; the skill checks what the registry offers.
- `createLink` is described for every SDK (from 0.2.0), and attestation of `createLink`
  (App Attest, Play Integrity) for the native iOS, native Android and Kotlin Multiplatform SDKs.
- New optional step for campaign values (`campaign` on every delivered link) and for deferred
  links. Both are added only when the user asks and defer to the platform guide.
- Deferred links are no longer described as "not supported": they are optional, off by default
  and **not verified on any device or against any real store**. Android uses the store's install
  referrer (Huawei and Samsung unverified); on iOS nothing is automatic, and the skill never
  enables probabilistic matching on its own initiative.
- The status table has a column saying from which version each SDK has deferred links and what
  the app has to supply itself.

## 0.1.0

- Initial version: the `integrate` skill (`/strybok:integrate`). It detects the platform, installs
  the Strybok SDK, does the native configuration, routes delivered links into the app's existing
  navigation, optionally adds link creation, and verifies what can be verified from a terminal.
  The integration guides of every SDK are included as the skill's reference material.
- Not yet exercised on third-party projects; see the README.
