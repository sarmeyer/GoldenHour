# Deferred Work

## Deferred from: code review of 2-4-llm-tip-augmentation-full-accessibility (2026-05-18)

- HTTP status code not checked in `fetchLLMTip` — Anthropic error bodies fail `AnthropicResponse` decode → absorbed in catch → nil, so behavior is correct; add explicit status guard if richer error differentiation is needed
- LLM tip cache key omits `windowType` — matches AC2 spec exactly; a future story could expand the key if per-window tips are desired
- `spot.lastVisited = .now` fires on every sheet open including quick-dismiss — intentional FR19 behavior; revisit if visit tracking semantics need a minimum dwell time

## Deferred from: code review of 2-3-static-shooting-tips (2026-05-18)

- Shutter speed parser: "1s" produces "1 seconds" rather than "1 second" — grammatically wrong; none of the current JSON entries use "1s" but add singular/plural handling if they ever do
- "For tonight's conditions" hardcoded string — will be wrong for morning golden hour use; revisit if morning sessions are added to v1 scope
- Note TextEditor placeholder `Text` not marked `accessibilityHidden(true)` — VoiceOver may read it alongside the empty TextEditor; pre-existing from 2.2

## Deferred from: code review of 2-2-spot-management-save-edit-delete (2026-05-17)

- `ObjectIdentifier` annotation sync and deleted-spot onDisappear write — pre-existing from 2.1; local SwiftData context handles both gracefully
- Long-press hit-test ascends only 1 superview — correct for current `SpotAnnotationView` hierarchy (1 UIView subview); add deeper traversal if annotation view gains nested subviews
- Two `.sheet(item:)` modifiers can compete if `onSaveAt` fires while SpotDetailSheet is open — unlikely in single-user app with 0.5s press threshold; guard by dismissing current sheet before allowing new action if this surfaces
- Deleted Spot accessed in SpotDetailSheet `.onDisappear` — SwiftData pending-delete objects handle property writes gracefully before save; would need an explicit guard if context is reset aggressively
- Tile overlay late API key (if key written to Keychain after MapView first appears) — pre-existing from 2.1
- `MapCoordinate.id` fresh UUID on construction — rapid replacement prevented by 0.5s long-press minimum

## Deferred from: code review of 2-1-map-display-spot-data-foundation (2026-05-17)

- `ObjectIdentifier` stability for SwiftData annotations — reliable for local-only context; revisit if CloudKit sync added; migrate to `persistentModelID` if object identity becomes unstable
- Maptiler API key embedded as URL query parameter — visible in network logs; acceptable for personal non-App-Store tool; use HTTP proxy or header injection if key ever needs protecting
- `accessibilityLabel` setter no-op on `SpotAnnotationView` — intentional; computed getter reads from `annotation` correctly after MapKit updates the property
- Spot coordinate mutations not propagated to live annotation — Story 2.2 edit flow will need to update annotation coordinates or use `@objc dynamic` observation
- Lat/lon validation on `Spot` model — coordinates sourced from CLLocationManager and MapKit tap gestures, both always valid; add bounds check if manual entry ever added
- Shadow rasterization not enabled on `SpotAnnotationView` — `shouldRasterize = true` on annotation view layer reduces compositing cost with many annotations; add if performance degrades
- Maptiler attribution shown even when API key absent / Apple tiles render — personal tool always has key; fix if app ever runs without Maptiler key by tracking overlay presence
- `StaticTips.json` not yet created — required by Story 2.3; `TipContent` struct is ready

## Deferred from: code review of 1-4-home-screen-verdict-display-visual-design (2026-05-17)

- `selectedTab` raw Int binding — a `TabItem` enum would make unhandled indices a compile error; defer until tabs are finalised
- `pillText` formatting logic duplicated in `TimePill` and `HomeView.tomorrowText` — extract to `LightWindows` extension for single source of truth
- `StaleDataLabel.hoursAgo` clamps to `max(1,…)` — benign given VerdictView's 3h gate; review if component is reused elsewhere
- `.accessibilityElement(children: .combine)` on normalContent VStack excludes `ConditionImageView` — ZStack layer separation makes combining impractical; separate VoiceOver element is acceptable
- `.id(conditionState)` forces image view recreation for transition — standard SwiftUI pattern; monitor for layout glitches on real hardware
- `GeometryReader` for scrim — `containerRelativeFrame` (iOS 17+) is an alternative but AC3 spec explicitly requires GeometryReader
- Empty `" "` verdict placeholder — `.frame(minHeight:)` or `.redacted(reason:.placeholder)` are cleaner; acceptable for v1
- `ContentView default:` silently renders MapView for any out-of-range index — safe with 2 tabs; add enum if tabs expand
- `requestLocation()` called on every HomeView appearance — idempotent in practice; consider `.task(id: appState.gpsState)` if performance becomes a concern
- String-based `Color("Name")` — type-safe `ShapeStyle` extension would surface typos at build time; acceptable for personal tool
- `TimePill` blank when `SolarService.calculate` returns nil (polar night) — polar latitudes out of v1 scope; add "No light window today" fallback if needed
- Verdict uses `.white` rather than a "warm white" named asset — spec wording was descriptive intent; add `WarmWhite` color if visual review reveals contrast issues
- `ConditionImageView` accessibility label not inside the combined VoiceOver group — ZStack architecture prevents combining; separate image description element is VoiceOver-navigable and acceptable
- No synchronous `weatherEntry` cache preload in `AppState.init` — async chain completes in <500ms for cache hits; skeleton shows correctly per AC8; add preload if launch-time performance regression observed

## Deferred from: code review of 1-3-weather-fetch-condition-mapping-verdict-logic (2026-05-17)

- Redundant stale-cache decode when `forceRefresh=true` — the staleCache closure re-reads UserDefaults even though step 1 already decoded it; minor inefficiency, not a correctness issue
- WMO code gaps (e.g. 97, 98 — thunderstorm with heavy hail variants) fall to `.unknown` silently — Open-Meteo uses standard codes in practice; `.unknown` is the correct fallback for anything unrecognized
- `UserDefaults` encode failure swallowed silently — `try? JSONEncoder().encode(entry)` discards error; next call will re-fetch; practically impossible for these value types
- `windows == nil` treated same as "out of window" in verdict — acceptable; strings are sensible for GPS-unavailable state
- No explicit URLSession timeout (defaults to 60s) — errors are absorbed and stale cache returned; spec-compliant; tighten if LLM-like timeout UX is desired
- isStale 3600 hardcoded — pre-existing observation; extract to named constant if threshold ever needs to change
- Verdict strings assume evening golden hour — morning golden hour not in v1 scope; revisit if app ever adds morning sessions

## Deferred from: code review of 1-2-solar-calculation-location-services (2026-05-17)

- blueEnd/goldenEnd may collapse at extreme latitudes — Sunlight returns nil for polar extremes; only a concern if app expands to polar regions
- `.granted`/nil fires before first fix; brief "normal content with empty verdict" state — UX can be tightened by tracking "awaiting first fix" separately from "granted"
- Weather cache is coordinate-blind — stale weather from previous location served within 60-min TTL after 500m move; acceptable for single-city personal use
- Concurrent `refresh()` Tasks race on rapid location updates — add stored cancellable task handle if this causes visible issues in practice
- `lastCalculatedCoordinate` set before solar calc; polar nil result blocks 500m retries — add retry on nil result if polar use cases arise
- 500m threshold duplicated as magic number in `LocationService.distanceFilter` and `AppState.handleLocationUpdate` — extract to a shared constant if either value changes
- No performance test for 100ms solar calculation NFR — add `measure {}` block if regression risk grows
- `startUpdatingLocation` called again on app foreground re-entry — add `isUpdatingLocation` guard if duplicate events surface
- `tomorrow` fallback to `.now` in `handleLocationUpdate` — unreachable; remove or make it a precondition failure in debug builds

## Deferred from: code review of 1-1-project-initialization-security-foundation (2026-05-17)

- `withTimeout` race at exact boundary — timeout task can win `group.next()` at the precise instant the operation completes, returning nil spuriously; negligible probability on 4s LLM timeout but technically violates "exceeds duration" spec contract
- API keys compiled into binary — static literals in `APIKeys.swift` are recoverable via binary inspection; intentional tradeoff for v1 personal tool; consider server-provisioned keys or xcconfig for any future distribution
- Key rotation requires reinstall — `bootstrapIfNeeded` only writes keys once; no path to update a stored key without Keychain wipe or reinstall
- No `kSecAttrAccessible` on Keychain items — defaults allow iCloud/device backup inclusion; acceptable for personal tool but would need `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` for production distribution
- No Keychain access group — any future app extension could read API keys without authorization; set `kSecAttrAccessGroup` if extensions are ever added
- `delete`-then-`add` Keychain write pattern — non-atomic; prefer `SecItemUpdate` with fallback to `SecItemAdd` on `errSecItemNotFound` if Keychain reliability becomes a concern
- `Formatters.shortTime` locale captured at static init; stale after mid-session locale change — consider `Locale.autoupdatingCurrent` or recreation on `NSCurrentLocaleDidChange`
- `GoldenHourApp` uses `Item.self` placeholder in ModelContainer — will be replaced with `Spot.self` in Story 2.1
- `fatalError` on ModelContainer creation — could be replaced with a recoverable error state (store deletion + recreation) for any future App Store distribution
- `handleLocationUpdate` spawns unstructured `Task`; rapid location changes can create concurrent refreshes that race on `weatherEntry` — store a cancellable task handle if this causes issues in practice
- `LightWindows` no start < end validation — Sunlight library doesn't produce inverted dates for normal geolocations; add if polar/DST edge cases surface
- `isStale` computed on `timeIntervalSince` which is negative if `fetchedAt` is in the future — false-fresh behavior acceptable; consider `max(0, interval)` if clock skew becomes a concern
- `refresh()` no-ops when `lastCalculatedCoordinate` is nil — correct guard behavior; add user-visible feedback if a manual refresh gesture is added later
