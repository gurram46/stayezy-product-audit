# Module 01D — Home Return State Preservation

## Test flow

Anonymous production-app behavioural pass:

`Home -> property listing/details -> Android back -> Home`

A screen recording supplied during the audit shows that returning to Home does not immediately restore the previously rendered Home state. Instead, the page visibly falls back to skeleton/loading placeholders and repopulates over roughly the next few seconds, including later image restoration.

## Finding H-008 — Returning from property details visibly re-enters Home loading/skeleton state

### User-visible behaviour

The recording shows:

- Home was already populated before entering the property;
- the user opened a property and browsed the details screen;
- after Android Back, Home was not restored immediately in its prior rendered state;
- Home briefly displayed skeleton/loading placeholders again;
- textual/card content returned before some imagery, recreating the same staged-load feeling seen during initial Home startup.

For a short push/pop navigation flow, the expected behaviour is normally that Home state is preserved and restored immediately: current scroll/rail position, already-loaded listing data and already-decoded/cached imagery should not unnecessarily regress to a first-load skeleton unless data invalidation is intentional.

**Current severity:** P2 UX/performance candidate.

### What is confirmed vs not confirmed

**Confirmed from visual behaviour:**

- returning from a property causes a visible Home loading-state regression;
- the user perceives it as a reload rather than an immediate state restoration.

**Not yet confirmed:**

- whether `/api/home-data` is requested again;
- whether a Home provider/controller is recreated;
- whether the widget subtree is reconstructed while data remains cached;
- whether only image providers are recreated;
- whether navigation is replacing Home rather than preserving/popping the existing route state.

Do not call this a backend reload until a filtered capture proves a new network request.

## Supplied log limitation

The later `stayezy-home-return-log.txt` does not contain the exact recorded navigation event. The screen recording began around `18:15:52`, whereas the supplied log begins at `18:18:57.505`.

The log is also almost completely consumed by the already-known BLAST surface error:

- 1,233 total lines;
- 1,219 `Can't acquire next buffer. Already acquired max frames 8 max:6 + 2` lines (~98.86%);
- the errors span approximately 95.09 seconds from `18:18:58.030` to `18:20:33.122`;
- `Choreographer` also reports `Skipped 30 frames!` immediately around the later app resume.

This capture strongly reconfirms H-003/native-surface pressure, but it cannot answer whether the Home API was called again because the relevant Flutter/API logs are absent.

## Next verification

Run a fresh Home -> property -> Back -> Home reproduction while capturing only the Flutter log tag live. This prevents BLASTBufferQueue spam from overwriting the API/controller evidence.

Inspect specifically for:

1. a second `API URL: https://admin.stayezy.co/api/home-data...` entry after Back;
2. Home/dashboard provider/controller initialization messages;
3. loading-state changes before and after route pop;
4. duplicate-request suppression messages;
5. image URL/cache activity after Home restoration.

### Likely correction depends on code evidence

If Home is being refetched/reinitialized unnecessarily, preserve the Home state above the details route and invalidate only when explicitly required. If the data remains available and only the presentation resets, keep the Home widget/state/controller alive and avoid replacing loaded content with a skeleton during route restoration. Image providers should also reuse cached state where the source URL is unchanged.

## Evidence

- Screen recording `Screen_recording_20260902_181552.mp4` supplied during the audit.
- Filtered key-log evidence committed at:
  `evidence/module-01-home-return/home-return-key-log-evidence.txt`
