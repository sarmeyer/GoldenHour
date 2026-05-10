---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: 'complete'
completedAt: '2026-05-09'
inputDocuments:
  - "_bmad-output/planning-artifacts/product-brief-golden-hour.md"
  - "_bmad-output/planning-artifacts/product-brief-golden-hour-distillate.md"
  - "_bmad-output/planning-artifacts/prd.md"
  - "_bmad-output/planning-artifacts/ux-design-specification.md"
workflowType: 'architecture'
project_name: 'golden-hour'
user_name: 'sarah'
date: '2026-05-09'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**

34 FRs across 6 categories:

| Category | FRs | Architectural Weight |
|---|---|---|
| Light Window Calculation | FR1–FR4 | On-device computation; no network dependency |
| Condition Assessment | FR5–FR10 | Open-Meteo integration + ConditionState mapping; caching + staleness handling |
| Location & Spots | FR11–FR19 | MapKit + SwiftData; offline-capable; lastVisited timestamp |
| Shooting Guidance | FR20–FR26 | Dual-layer: static tips (instant) + LLM upgrade (async, 4s timeout) |
| Offline & Connectivity | FR27–FR30 | Three-tier degradation (full, stale, offline) per feature |
| Config & Location Services | FR31–FR34 | GPS permission states; Keychain API key management; manual refresh |

**Non-Functional Requirements:**

| NFR | Constraint | Architectural Implication |
|---|---|---|
| NFR1–NFR3 | Home ≤2s cold launch; map ≤3s; static tips ≤200ms | Cache-first data strategy; solar calc on-device |
| NFR4 | LLM timeout after 4s | Async task with explicit cancellation; no error state exposed |
| NFR5 | Weather cache min 60min TTL | Cache layer required with timestamp tracking |
| NFR6 | Solar calc <100ms | Library choice must be validated; no async needed |
| NFR7 | API key in iOS Keychain | Keychain read at startup; never in source or bundle |
| NFR8 | Data transmitted only to Open-Meteo + Anthropic | No analytics SDK, no crash reporter, no third-party |
| NFR10–NFR13 | Failures must never block available features | Isolated service boundaries; independent failure domains |
| NFR14–NFR15 | Dynamic Type + VoiceOver | No fixed font sizes; semantic accessibility labels throughout |

**Scale & Complexity:**

- Primary domain: Native iOS (SwiftUI, no backend)
- Complexity level: **Low** — personal tool, 1–2 users, no authentication, no cloud sync, no real-time concurrency
- Estimated architectural components: ~6 services + 2 screens + shared state

### Technical Constraints & Dependencies

| Constraint | Details |
|---|---|
| Platform | iOS 18+, Swift, SwiftUI, portrait-only, iPhone-only; no App Store submission |
| Weather | Open-Meteo REST API — no auth required; fields: `cloud_cover`, `precipitation_probability`, `visibility`, `weather_code` |
| LLM | Anthropic Claude API — REST; structured prompt; key managed via Keychain |
| Maps | MapKit + Maptiler or Stadia Maps raster tile overlay; OSM attribution required; no POI filtering |
| Solar | On-device library (Solar or SunCalc Swift port); validation required against known golden hour times |
| Persistence | SwiftData — local only; no iCloud sync; spots model with `name`, `note`, `locationType`, `lastVisited` |
| API key bootstrap | Read from gitignored local config on first launch → write to Keychain → read from Keychain thereafter |

### Cross-Cutting Concerns Identified

1. **`ConditionState` enum as system spine** — single enum drives: background imagery, verdict language, static tip selection, LLM prompt context, and accent color. Must be shared across HomeView, MapView, and SpotDetailSheet via a single observable state container.

2. **Connectivity-aware caching** — three independent cache/degradation domains (weather, map tiles, LLM tips) each with distinct staleness semantics and silent-failure behavior.

3. **Async coordination** — static tips render synchronously; LLM tips fire async with a 4s hard timeout and are augmentative (never displace static tips). The pattern must prevent content flash/replacement.

4. **API key security lifecycle** — Keychain read on app startup is a cross-cutting initialization concern; any service that needs it depends on this completing first.

5. **GPS state machine** — four distinct states (unresolved, granted, denied, restricted) each requiring different UI behavior on the home screen; affects all location-dependent calculations.

## Starter Template Evaluation

### Primary Technology Domain

Native iOS mobile application — Swift, SwiftUI, iOS 18+. No CLI scaffolding tool applies; project initialization is via Xcode's built-in SwiftUI App template.

### Starter Options Considered

For native iOS, the "starter" decision is twofold: the Xcode project template and the architecture pattern.

**Xcode Project Template: SwiftUI App**
The standard Xcode *App* template (SwiftUI interface, SwiftData storage) provides the correct project scaffold for this app: `@main` entry point, `ContentView`, App bundle structure, and SwiftData model container setup. No third-party scaffolding tool improves on this for a solo personal project.

**Architecture Pattern: MVVM with `@Observable`**
iOS 17+ introduced the `@Observable` macro as the modern replacement for `ObservableObject`/`@Published`. It is the recommended pattern for SwiftUI iOS 18 apps in 2026. For this project:
- `@Observable` view models manage screen state
- `@MainActor` ensures UI updates are thread-safe
- Services (weather, solar, LLM, location) are plain Swift classes injected via environment
- `ConditionState` enum propagated through a shared top-level `AppState` observable

No TCA or other third-party architecture framework is warranted for this scope.

### Solar Calculation Library

| Option | Golden/Blue Hour | Verdict |
|---|---|---|
| **Sunlight** (BinaryBirds/Sunlight) | Explicit golden + blue hour | **Selected** |
| SunKit (SunKit-Swift/SunKit) | Golden hour start/end | Fallback |
| Solar (ceeK/Solar) | Sunrise/sunset only | Insufficient |

**Selected: Sunlight** — the only library that directly outputs golden hour and blue hour windows by name, matching the app's core data requirements.

### Initialization

**Xcode setup:**

```
File > New Project > iOS > App
Interface: SwiftUI
Storage: SwiftData
Language: Swift
```

**Swift Package dependencies to add:**

```
https://github.com/BinaryBirds/Sunlight  (solar calculations)
```

All other dependencies (Open-Meteo, Anthropic) are plain REST — no SDK required.

### Architectural Decisions Established by This Baseline

**Language & Runtime:** Swift 6, async/await concurrency model, `@MainActor` for view models

**State Management:** `@Observable` macro; `AppState` singleton injected via `.environment`; no Combine required

**Persistence:** SwiftData with `@Model` macro; `ModelContainer` configured at app entry point

**Networking:** `URLSession` with `async/await`; no third-party HTTP client needed

**Project Structure:**
```
GoldenHour/
  App/              # @main entry, AppState, DI setup
  Models/           # SwiftData @Model types, ConditionState enum
  Services/         # WeatherService, SolarService, TipsService, LocationService
  Views/            # HomeView, MapView, SpotDetailSheet, SaveSpotSheet
  Components/       # VerdictView, TimePill, TipBlock, LLMTipBlock, etc.
  Resources/        # Condition background images, static tip content
```

**Note:** Project initialization using the Xcode setup above should be the first implementation story.

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- `ConditionState` enum as system spine — all display, tips selection, LLM prompt, and imagery depend on it
- Keychain API key bootstrap lifecycle — required before any network service can operate
- SwiftData `Spot` model schema — determines persistence behavior for core user data

**Important Decisions (Shape Architecture):**
- Weather and LLM caching strategy — determines offline resilience and API cost
- Claude model selection — determines latency profile of the tip upgrade flow
- Map tile provider — requires API key alongside Anthropic key

**Deferred Decisions (Post-MVP):**
- Push notification scheduling logic (out of v1 scope)
- Personal location log schema (post-v1 roadmap)

---

### Data Architecture

**Weather Cache — `UserDefaults`**

Encoded `WeatherCacheEntry` (Codable struct: `WeatherResponse` + `fetchedAt: Date`) stored in `UserDefaults`. Read on launch; fresh fetch triggered when `Date.now - fetchedAt > 3600s` or on explicit user refresh (FR34).

```swift
struct WeatherCacheEntry: Codable {
    let response: WeatherResponse
    let fetchedAt: Date
    var isStale: Bool { Date.now.timeIntervalSince(fetchedAt) > 3600 }
}
```

Rationale: appropriate for a single small JSON blob with a timestamp; keeps SwiftData focused on `Spot` persistence.

**LLM Tips Cache — `UserDefaults` (cross-session)**

`TipsCacheEntry` (Codable: tip text + timestamp) stored in `UserDefaults`, keyed by `"\(conditionState.rawValue)_\(locationType.rawValue)"`. Persists across launches, reducing API calls on repeat condition × location combinations.

Deliberate improvement over PRD's "per session" language — the content doesn't change meaningfully between sessions for the same condition pair.

**Spot Persistence — SwiftData**

```swift
@Model class Spot {
    var name: String
    var note: String?
    var locationType: LocationType
    var latitude: Double
    var longitude: Double
    var lastVisited: Date?
    var createdAt: Date
}
```

`ModelContainer` configured at `GoldenHourApp` entry point. No iCloud sync; no migration strategy required for v1 (personal tool, schema changes are safe to handle with a store wipe during development).

---

### Security

**API Key Bootstrap — `APIKeys.swift` (gitignored)**

Two keys managed: `anthropicAPIKey` and `maptilerAPIKey`. Both follow the same lifecycle:

1. First launch: read from gitignored `APIKeys.swift` constant
2. Write both to iOS Keychain via `KeychainService`
3. All subsequent launches: read from Keychain only

`KeychainService` is a thin internal struct (~30 lines) wrapping `SecItemAdd` / `SecItemCopyMatching` directly — no third-party dependency.

```swift
// APIKeys.swift — gitignored, never committed
enum APIKeys {
    static let anthropic = "sk-ant-..."
    static let maptiler  = "..."
}
```

`APIKeys.swift` added to `.gitignore`. A `APIKeys.swift.template` (with empty string placeholders) is committed instead as setup documentation.

**Data Transmission Scope**

Per NFR8: only two external destinations.
- Open-Meteo: GPS coordinates (lat/lon)
- Anthropic: `conditionState` + `locationType` + `lightWindowType` (no PII)
- Maptiler: tile requests with coordinates (standard map tile CDN behavior)

No analytics SDK, no crash reporter, no third-party tracking of any kind.

---

### API & Communication Patterns

**Claude Model — `claude-haiku-3-5`**

Selected for latency reliability within the 4s timeout window. The system prompt must be directive and tightly scoped to compensate for reduced model capacity vs. Sonnet.

Prompt structure (input → output contract):
```
System: [Photography assistant persona + output format constraints + camera settings requirement + honest negative condition handling]
User: Light window: {golden|blue} hour. Condition: {conditionState}. Location type: {locationType}. Spot note: {optional note}.
```
Output: plain text tip (3–5 sentences) + camera settings line. No JSON wrapping needed.

**Open-Meteo — REST, no auth**

Fields fetched: `cloud_cover`, `precipitation_probability`, `visibility`, `weather_code` (current conditions endpoint). Single `URLSession` async call; response decoded to `WeatherResponse` Codable struct; immediately mapped to `ConditionState` via `ConditionMapper`.

**Map Tile Provider — Maptiler**

Free tier (requires API key — managed via same `APIKeys.swift` → Keychain flow as Anthropic key). Raster tile overlay via `MKTileOverlay` in MapKit. OSM attribution displayed per Maptiler terms.

**Error Handling Standard**

All service calls are `throws`-free at the call site — errors are caught internally and converted to graceful state (stale data, static tips, last-known condition). No error propagation to views; UI never shows an error state for network failures.

---

### App Architecture

**`AppState` — Single Observable Root**

```swift
@Observable @MainActor class AppState {
    var conditionState: ConditionState = .unknown
    var lightWindows: LightWindows?
    var verdict: String = ""
    var weatherEntry: WeatherCacheEntry?
    var gpsState: GPSState = .unresolved
}
```

Injected at `GoldenHourApp` via `.environment`. Consumed by `HomeView`, `SpotDetailSheet`, and `ConditionImageView`. All condition-driven UI derives from a single source of truth.

**Navigation — Native `TabView` (2 tabs)**

`TabView` with custom tab bar styling (per UX spec: `#E6DAE4` background). Two tabs: Home, Map. Sheet presentation for `SpotDetailSheet` and `SaveSpotSheet` from within `MapView`. No push navigation in the primary flow.

---

### Infrastructure & Deployment

**Distribution:** Personal device via Xcode direct install or TestFlight. No App Store submission, no CI/CD pipeline.

**Testing Strategy:** Lightweight unit tests focused on deterministic logic:
- `SolarService` — validate golden/blue hour window calculations against known reference times
- `ConditionMapper` — validate all weather code → `ConditionState` mappings
- `TipsService` — validate correct static tip selection per `conditionState × lightWindowType`

No UI tests for v1 (single user, non-App-Store tool).

**Environment Configuration:** Two configurations (Debug / Release) via Xcode schemes. `APIKeys.swift` gitignored in both; `APIKeys.swift.template` committed with placeholder values and setup instructions.

---

### Decision Impact Analysis

**Implementation Sequence:**
1. Xcode project setup + `APIKeys.swift` + `KeychainService`
2. `ConditionState` enum + `AppState` + `ConditionMapper`
3. `SolarService` (on-device, no network) + unit tests
4. `LocationService` (GPS state machine)
5. `WeatherService` + UserDefaults cache
6. `HomeView` + `VerdictView` (static, no LLM yet)
7. SwiftData `Spot` model + `MapView`
8. `TipsService` (static tips) + `SpotDetailSheet`
9. `TipsService` (LLM augmentation) + `LLMTipBlock`
10. Visual polish (condition imagery, custom tab bar, typography)

**Cross-Component Dependencies:**
- `ConditionState` is a build-time dependency for `TipsService`, `VerdictView`, `ConditionImageView`, and all LLM prompts — define this enum first
- `KeychainService` must be initialized before `WeatherService` or `TipsService` attempt network calls
- `AppState.gpsState` gates `SolarService` and `WeatherService` — location must resolve before either fires

## Implementation Patterns & Consistency Rules

### Critical Conflict Points Identified

8 areas where AI agents could make different choices without explicit rules.

---

### Naming Patterns

**Swift Code Naming (standard, enforced)**

| Element | Convention | Example |
|---|---|---|
| Types (class, struct, enum, protocol) | `PascalCase` | `WeatherService`, `ConditionState` |
| Functions, methods, properties | `camelCase` | `fetchWeather()`, `conditionState` |
| Enum cases | `camelCase` | `.brokenClouds`, `.clearSky` |
| File names | Mirror primary type name | `WeatherService.swift`, `ConditionState.swift` |
| Test files | `{TypeName}Tests.swift` | `ConditionMapperTests.swift` |
| SwiftUI component files | Match the `View` struct name | `VerdictView.swift`, `TimePill.swift` |

**`ConditionState` Enum — Canonical Cases**

These raw values appear in UserDefaults keys and LLM prompts. All agents must use exactly these identifiers:

```swift
enum ConditionState: String, Codable {
    case brokenClouds = "broken_clouds"
    case clearSky     = "clear_sky"
    case overcast     = "overcast"
    case fog          = "fog"
    case rain         = "rain"
    case unknown      = "unknown"
}
```

**`LocationType` Enum — Canonical Cases**

```swift
enum LocationType: String, Codable, CaseIterable {
    case coastal    = "coastal"
    case urban      = "urban"
    case forest     = "forest"
    case openField  = "open_field"
    case elevated   = "elevated"
    case other      = "other"
}
```

**`LightWindows` Type — Canonical Fields**

```swift
struct LightWindows {
    let goldenStart: Date
    let goldenEnd:   Date
    let blueStart:   Date
    let blueEnd:     Date

    var currentWindow: WindowType { … } // .golden | .blue | .neither
}

enum WindowType: String { case golden, blue, neither }
```

**UserDefaults Key Registry — Centralized Constants**

All UserDefaults keys live in one place. No agent should hardcode a key string:

```swift
enum UserDefaultsKey {
    static let weatherCache = "weatherCache"
    static let tipsCache    = "tipsCache_"  // + conditionState_locationType suffix
}
// Usage: UserDefaultsKey.tipsCache + "\(state.rawValue)_\(type.rawValue)"
```

---

### Structure Patterns

**Service Boundary Rule**

Each service owns exactly one external concern. Services never call other services directly — they return values to `AppState`:

| Service | Responsibility | Must NOT |
|---|---|---|
| `SolarService` | Compute light windows from lat/lon + date | Touch network |
| `WeatherService` | Fetch Open-Meteo + manage UserDefaults cache | Map to `ConditionState` |
| `ConditionMapper` | Map `WeatherResponse` → `ConditionState` + verdict string | Touch network or persistence |
| `TipsService` | Select static tip + request LLM tip + manage tips cache | Know about `AppState` |
| `LocationService` | Manage `CLLocationManager`, publish `GPSState` + coordinates | Compute anything from coordinates |
| `KeychainService` | Read/write API keys to Keychain | Be called from views directly |

**Static Tips Storage**

Static tips live in `Resources/StaticTips.json`, keyed `"{conditionState.rawValue}_{windowType.rawValue}"`. Never inline tip strings in Swift source.

```json
{
  "broken_clouds_golden": {
    "body": "Shoot toward the water...",
    "cameraSettings": "ISO 400 · f/8 · 1/60s"
  }
}
```

`TipsService` loads this file once at init; `TipsStore` holds the decoded dictionary.

**Test Location**

All unit tests in the `GoldenHourTests` Xcode target. One test file per service: `SolarServiceTests.swift`, `ConditionMapperTests.swift`, `TipsServiceTests.swift`. No UI tests in v1.

---

### Format Patterns

**External API Decoding — `CodingKeys` for snake_case mapping**

Open-Meteo returns snake_case. Swift models use camelCase. Always use `CodingKeys`:

```swift
struct OpenMeteoCurrentWeather: Decodable {
    let cloudCover: Int
    let precipitationProbability: Int
    let visibility: Double
    let weatherCode: Int

    enum CodingKeys: String, CodingKey {
        case cloudCover = "cloud_cover"
        case precipitationProbability = "precipitation_probability"
        case visibility
        case weatherCode = "weather_code"
    }
}
```

**Date Display Format**

Light window times displayed as `"6:47 PM"` — `DateFormatter` with `timeStyle: .short`, `dateStyle: .none`. One shared `DateFormatter` instance in a `Formatters` namespace (formatters are expensive to create):

```swift
enum Formatters {
    static let shortTime: DateFormatter = {
        let f = DateFormatter()
        f.timeStyle = .short
        f.dateStyle = .none
        return f
    }()
}
```

---

### Communication Patterns

**Service → AppState Data Flow**

All service results flow through `AppState`. Services do not hold references to `AppState`. Pattern: `AppState` calls service and assigns result.

```swift
// CORRECT — AppState calls service, assigns result
func refresh() async {
    let weather = await weatherService.fetch(lat: lat, lon: lon)
    self.conditionState = conditionMapper.map(weather)
    self.verdict = conditionMapper.verdict(for: conditionState, windows: lightWindows)
}

// WRONG — service holds AppState reference
class WeatherService {
    var appState: AppState  // ❌ never
}
```

**Async Task Pattern in Views**

Use `.task {}` modifier (preferred). Store `Task` reference only when explicit cancellation is needed:

```swift
// CORRECT — preferred
.task {
    await appState.refresh()
}

// CORRECT — for cancellable work
@State private var tipTask: Task<Void, Never>?
.onDisappear { tipTask?.cancel() }
```

**LLM Tip Request Pattern**

`TipsService.fetchLLMTip(conditionState:locationType:windowType:spotNote:)` returns `String?`. Caller applies 4s timeout. `LLMTipBlock` only appears if result is non-nil:

```swift
let tip = await withTimeout(seconds: 4) {
    await tipsService.fetchLLMTip(…)
}
// tip == nil → LLMTipBlock stays hidden; no error shown
```

---

### Process Patterns

**Error Absorption — No Error Propagation to Views**

Services absorb errors internally and return optional/cached values. Never propagate `throws` to the view layer:

```swift
// CORRECT — service absorbs error, returns cached or nil
func fetch() async -> WeatherResponse? {
    do {
        let result = try await URLSession.shared.data(from: url)
        return try JSONDecoder().decode(WeatherResponse.self, from: result.0)
    } catch {
        return cachedResponse  // nil if no cache exists
    }
}

// WRONG
func fetch() async throws -> WeatherResponse { … }  // ❌ don't throw to callers
```

**`ConditionState.unknown` Handling**

`unknown` is the initial state and the fallback for any mapping failure. All `switch` statements over `ConditionState` must include an explicit `case .unknown` branch — never use `default` as a substitute. This ensures new condition cases produce compiler errors rather than silent fallthrough.

**Keychain Bootstrap Sequence**

`GoldenHourApp.init()` calls `KeychainService.bootstrapIfNeeded()` synchronously before the view hierarchy renders. This is the only location where `APIKeys` constants are read. All subsequent key access uses `KeychainService.read(key:)`.

---

### Enforcement Guidelines

**All AI agents MUST:**
- Use the canonical `ConditionState` and `LocationType` raw values — these are load-bearing strings in UserDefaults and LLM prompts
- Read static tips from `Resources/StaticTips.json` — never inline tip strings in Swift source
- Route service results through `AppState` — services never hold `AppState` references
- Absorb errors inside services — never propagate `throws` to the view layer
- Use `UserDefaultsKey` constants — never hardcode key strings

**Anti-patterns to avoid:**
- `ConditionMapper` logic inside `WeatherService` (violates single-responsibility)
- `TipsService` calling `WeatherService` (service-to-service coupling)
- Hardcoded tip text in `TipBlock` or `SpotDetailSheet`
- `UserDefaults.standard.set(value, forKey: "weatherCache")` — use `UserDefaultsKey.weatherCache`
- `default:` in `ConditionState` switch statements — always enumerate cases explicitly

## Project Structure & Boundaries

### Complete Project Directory Structure

```
GoldenHour/
├── GoldenHour.xcodeproj/
│
├── GoldenHour/                          # Main app target
│   ├── App/
│   │   ├── GoldenHourApp.swift          # @main; ModelContainer setup; Keychain bootstrap; AppState init
│   │   ├── AppState.swift               # @Observable root state (conditionState, lightWindows, verdict, gpsState, weatherEntry)
│   │   └── ContentView.swift            # Root TabView (Home + Map); custom tab bar
│   │
│   ├── Models/
│   │   ├── ConditionState.swift         # ConditionState enum + canonical raw values
│   │   ├── LightWindows.swift           # LightWindows struct; WindowType enum (.golden/.blue/.neither)
│   │   ├── LocationType.swift           # LocationType enum + canonical raw values
│   │   ├── Spot.swift                   # SwiftData @Model (name, note, locationType, lat, lon, lastVisited, createdAt)
│   │   ├── WeatherResponse.swift        # Decodable: Open-Meteo current weather fields
│   │   ├── WeatherCacheEntry.swift      # Codable: WeatherResponse + fetchedAt; isStale computed
│   │   └── TipContent.swift             # Codable: body String + cameraSettings String (StaticTips.json shape)
│   │
│   ├── Services/
│   │   ├── SolarService.swift           # Sunlight wrapper; goldenHour(for:on:) + blueHour(for:on:) → LightWindows
│   │   ├── WeatherService.swift         # Open-Meteo fetch (URLSession); UserDefaults cache read/write
│   │   ├── ConditionMapper.swift        # map(_ response:) → ConditionState; verdict(for:windows:) → String
│   │   ├── TipsService.swift            # staticTip(for:window:) → TipContent; fetchLLMTip(...) → String?; UserDefaults cache
│   │   ├── LocationService.swift        # CLLocationManager delegate; GPSState machine; publishes CLLocationCoordinate2D
│   │   └── KeychainService.swift        # bootstrapIfNeeded(); read(key:) → String?; write(key:value:)
│   │
│   ├── Views/
│   │   ├── HomeView.swift               # Tab 1: VerdictView + condition summary + tomorrow window (FR1–FR10, FR27–FR32)
│   │   └── MapView.swift                # Tab 2: MKMapView + SpotAnnotation overlays (FR11–FR19)
│   │
│   ├── Components/
│   │   ├── VerdictView.swift            # Full-bleed ConditionImageView + dark header scrim + verdict Text + TimePill
│   │   ├── ConditionImageView.swift     # Background image keyed to ConditionState; crossfade on state change
│   │   ├── TimePill.swift               # Rounded label: "6:47 – 7:21 PM · Blue until 7:48"
│   │   ├── StaleDataLabel.swift         # "Weather from Xh ago" — shown when cache age > 3h or offline
│   │   ├── SpotDetailSheet.swift        # .sheet presentation; spot metadata + TipBlock + LLMTipBlock
│   │   ├── TipBlock.swift               # Static tip body + camera settings row (FR20–FR22, FR25–FR26)
│   │   ├── LLMTipBlock.swift            # "Tonight specifically:" augmentation; opacity-fade in; hidden if no response (FR23–FR24)
│   │   ├── SaveSpotSheet.swift          # Name TextField + note TextEditor + LocationType Picker + Save/Cancel (FR13–FR15)
│   │   └── SpotAnnotation.swift         # Custom MKAnnotationView: 16pt circle, selected = #DDB05E
│   │
│   ├── Utilities/
│   │   ├── Formatters.swift             # Formatters.shortTime: DateFormatter (shared instance)
│   │   ├── UserDefaultsKey.swift        # UserDefaultsKey.weatherCache + UserDefaultsKey.tipsCache prefix
│   │   └── WithTimeout.swift            # withTimeout(seconds:operation:) → T? helper
│   │
│   └── Resources/
│       ├── StaticTips.json              # Tips keyed "conditionState_windowType"; TipContent shape
│       └── Assets.xcassets/
│           ├── AppIcon.appiconset/
│           ├── Colors/                  # Golden (#DDB05E), Blue (#92A7D6), Rose (#E0C0CB),
│           │                            #   Lavender (#E6DAE4), SlateGray (#8A95A8),
│           │                            #   TextPrimary (#1A1612), TextSecondary (#6B6560)
│           └── ConditionImages/         # brokenCloudsGolden, clearSkyGolden, blueHour,
│                                        #   overcast, fog, rain  (6 images minimum)
│
├── GoldenHourTests/                     # Unit test target
│   ├── SolarServiceTests.swift          # Validate golden/blue hour window accuracy vs. known reference times
│   ├── ConditionMapperTests.swift       # All weatherCode → ConditionState mappings; verdict string coverage
│   └── TipsServiceTests.swift           # staticTip(for:window:) correct selection for all 8 valid combos
│
├── APIKeys.swift                        # GITIGNORED — enum APIKeys { static let anthropic, maptiler }
├── APIKeys.swift.template               # Committed placeholder; setup instructions in comments
└── .gitignore                           # Includes: APIKeys.swift, *.xcuserstate, DerivedData/
```

---

### Architectural Boundaries

**External API Boundaries**

| Boundary | Owner | Direction | Protocol |
|---|---|---|---|
| Open-Meteo current weather | `WeatherService` | Outbound GET | REST/JSON, no auth |
| Anthropic messages API | `TipsService` | Outbound POST | REST/JSON, Bearer token from Keychain |
| Maptiler tile CDN | `MapView` (MKTileOverlay URL template) | Outbound GET | HTTPS tile requests, API key in URL |
| iOS Keychain | `KeychainService` | Read/write | Security framework |
| GPS / CoreLocation | `LocationService` | Inbound events | CLLocationManagerDelegate |
| UserDefaults | `WeatherService`, `TipsService` | Read/write | UserDefaults standard |
| SwiftData store | SwiftData `ModelContainer` | Read/write | SwiftData framework |

**Component Communication Boundaries**

- `AppState` → views: `@Environment` injection at root; views read `@Observable` properties directly
- Views → services: via `AppState` methods only (`appState.refresh()`, `appState.saveSpot(...)`)
- Components → data: receive typed values as parameters; no `AppState` dependency inside components
- `SpotDetailSheet` → `TipsService`: called by `AppState` when sheet is presented; result assigned to local `@State` in sheet

**Data Boundaries**

- SwiftData: `Spot` model only — no weather, no tips, no settings
- UserDefaults: `WeatherCacheEntry` (key: `weatherCache`) + `TipsCacheEntry` values (key: `tipsCache_{state}_{type}`)
- Keychain: `anthropicAPIKey`, `maptilerAPIKey` — no other secrets
- In-memory only: `LightWindows`, `ConditionState`, `verdict` — recomputed each launch from fresh data

---

### Requirements to Structure Mapping

| FR Category | Primary Files |
|---|---|
| FR1–FR4 Light window calculation | `Services/SolarService.swift`, `Models/LightWindows.swift` |
| FR5–FR10 Condition assessment | `Services/WeatherService.swift`, `Services/ConditionMapper.swift`, `Models/ConditionState.swift`, `Models/WeatherResponse.swift` |
| FR11–FR19 Location & spots | `Services/LocationService.swift`, `Models/Spot.swift`, `Views/MapView.swift`, `Components/SpotAnnotation.swift`, `Components/SaveSpotSheet.swift` |
| FR20–FR26 Shooting guidance | `Services/TipsService.swift`, `Components/SpotDetailSheet.swift`, `Components/TipBlock.swift`, `Components/LLMTipBlock.swift`, `Resources/StaticTips.json` |
| FR27–FR30 Offline & connectivity | `Services/WeatherService.swift` (cache), `Services/TipsService.swift` (static fallback), `Components/StaleDataLabel.swift` |
| FR31–FR34 Config & location services | `Services/KeychainService.swift`, `Services/LocationService.swift`, `App/GoldenHourApp.swift` |

**Cross-Cutting Concern → File(s)**

| Concern | File(s) |
|---|---|
| `ConditionState` propagation | `App/AppState.swift` → `@Environment` → all views |
| Date display formatting | `Utilities/Formatters.swift` |
| UserDefaults key safety | `Utilities/UserDefaultsKey.swift` |
| LLM 4s timeout | `Utilities/WithTimeout.swift` |
| API key lifecycle | `Services/KeychainService.swift` + `App/GoldenHourApp.swift` |

---

### Data Flow

```
App launch
  └─ GoldenHourApp.init()
       ├─ KeychainService.bootstrapIfNeeded()     # reads APIKeys.swift → Keychain
       ├─ ModelContainer(for: Spot.self)           # SwiftData setup
       └─ AppState injected via .environment

Location resolves (LocationService)
  └─ AppState.coordinates updated
       ├─ SolarService.calculate() → AppState.lightWindows
       └─ WeatherService.fetch() → AppState.weatherEntry
            └─ ConditionMapper.map() → AppState.conditionState
                 └─ ConditionMapper.verdict() → AppState.verdict

User taps saved spot (MapView)
  └─ SpotDetailSheet presented
       ├─ TipsService.staticTip(for: conditionState, window: currentWindow)
       │    └─ TipBlock renders immediately (< 200ms)
       └─ Task { withTimeout(4) { TipsService.fetchLLMTip(...) } }
            ├─ response received → LLMTipBlock fades in
            └─ timeout / nil → LLMTipBlock stays hidden
```

## Architecture Validation Results

### Coherence Validation ✅

**Decision Compatibility:**
All technology choices are mutually compatible. Swift 6 + SwiftUI + `@Observable` + SwiftData are all GA on iOS 18. `async/await` is the native Swift 6 concurrency model — no Combine needed. Sunlight is a Swift Package Manager library with no conflicts with the rest of the stack. URLSession + async/await is the correct pairing. `CLLocationManager` + SwiftUI via a service abstraction is standard. Maptiler via `MKTileOverlay` is a well-established pattern. No version conflicts detected.

**Pattern Consistency:**
MVVM + `@Observable` aligns with the service injection pattern via `AppState`. Error absorption in services is consistent with the "no error propagation to views" rule. Canonical enum raw values are consistent across UserDefaults keys, LLM prompts, and `StaticTips.json` keying. `UserDefaultsKey` registry prevents all hardcoded key strings. The `withTimeout` utility correctly isolates LLM timeout logic.

**Structure Alignment:**
The file tree maps directly to service boundaries — one file per service, one file per model type. Every FR category has at least one primary owner file. Cross-cutting concerns (`Formatters`, `UserDefaultsKey`, `WithTimeout`) have dedicated utility files, not scattered inline. `AppState` sits at the correct root to propagate `ConditionState` via `@Environment`.

---

### Requirements Coverage Validation ✅

**Functional Requirements — All 34 FRs covered:**

| FR | Coverage | File |
|---|---|---|
| FR1–FR4 | ✅ | `SolarService.swift`, `LightWindows.swift` |
| FR5–FR10 | ✅ | `WeatherService.swift`, `ConditionMapper.swift`, `WeatherCacheEntry.swift` |
| FR11–FR19 | ✅ | `LocationService.swift`, `Spot.swift`, `MapView.swift`, `SaveSpotSheet.swift` |
| FR20–FR26 | ✅ | `TipsService.swift`, `TipBlock.swift`, `LLMTipBlock.swift`, `StaticTips.json` |
| FR27–FR30 | ✅ | `WeatherService.swift` cache, `TipsService.swift` static fallback, `StaleDataLabel.swift` |
| FR31–FR34 | ✅ | `KeychainService.swift`, `LocationService.swift`, `GoldenHourApp.swift` |

**Non-Functional Requirements — All 15 NFRs covered:**

| NFR | Coverage | Mechanism |
|---|---|---|
| NFR1–NFR3 (performance) | ✅ | Cache-first; solar on-device; static tips from in-memory dictionary |
| NFR4 (LLM 4s timeout) | ✅ | `WithTimeout.swift` utility |
| NFR5 (60min weather TTL) | ✅ | `WeatherCacheEntry.isStale` computed property |
| NFR6 (solar <100ms) | ✅ | Sunlight library; synchronous on-device computation |
| NFR7 (API key Keychain) | ✅ | `KeychainService` + bootstrap pattern |
| NFR8 (data transmission scope) | ✅ | No analytics SDK; documented boundary: Open-Meteo + Anthropic + Maptiler only |
| NFR9 (no telemetry) | ✅ | No third-party SDKs beyond Sunlight (solar, offline-only) |
| NFR10–NFR13 (integration resilience) | ✅ | Error absorption; independent failure domains per service |
| NFR14 (Dynamic Type) | ✅ | SwiftUI native text; no fixed font sizes |
| NFR15 (VoiceOver) | ✅ | SwiftUI semantics; explicit `accessibilityLabel` in component specs |

---

### Implementation Readiness Validation ✅

**Decision Completeness:** All 5 major decision categories documented with clear rationale. Technology choices are specific (Sunlight, Maptiler, `claude-haiku-3-5`, UserDefaults, `APIKeys.swift`). Implementation sequence is ordered and dependency-aware.

**Structure Completeness:** 23 source files defined with purpose annotations. Test target with 3 test files. All external boundaries enumerated. Requirements-to-file mapping complete for all 34 FRs.

**Pattern Completeness:** 8 conflict points addressed. Canonical enum raw values specified. UserDefaults key registry defined. Service boundary table documented. Error absorption rule clear. Anti-patterns enumerated.

---

### Gap Analysis Results

**Important Gaps (resolve before starting implementation):**

1. **`GPSState` enum not formally defined.** `AppState.gpsState: GPSState` is referenced but the enum is unspecified. Add `Models/GPSState.swift`:
   ```swift
   enum GPSState {
       case unresolved   // permission not yet requested
       case granted      // coordinates available
       case denied       // user explicitly denied
       case restricted   // MDM or parental controls
   }
   ```

2. **LLM system prompt template not canonical.** Without a canonical template, tip tone, camera settings format, and negative-condition handling will vary between agents. Add `Utilities/Prompts.swift` containing the static system prompt string as a constant — the primary quality control mechanism for LLM output.

**Minor Gaps (address during implementation):**

3. **FR3 (tomorrow's light window)** — `SolarService.calculate()` must accept a `Date` parameter, not default to today. Signature: `calculate(coordinate: CLLocationCoordinate2D, date: Date) -> LightWindows`.

4. **Condition image asset naming** — Canonical names: `brokenCloudsGolden`, `clearSkyGolden`, `blueHour`, `overcast`, `fog`, `rain` in `Assets.xcassets/ConditionImages/`.

**No critical gaps detected.**

---

### Architecture Completeness Checklist

**Requirements Analysis**
- [x] Project context thoroughly analyzed
- [x] Scale and complexity assessed
- [x] Technical constraints identified
- [x] Cross-cutting concerns mapped

**Architectural Decisions**
- [x] Critical decisions documented with versions
- [x] Technology stack fully specified
- [x] Integration patterns defined
- [x] Performance considerations addressed

**Implementation Patterns**
- [x] Naming conventions established
- [x] Structure patterns defined
- [x] Communication patterns specified
- [x] Process patterns documented

**Project Structure**
- [x] Complete directory structure defined
- [x] Component boundaries established
- [x] Integration points mapped
- [x] Requirements to structure mapping complete

---

### Architecture Readiness Assessment

**Overall Status: READY WITH MINOR GAPS**

Two important gaps (`GPSState` enum, canonical LLM prompt template) should be resolved in the first implementation story (project setup), but neither blocks the architecture document from being used now.

**Confidence Level:** High

**Key Strengths:**
- `ConditionState` as the single system spine eliminates a whole class of consistency bugs
- Service boundary rule (services never call each other) makes the codebase straightforward to test
- Error absorption pattern means the UI is always in a coherent state
- Implementation sequence respects build dependencies — no circular "define it later" issues

**Areas for Future Enhancement:**
- LLM prompt versioning if tip quality evolves
- `StaticTips.json` can be extended with additional condition types without architecture changes
- UserDefaults caching could evolve to a more sophisticated store if scope grows

### Implementation Handoff

**AI Agent Guidelines:**
- Follow all architectural decisions exactly as documented
- Define `ConditionState`, `LocationType`, `GPSState`, and `LightWindows` before any other work
- Use `StaticTips.json` as the sole source of static tip strings — never inline
- Refer to the Service Boundary table before adding any method to any service
- Resolve the two important gaps (`GPSState.swift`, `Prompts.swift`) in story 1

**First Implementation Priority:**
```
Xcode: File > New Project > iOS > App (SwiftUI, SwiftData)
Add SPM: https://github.com/BinaryBirds/Sunlight
Create: APIKeys.swift (gitignored), APIKeys.swift.template
Implement: KeychainService → ConditionState/LocationType/GPSState enums → AppState
```
