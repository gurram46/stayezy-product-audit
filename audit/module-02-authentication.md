# Module 02 — Authentication and Authenticated Baseline

## Scope

Production Android 1.1.0 authentication review after completing the anonymous Module 01 baseline. This run used Google sign-in to create a new user account. Phone/OTP authentication has **not** yet been tested.

Raw logcat is intentionally **not committed** because the production build logs live account/session information. This document contains sanitized evidence only.

## Auth flow baseline — Google sign-in

Observed timeline from the captured run:

- `02:04:14.151` — `LoginScreen` becomes active.
- `02:04:15.946` — Google `SignInHubActivity` is created.
- `02:04:21.934` — Google sign-in activity closes and Stayezy resumes.
- `02:04:23.268` — Firebase Auth reports the Google-authenticated identity.
- `02:04:23.273` — Stayezy logs that Google user identity internally and shows a progress dialog.
- `02:04:24.655` — Firebase Auth reports another Firebase identity.
- `02:04:24.685` — authenticated state is true and navigation returns to `NewDashboardScreen`.
- `02:04:26.299` — authenticated profile data is successfully retrieved from the Stayezy backend.

Google signup therefore completed successfully with no fatal crash or visible auth blocker in this run. Human account-selection time is included between opening and closing Google's sign-in UI, so the full ~10.5 s LoginScreen-to-dashboard interval must not be treated as pure backend latency.

## Finding AUTH-001 — Live authenticated profile/session data is printed to release logcat

### Confirmed

After account creation the production build prints the full backend profile response to logcat. The response includes personally identifying account information and a live application authentication/session token. Earlier Module 01 captures also showed FCM tokens, precise location, analytics identifiers and full API responses.

### Assessment

This materially escalates Module 01 finding H-007. Logging a live application auth token is a security issue, not merely noisy debugging. Any process/person with access to captured production logs, bug reports or support dumps could potentially obtain reusable credentials depending on token lifetime and server-side controls.

### Severity

**P1 security/privacy.**

### Required correction direction

- Never log access/session/auth tokens, Firebase tokens, OTP values or authorization headers in release builds.
- Do not print complete profile/auth API responses.
- Apply structured redaction at the networking/logging layer rather than relying on developers to remember per-call redaction.
- Disable verbose analytics/network debug logging in release builds.
- Review server token lifetime, rotation/revocation and scope after determining how long the exposed token remains valid.

## Finding AUTH-002 — Firebase App Check provider is not installed during Google auth

### Confirmed

During the Google sign-in sequence Firebase repeatedly reports:

`Error getting App Check token; using placeholder token instead. Error: ... No AppCheckProvider installed.`

Four such warnings occur during the isolated auth interval.

### Assessment

This proves the client is not supplying a real Firebase App Check token for these requests. It does **not** by itself prove an exploitable authentication vulnerability, because enforcement depends on Firebase/project configuration and which Firebase products/endpoints are protected.

### Severity

**P2 security-hardening candidate** pending Firebase console/config and source review.

### Code/config checks

- verify whether Firebase App Check is intended for the production app;
- verify enforcement state for applicable Firebase services;
- if intended, configure the Android App Check provider and production attestation correctly;
- ensure debug/placeholder providers cannot be used in production.

## Finding AUTH-003 — Auth identity changes during one Google signup sequence

### Confirmed

The same Google signup produces two different Firebase Auth user identifiers about 1.4 seconds apart:

1. Google/Firebase authenticated identity A is reported at `02:04:23.268` and logged by Stayezy as the Google user.
2. Firebase Auth then reports identity B at `02:04:24.655`.
3. The Stayezy backend profile subsequently stores/reports the original Google identifier separately and returns its own internal user ID.

Identifiers are deliberately omitted from this audit document.

### Assessment

This may be intentional — for example, a Google identity being exchanged for a custom Firebase identity/session — but the transition is unusual enough to inspect. Authentication correctness depends on deterministic mapping between Google identity, Firebase identity and Stayezy backend user identity.

### Severity

**P1/P2 candidate**, not a confirmed defect until auth source/backend flow is inspected.

### Code checks

Trace the complete path:

`GoogleSignIn -> Firebase credential -> Stayezy backend login/register -> any custom Firebase token/sign-in -> persisted local session`

Verify account-linking, duplicate-account prevention, logout/re-login identity stability and authorization ownership checks.

## Finding AUTH-004 — Dashboard/rendering pressure immediately resumes after successful auth

After authentication, navigation reconstructs `NewDashboardScreen`, authenticated state becomes true and Google Maps/platform-view initialization begins again. During the ~11-second LoginScreen-through-dashboard interval there are 184 `BLASTBufferQueue` max-frame acquisition errors. This is consistent with Module 01's existing rendering/surface-pressure findings rather than a new auth-specific root cause.

The post-run graphics snapshot reports:

- 424 rendered UI frames
- 66 janky frames — **15.57%**
- 86 high-input-latency events
- 42 slow UI-thread events
- 59 slow draw-command events
- 66 missed frame deadlines
- p95 frame time 21 ms
- p99 frame time 48 ms

Caveat: this snapshot covers the broader signup/post-login session, including subsequent navigation, and must not be interpreted as an auth-only jank percentage.

## Finding AUTH-005 — Favourite service JSON parse exception occurs during dashboard initialization

Before authentication is initiated, dashboard `_initializeData` invokes `favoritepost`, which reaches `PropertyService.toggleFavourite`. The service then throws:

`FormatException: Unexpected end of input`

while JSON-decoding an empty response.

Stack path captured:

`_initializeData -> favoritepost -> PropertyService.toggleFavourite -> jsonDecode`

### Assessment

This is a confirmed unhandled service-level response-shape error. The method name `toggleFavourite` suggests a mutation-like operation, but source/API inspection is required before claiming that dashboard initialization is actually changing favourite state.

### Severity

**P2 candidate.** It is particularly relevant now that authenticated Favourites will be tested.

## Current auth coverage

Covered:

- Google account creation/sign-in
- authenticated dashboard transition
- backend profile retrieval
- release-log/security behavior

Not yet covered:

- phone/OTP login/signup
- invalid OTP / resend OTP / rate limits
- logout and re-login
- session persistence after force-stop/restart
- account deletion
- authenticated authorization boundaries

## Immediate next authenticated test

Test Favourites with the newly authenticated account before changing any account/profile state. Verify initial empty state, add/remove favourite behavior, persistence after navigation, and whether the `toggleFavourite` parse exception reproduces in a real user interaction.
