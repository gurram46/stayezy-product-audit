# Module 01D — Home Return / Navigation-State Behaviour

## Test state

- Device: Motorola Edge 40
- Android: 15
- App version: 1.1.0
- Account state: anonymous / no account created
- Flow: fully loaded Home → open property details → use Android Back → observe Home
- User-observed behaviour: after returning from a property, Home visibly falls back to skeleton/loading state and takes roughly 2–3 seconds to repopulate instead of immediately restoring the previously loaded Home state.

## Finding HR-001 — Returning from property details re-runs dashboard network/state initialization

This is now **confirmed beyond a visual-only reload**.

A filtered Flutter log captured two Home → Property → Back → Home cycles. In both reproductions, returning from `Listingdetials` to `NewDashboardScreen` immediately restarts dashboard work.

### Reproduction A

- `18:44:23.796` — `Listingdetials` exits and `NewDashboardScreen` is viewed again.
- `18:44:23.830` — `provider>>>>>>>null`.
- `18:44:24.051` — Map API GET is started again.
- `18:44:24.725` — `/api/home-data?page=1&type` is called again.
- `18:44:25.138` — anonymous request again returns `Unauthorized Token`.
- `18:44:25.376` — Map API returns 200 with 300 properties.
- `18:44:26.438` — `Map API properties loaded: 300` and dashboard API timing completes.

### Reproduction B

- `18:44:39.515` — `Listingdetials` exits and `NewDashboardScreen` is viewed again.
- `18:44:39.569` — `provider>>>>>>>null`.
- `18:44:39.708` — Map API GET is started again.
- `18:44:40.286` — `/api/home-data?page=1&type` is called again.
- `18:44:40.675` — anonymous request again returns `Unauthorized Token`.
- `18:44:41.504` — `addPropertyGetData` API time is 1230 ms.
- `18:44:41.679` — API time is 1393 ms.
- `18:44:41.744` — total initialization time is 1458 ms and `_initializeData` completes.

## Interpretation

The Home skeleton seen after pressing Back is not merely a cosmetic repaint. The dashboard path is re-entering initialization and issuing fresh network work.

Most notably, **the Map endpoint is also refetched on every return to Home**, even though the user has not selected the Map tab. This repeats the same architectural concern already identified in the startup baseline: Map work is coupled to dashboard/Home lifecycle rather than to actual Map usage.

The `provider>>>>>>>null` messages immediately around the route return are also important. They suggest state/provider ownership may not be surviving navigation, but the log alone cannot prove whether this is caused by provider scope, route replacement/reconstruction, explicit clearing, or an initialization hook that runs whenever `NewDashboardScreen` resumes.

### User-facing failure scenario

A user browses a property, presses Back expecting the previous Home state, and instead sees loading skeletons while Home and Map data are initialized again. This creates avoidable latency, network usage, image churn and a perception that the app is unstable or slow.

### Expected behaviour

For a simple drill-in/drill-out navigation flow:

`Home → Property → Back`

Home should normally restore the already-loaded view immediately, preserving relevant list/scroll state and cached data. A background refresh may be performed if data is stale, but it should not blank an otherwise valid UI unless there is a strong correctness reason.

### Code-level checks once source is available

1. Inspect how `Listingdetials` is opened and closed: `push`, `pushReplacement`, custom navigation helper, or route reconstruction.
2. Inspect provider/controller scope for Home/dashboard state and why it logs `provider>>>>>>>null` on return.
3. Find the lifecycle hook that invokes `_initializeData` when `NewDashboardScreen` is shown/resumed.
4. Prevent the Home UI from resetting to skeleton when valid cached/state data already exists.
5. Decouple Google Map/platform-view and 300-property Map fetch from a simple return to Home.
6. Do not issue authenticated-only anonymous requests that predictably return `Unauthorized Token`.
7. Preserve scroll/list position when returning from property details unless product requirements explicitly specify otherwise.

**Current severity:** P1/P2 performance/UX candidate. It is directly user-visible, reproducible twice in one isolated capture, and triggers redundant Home + Map work during a common navigation path.

## Evidence

- User-supplied screen recording showing Home skeleton/repopulation after Back.
- Filtered Flutter log captured specifically for Home-return behaviour.
- `evidence/module-01-home/home-return-refetch-key-evidence.txt`
