# Story 1.3: Weather Fetch, Condition Mapping & Verdict Logic

Status: done

## Story

As a photographer using Golden Hour,
I want the app to fetch current weather and interpret it photographically,
So that I receive a confident go/no-go verdict in opinionated photographic language rather than raw meteorological data.

## Acceptance Criteria

1. **Given** `WeatherResponse.swift` is implemented as a `Decodable` struct
   **When** decoded from Open-Meteo JSON (nested under `"current"` key)
   **Then** `cloud_cover`, `precipitation_probability`, `visibility`, `weather_code` map correctly to camelCase Swift properties via `CodingKeys`

2. **Given** `WeatherCacheEntry.swift` is updated to a complete `Codable` struct
   **When** `fetchedAt` is more than 3600 seconds ago
   **Then** `isStale` computed property returns `true`

3. **Given** `WeatherService.fetch(latitude:longitude:forceRefresh:)` is called and a cache entry with age < 60 minutes exists in `UserDefaults`
   **When** `forceRefresh` is `false` (default)
   **Then** the cached `WeatherCacheEntry` is returned without a network call (NFR5)

4. **Given** the cache is stale or absent and the Open-Meteo API request succeeds
   **When** `WeatherService.fetch` completes
   **Then** the decoded response is wrapped in a new `WeatherCacheEntry(response:fetchedAt:.now)`, stored in `UserDefaults` under `UserDefaultsKey.weatherCache`, and returned
   **And** the method never throws to callers — errors are absorbed internally (NFR10)

5. **Given** the API request fails and a stale cached entry exists
   **When** `WeatherService.fetch` completes
   **Then** the stale cached entry is returned and `AppState.weatherEntry` reflects it with its original `fetchedAt` timestamp

6. **Given** `ConditionMapper.map(_ response: WeatherResponse) -> ConditionState` is implemented
   **When** called with any valid `WeatherResponse`
   **Then** it returns the correct `ConditionState` for that weather data combination
   **And** every `switch` over `ConditionState` enumerates all 6 cases explicitly — no `default:` clause
   **And** any unmapped `weather_code` input returns `.unknown` — never a silent fallthrough

7. **Given** `ConditionMapper.verdict(for:windows:) -> String` is implemented
   **When** condition is positive (e.g., `.brokenClouds`) during an active light window
   **Then** the returned string is a confident, non-hedging single sentence (e.g., "Go tonight. Broken clouds — this is the good kind.")

8. **Given** `ConditionMapper.verdict` is called with a negative condition (e.g., `.overcast`)
   **When** conditions are poor for golden hour
   **Then** the verdict is honest and includes a falsifiable photographic reason (e.g., "Skip tonight. Heavy overcast flattens everything.")

9. **Given** `ConditionMapperTests.swift` exists in `GoldenHourTests`
   **When** tests run
   **Then** all valid `weather_code` ranges map to their expected `ConditionState`, boundary values are tested, and `.unknown` is returned for any unrecognized input

10. **Given** `AppState` has valid GPS coordinates and calls `refresh()`
    **When** `WeatherService.fetch` succeeds and `ConditionMapper.map` runs
    **Then** `AppState.conditionState`, `AppState.verdict`, and `AppState.weatherEntry` are updated on `@MainActor`

11. **Given** the user triggers a manual weather refresh via a UI control
    **When** `AppState.refresh(forceRefresh: true)` fires
    **Then** `WeatherService` ignores the current cache TTL and makes a fresh Open-Meteo API call

## Tasks / Subtasks

- [x] **Task 1: Create WeatherResponse.swift**
  - [x] Create `GoldenHour/Models/WeatherResponse.swift`
  - [x] Define `struct WeatherResponse: Decodable` with four properties (cloudCover: Int, precipitationProbability: Int, visibility: Double, weatherCode: Int)
  - [x] Add `CodingKeys` enum mapping snake_case JSON keys to camelCase Swift properties
  - [x] Verify struct compiles with no network dependency

- [x] **Task 2: Update WeatherCacheEntry.swift (completing the Story 1.2 stub)**
  - [x] Add `let response: WeatherResponse` as first stored property to `WeatherCacheEntry`
  - [x] Confirm `isStale` computed property (>3600s threshold) is preserved exactly as-is
  - [x] Confirm `WeatherCacheEntry: Codable` still synthesizes — no custom coding needed

- [x] **Task 3: Create WeatherService.swift**
  - [x] Create `GoldenHour/Services/WeatherService.swift`
  - [x] Define `struct WeatherService` (not an actor — matches SolarService pattern as a pure static service)
  - [x] Implement `static func fetch(latitude: Double, longitude: Double, forceRefresh: Bool = false) async -> WeatherCacheEntry?`
  - [x] Cache read: decode `WeatherCacheEntry` from `UserDefaults.standard.data(forKey: UserDefaultsKey.weatherCache)` using `JSONDecoder`
  - [x] Return cached entry if `!entry.isStale && !forceRefresh`
  - [x] Build Open-Meteo URL (see Dev Notes for exact URL template)
  - [x] Make `URLSession.shared.data(from:)` GET request
  - [x] Decode outer `OpenMeteoResponse` wrapper → extract `.current` → `WeatherResponse`
  - [x] Construct `WeatherCacheEntry(response: decoded, fetchedAt: .now)` and persist via `JSONEncoder` to `UserDefaults`
  - [x] On any error: return stale cached entry if available, else return `nil` — never throw

- [x] **Task 4: Create ConditionMapper.swift**
  - [x] Create `GoldenHour/Services/ConditionMapper.swift`
  - [x] Define `enum ConditionMapper` (caseless enum as namespace, matching project patterns)
  - [x] Implement `static func map(_ response: WeatherResponse) -> ConditionState` using WMO code table (see Dev Notes)
  - [x] Implement `static func verdict(for condition: ConditionState, windows: LightWindows?) -> String`
  - [x] Verdict switch must enumerate all 6 ConditionState cases — no `default:`
  - [x] Verdict strings must be confident, photographic, non-hedging (see Dev Notes for reference strings)

- [x] **Task 5: Update AppState.swift**
  - [x] Add `@ObservationIgnored private var lastWeatherCoordinate: CLLocationCoordinate2D?` to track the coordinate used for the last weather fetch
  - [x] Add `func refresh(forceRefresh: Bool = false) async` method on AppState
  - [x] Inside `refresh()`: guard against nil lastCalculatedCoordinate, call `WeatherService.fetch`, update `weatherEntry`, map and update `conditionState` and `verdict`
  - [x] Update `handleLocationUpdate` to store the coordinate and call `refresh()` after solar calculations

- [x] **Task 6: Create ConditionMapperTests.swift**
  - [x] Create `GoldenHourTests/ConditionMapperTests.swift`
  - [x] Test `map()`: all WMO code ranges return their expected `ConditionState` (see boundary values in Dev Notes)
  - [x] Test `map()`: an unrecognized weather_code (e.g., 999) returns `.unknown`
  - [x] Test `verdict()`: positive conditions (clearSky, brokenClouds) with active windows return "Go" verdicts
  - [x] Test `verdict()`: negative conditions (overcast, rain) return "Skip" verdicts
  - [x] Test `verdict()`: `.unknown` produces a non-crashing string output
  - [x] Run full test suite to confirm no regressions from WeatherCacheEntry struct change

## Dev Notes

### Architecture Compliance — READ FIRST

**Service boundary rules (from architecture.md):**
- `WeatherService` fetches Open-Meteo + manages UserDefaults cache. It must NOT map to `ConditionState`.
- `ConditionMapper` maps `WeatherResponse` → `ConditionState` + verdict string. It must NOT touch network or persistence.
- Services never hold `AppState` references. Results flow through AppState via return values.
- `WeatherService.fetch` absorbs all errors — never propagates `throws` to callers (NFR10).

**No `default:` in ConditionState switches** — architecture.md explicitly forbids this. Every switch must enumerate all 6 cases: `.brokenClouds`, `.clearSky`, `.overcast`, `.fog`, `.rain`, `.unknown`.

**UserDefaults key** — always use the constant, never a hardcoded string:
```swift
// CORRECT
UserDefaults.standard.set(data, forKey: UserDefaultsKey.weatherCache)

// WRONG — never do this
UserDefaults.standard.set(data, forKey: "weatherCache")
```

### WeatherResponse — Struct Design & JSON Wrapping

The Open-Meteo endpoint nests current conditions under a `"current"` key. The outer container is private to WeatherService:

```swift
// GoldenHour/Models/WeatherResponse.swift
struct WeatherResponse: Decodable {
    let cloudCover: Int
    let precipitationProbability: Int
    let visibility: Double
    let weatherCode: Int

    enum CodingKeys: String, CodingKey {
        case cloudCover               = "cloud_cover"
        case precipitationProbability = "precipitation_probability"
        case visibility               // identical — no rename needed
        case weatherCode              = "weather_code"
    }
}
```

WeatherService uses a private wrapper to unwrap the `"current"` layer:
```swift
// Private to WeatherService.swift — not exported
private struct OpenMeteoResponse: Decodable {
    let current: WeatherResponse
}
```

### Open-Meteo API URL

```
https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current=cloud_cover,precipitation_probability,visibility,weather_code
```

No API key required. Build with `URLComponents` to ensure proper URL encoding of coordinates:

```swift
var components = URLComponents(string: "https://api.open-meteo.com/v1/forecast")!
components.queryItems = [
    URLQueryItem(name: "latitude",  value: String(format: "%.4f", latitude)),
    URLQueryItem(name: "longitude", value: String(format: "%.4f", longitude)),
    URLQueryItem(name: "current",   value: "cloud_cover,precipitation_probability,visibility,weather_code"),
]
let url = components.url!
```

### WeatherCacheEntry — Completing the Stub

The Story 1.2 stub in `WeatherCacheEntry.swift` must be updated:

```swift
// BEFORE (Story 1.2 stub)
struct WeatherCacheEntry: Codable {
    let fetchedAt: Date

    var isStale: Bool {
        Date.now.timeIntervalSince(fetchedAt) > 3600
    }
}

// AFTER (Story 1.3 complete)
struct WeatherCacheEntry: Codable {
    let response: WeatherResponse
    let fetchedAt: Date

    var isStale: Bool {
        Date.now.timeIntervalSince(fetchedAt) > 3600
    }
}
```

`WeatherResponse` is `Decodable`, so for `WeatherCacheEntry` to be `Codable`, `WeatherResponse` must also be `Codable` — add `Codable` conformance to `WeatherResponse` (not just `Decodable`). Synthesized conformance works without custom coding; CodingKeys already cover both directions.

### WeatherService — Cache Read/Write Pattern

```swift
struct WeatherService {
    static func fetch(latitude: Double, longitude: Double, forceRefresh: Bool = false) async -> WeatherCacheEntry? {

        // 1. Try cache first
        if !forceRefresh,
           let data    = UserDefaults.standard.data(forKey: UserDefaultsKey.weatherCache),
           let cached  = try? JSONDecoder().decode(WeatherCacheEntry.self, from: data),
           !cached.isStale {
            return cached
        }

        // 2. Stale or absent — fetch fresh
        let stale: WeatherCacheEntry? = {
            guard let data   = UserDefaults.standard.data(forKey: UserDefaultsKey.weatherCache),
                  let cached = try? JSONDecoder().decode(WeatherCacheEntry.self, from: data)
            else { return nil }
            return cached
        }()

        do {
            var components = URLComponents(string: "https://api.open-meteo.com/v1/forecast")!
            components.queryItems = [
                URLQueryItem(name: "latitude",  value: String(format: "%.4f", latitude)),
                URLQueryItem(name: "longitude", value: String(format: "%.4f", longitude)),
                URLQueryItem(name: "current",
                             value: "cloud_cover,precipitation_probability,visibility,weather_code"),
            ]
            let (data, _) = try await URLSession.shared.data(from: components.url!)
            let outer     = try JSONDecoder().decode(OpenMeteoResponse.self, from: data)
            let entry     = WeatherCacheEntry(response: outer.current, fetchedAt: .now)
            if let encoded = try? JSONEncoder().encode(entry) {
                UserDefaults.standard.set(encoded, forKey: UserDefaultsKey.weatherCache)
            }
            return entry
        } catch {
            return stale   // error absorbed; return stale cache or nil
        }
    }
}
```

### ConditionMapper — WMO Weather Code Mapping

Open-Meteo uses standard WMO weather interpretation codes. The mapping table:

| WMO Code(s) | Meaning | ConditionState |
|---|---|---|
| 0 | Clear sky | `.clearSky` |
| 1 | Mainly clear | `.clearSky` |
| 2 | Partly cloudy | `.brokenClouds` |
| 3 | Overcast | `.overcast` |
| 45, 48 | Fog and rime fog | `.fog` |
| 51, 53, 55 | Drizzle (light/moderate/heavy) | `.rain` |
| 56, 57 | Freezing drizzle (light/heavy) | `.rain` |
| 61, 63, 65 | Rain (light/moderate/heavy) | `.rain` |
| 66, 67 | Freezing rain (light/heavy) | `.rain` |
| 71, 73, 75 | Snowfall (light/moderate/heavy) | `.rain` |
| 77 | Snow grains | `.rain` |
| 80, 81, 82 | Rain showers (light/moderate/heavy) | `.rain` |
| 85, 86 | Snow showers (light/heavy) | `.rain` |
| 95 | Thunderstorm (slight/moderate) | `.rain` |
| 96, 99 | Thunderstorm with hail | `.rain` |
| any other value | Unrecognized | `.unknown` |

Implement `map()` using a switch over `weatherCode` with explicit `case` groups. Unrecognized codes fall through to a final `return .unknown` (or use `default: return .unknown` only in the outer switch on `weatherCode` — the no-default rule applies only to switches over `ConditionState` itself).

**Photographic rationale for snow → .rain:** Snow has the same photographic impact as precipitation (soft/flat/wet light, no golden hour). No separate enum case exists; `.rain` is the correct bucket.

### ConditionMapper — Verdict Strings

`verdict(for:windows:)` must return a confident photographic opinion. Reference strings (exact wording may vary; spirit must not):

| ConditionState | Active Window | No Active Window |
|---|---|---|
| `.brokenClouds` | "Go tonight. Broken clouds — this is the good kind." | "Light window coming. Broken clouds predicted — prime conditions ahead." |
| `.clearSky` | "Go. Clear sky tonight — hard golden light. Great for silhouettes." | "Clear sky later. Expect clean, direct light — good for minimalist shots." |
| `.overcast` | "Skip tonight. Heavy overcast flattens everything." | "Skip. Overcast predicted — no golden light on offer." |
| `.fog` | "Interesting conditions. Dense fog — shoot close, lean into texture." | "Fog predicted. If it holds, try abstract and close-range work." |
| `.rain` | "Skip tonight. Active precipitation — no golden light window." | "Skip. Rain predicted during your light window." |
| `.unknown` | "Conditions unclear. Check conditions on-site." | "Conditions unclear. Check on-site before committing." |

`windows` parameter (nullable LightWindows) is used to determine active vs. upcoming context:
- `windows?.currentWindow == .golden || windows?.currentWindow == .blue` → active window verdicts
- `windows != nil && currentWindow == .neither` → upcoming window verdicts
- `windows == nil` → GPS/calculation unavailable → simplify verdict (conditions-only, no timing reference)

### AppState — refresh() Integration

Add these changes to the existing AppState (do not remove or alter any Story 1.2 code):

```swift
// Add to AppState:
func refresh(forceRefresh: Bool = false) async {
    guard let coordinate = lastCalculatedCoordinate else { return }

    let entry = await WeatherService.fetch(
        latitude: coordinate.latitude,
        longitude: coordinate.longitude,
        forceRefresh: forceRefresh
    )
    weatherEntry = entry

    if let response = entry?.response {
        conditionState = ConditionMapper.map(response)
    } else if entry == nil {
        conditionState = .unknown
    }
    // Stale cache: keep existing conditionState (entry != nil but response was stale)
    verdict = ConditionMapper.verdict(for: conditionState, windows: lightWindows)
}
```

**Update `handleLocationUpdate`** — after solar calculations, kick off weather refresh:

```swift
private func handleLocationUpdate(state: GPSState, coordinate: CLLocationCoordinate2D?) {
    gpsState = state
    guard state == .granted, let coordinate else { return }

    if let last = lastCalculatedCoordinate {
        let lastCL = CLLocation(latitude: last.latitude, longitude: last.longitude)
        let newCL  = CLLocation(latitude: coordinate.latitude, longitude: coordinate.longitude)
        if newCL.distance(from: lastCL) < 500 { return }
    }
    lastCalculatedCoordinate = coordinate

    lightWindows         = SolarService.calculate(coordinate: coordinate, date: .now)
    let tomorrow         = Calendar.current.date(byAdding: .day, value: 1, to: .now) ?? .now
    lightWindowsTomorrow = SolarService.calculate(coordinate: coordinate, date: tomorrow)

    // Kick off async weather fetch — errors absorbed inside WeatherService
    Task { await refresh() }
}
```

Note: `Task { await refresh() }` is valid inside a `@MainActor`-isolated method because `refresh()` is also `@MainActor`. The Task inherits the actor context.

### ConditionMapperTests — Coverage Requirements

**Required test cases for `map()`:**

```swift
// Boundary and representative values for all WMO code groups:
testMap(weatherCode: 0,   expected: .clearSky)
testMap(weatherCode: 1,   expected: .clearSky)
testMap(weatherCode: 2,   expected: .brokenClouds)
testMap(weatherCode: 3,   expected: .overcast)
testMap(weatherCode: 45,  expected: .fog)
testMap(weatherCode: 48,  expected: .fog)
testMap(weatherCode: 51,  expected: .rain)
testMap(weatherCode: 55,  expected: .rain)   // boundary
testMap(weatherCode: 61,  expected: .rain)
testMap(weatherCode: 65,  expected: .rain)   // boundary
testMap(weatherCode: 80,  expected: .rain)
testMap(weatherCode: 95,  expected: .rain)
testMap(weatherCode: 99,  expected: .rain)   // boundary
testMap(weatherCode: 999, expected: .unknown) // unrecognized
testMap(weatherCode: -1,  expected: .unknown) // negative/unrecognized
```

Helper: build a minimal `WeatherResponse` with only `weatherCode` varying:
```swift
private func response(weatherCode: Int) -> WeatherResponse {
    WeatherResponse(cloudCover: 50, precipitationProbability: 0, visibility: 10000, weatherCode: weatherCode)
}
```

**Required test cases for `verdict()`:**
- `.brokenClouds` with active golden window → contains "Go" or "go"
- `.overcast` with active window → contains "Skip" or "skip"
- `.unknown` → returns non-empty string without crashing
- All 6 ConditionState cases must produce non-empty strings (exhaustiveness guard)

### File Locations (from architecture.md)

```
GoldenHour/
  Models/
    WeatherResponse.swift         ← NEW (Task 1)
    WeatherCacheEntry.swift       ← MODIFY: add `response: WeatherResponse` (Task 2)
  Services/
    WeatherService.swift          ← NEW (Task 3)
    ConditionMapper.swift         ← NEW (Task 4)
  App/
    AppState.swift                ← MODIFY: add refresh() + handleLocationUpdate change (Task 5)

GoldenHourTests/
  ConditionMapperTests.swift      ← NEW (Task 6)
```

No `project.pbxproj` edits needed — PBXFileSystemSynchronizedRootGroup auto-includes new Swift files in `GoldenHour/` and `GoldenHourTests/` (confirmed in Story 1.2).

### WeatherResponse Codable Note

`WeatherCacheEntry: Codable` synthesizes `encode`/`decode` only if all stored properties are `Codable`. After adding `let response: WeatherResponse`, change `WeatherResponse`'s conformance from `Decodable` to `Codable`:

```swift
struct WeatherResponse: Codable {   // ← was Decodable
```

CodingKeys work for both encoding and decoding, so no additional code is required.

### Swift 6 / Concurrency Notes

- `WeatherService` uses `static async` — no actor isolation needed since it has no mutable state.
- `URLSession.shared.data(from:)` is `async throws` — call with `try await` inside the `do/catch`.
- `AppState.refresh()` is `@MainActor` (inherited from AppState's `@MainActor` annotation) — all property updates happen on the main actor automatically.
- `Task { await refresh() }` called from `handleLocationUpdate` (also `@MainActor`) is actor-safe.
- No `@Sendable` annotations needed on WeatherService since it holds no state.

### Previous Story Context (Story 1.2)

The following patterns are established and must be preserved:

- `SolarService` — static struct, synchronous `calculate()`, no network
- `LocationService` — `@Observable @MainActor` NSObject subclass, `nonisolated` delegate methods, `Task { @MainActor [weak self] in }` dispatch pattern
- `AppState` — `@Observable @MainActor final class`, `let locationService = LocationService()` (not var), `@ObservationIgnored` for helper properties
- `LightWindows.currentWindow` — returns `.golden`, `.blue`, or `.neither` based on `Date.now`
- `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` in build settings — all types implicitly `@MainActor` unless marked `nonisolated`
- `PBXFileSystemSynchronizedRootGroup` — new Swift files added to correct directories are auto-included; no pbxproj edits for source files

---

## Dev Agent Record

### Implementation Plan

Followed the story task sequence exactly:

1. **WeatherResponse.swift** — `struct WeatherResponse: Codable` with four properties and explicit `CodingKeys` for snake_case → camelCase mapping. Conformance upgraded from `Decodable` to `Codable` so `WeatherCacheEntry` (which embeds it) can be round-tripped via `JSONEncoder`/`JSONDecoder`.

2. **WeatherCacheEntry.swift** — Added `let response: WeatherResponse` as first stored property. Synthesized `Codable` conformance covers both fields. `isStale` computed property preserved unchanged (>3600s threshold).

3. **WeatherService.swift** — Stateless `struct` with `static async fetch()`. Cache-first logic: return fresh cached entry if not stale and `forceRefresh == false`. Private `OpenMeteoResponse` wrapper decodes the outer `"current"` JSON key. All errors are absorbed; stale cache returned on failure, `nil` if no cache. URL built with `URLComponents` per architecture spec.

4. **ConditionMapper.swift** — Caseless `enum` namespace matching project conventions. `map()` uses a `switch` over `weatherCode` with WMO code groups; unrecognized codes hit `default: return .unknown` (the no-default rule applies only to switches over `ConditionState`, not over raw `Int`). `verdict()` switches exhaustively over all 6 `ConditionState` cases with no `default:`, returning confident photographic single-sentence strings.

5. **AppState.swift** — Added `refresh(forceRefresh:)` async method and updated `handleLocationUpdate` to call `Task { await refresh() }` after solar calculations. All property updates remain on `@MainActor` via inherited isolation.

6. **ConditionMapperTests.swift** — 20+ test cases covering all WMO code boundary values, gap codes, negative codes, unrecognized codes, active/future/nil window verdicts, and exhaustiveness guard for all 6 `ConditionState` cases.

### Debug Log

No issues encountered. All implementation matched story spec precisely. Tests passed on first run.

### Completion Notes

All 6 tasks complete. Full acceptance criteria satisfied:
- AC1: `WeatherResponse` decodes correctly from Open-Meteo JSON via `CodingKeys`
- AC2: `WeatherCacheEntry.isStale` returns true after 3600 seconds
- AC3: Cache hit path returns without network call when entry is fresh
- AC4: Fresh fetch stores new `WeatherCacheEntry` to `UserDefaults`; errors absorbed
- AC5: Stale cache returned on network failure
- AC6: `ConditionMapper.map()` returns correct `ConditionState` for all WMO codes; no `default:` in ConditionState switches; unrecognized codes return `.unknown`
- AC7: Positive conditions + active window → confident "Go" verdict
- AC8: Negative conditions → honest "Skip" verdict with photographic reason
- AC9: `ConditionMapperTests` covers all WMO boundary values and `.unknown` for unrecognized input
- AC10: `AppState.refresh()` updates `conditionState`, `verdict`, and `weatherEntry` on `@MainActor`
- AC11: `refresh(forceRefresh: true)` bypasses cache TTL

---

## File List

**New files:**
- `GoldenHour/GoldenHour/Models/WeatherResponse.swift`
- `GoldenHour/GoldenHour/Services/WeatherService.swift`
- `GoldenHour/GoldenHour/Services/ConditionMapper.swift`
- `GoldenHour/GoldenHourTests/ConditionMapperTests.swift`

**Modified files:**
- `GoldenHour/GoldenHour/Models/WeatherCacheEntry.swift` — added `let response: WeatherResponse`; upgraded conformance to `Codable`
- `GoldenHour/GoldenHour/AppState.swift` — added `refresh(forceRefresh:)` method; updated `handleLocationUpdate` to kick off async weather fetch

## Change Log

| Date | Change | Author |
|---|---|---|
| 2026-05-10 | Story created, status: ready-for-dev | bmad-create-story |
| 2026-05-15 | All tasks implemented and tests passing; status: review | bmad-dev-story |
| 2026-05-17 | Code review: 3 patches applied, 7 deferred, status: done | bmad-code-review |
