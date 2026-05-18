# Story 1.1: Project Initialization & Security Foundation

Status: done

## Story

As a developer building the Golden Hour app,
I want the Xcode project initialized with all core types, utilities, and the Keychain security layer in place,
so that all subsequent stories have a consistent, dependency-safe foundation to build upon.

## Acceptance Criteria

1. **Given** a new Xcode project is created (File > New Project > iOS > App, SwiftUI interface, SwiftData storage, Swift language) with targets `GoldenHour` and `GoldenHourTests` **When** the Sunlight Swift Package (https://github.com/BinaryBirds/Sunlight) is added via SPM **Then** the project builds successfully on an iOS 18 simulator with no errors

2. **Given** `APIKeys.swift.template` is committed with empty string placeholders and setup instructions **When** a developer clones the repository **Then** `APIKeys.swift.template` is present; `APIKeys.swift` is absent (listed in `.gitignore`)

3. **Given** `APIKeys.swift` exists locally with real values for `APIKeys.anthropic` and `APIKeys.maptiler` **When** `GoldenHourApp.init()` executes before the view hierarchy renders **Then** `KeychainService.bootstrapIfNeeded()` is called synchronously, reads both keys, and writes them to iOS Keychain if not already present **And** on all subsequent launches, `KeychainService.read(key:)` retrieves keys from Keychain — `APIKeys.swift` is never referenced again

4. **Given** `ConditionState.swift` is implemented **When** the enum is inspected **Then** it has exactly 6 cases with canonical raw values: `brokenClouds="broken_clouds"`, `clearSky="clear_sky"`, `overcast="overcast"`, `fog="fog"`, `rain="rain"`, `unknown="unknown"`, conforming to `String, Codable` **And** `LocationType` has 6 cases with matching snake_case raw values **And** `GPSState` has 4 cases: `unresolved`, `granted`, `denied`, `restricted` **And** `LightWindows` struct has `goldenStart`, `goldenEnd`, `blueStart`, `blueEnd` (`Date`) and `currentWindow: WindowType` computed property

5. **Given** `AppState.swift` is implemented as `@Observable @MainActor final class AppState` **When** injected at root via `.environment` **Then** it exposes: `conditionState: ConditionState = .unknown`, `lightWindows: LightWindows?`, `verdict: String = ""`, `weatherEntry: WeatherCacheEntry?`, `gpsState: GPSState = .unresolved`

6. **Given** Utilities group is created **When** `UserDefaultsKey.swift`, `Formatters.swift`, `WithTimeout.swift`, and `Prompts.swift` are implemented **Then** `UserDefaultsKey.weatherCache = "weatherCache"` and `UserDefaultsKey.tipsCache = "tipsCache_"` are the only UserDefaults key strings in the codebase **And** `Formatters.shortTime` is a single shared `DateFormatter` instance (`timeStyle: .short`, `dateStyle: .none`) **And** `withTimeout(seconds:operation:) -> T?` returns nil if the async operation exceeds the given duration **And** `Prompts.systemPrompt` is a non-empty `static let String` defining the photography assistant persona with output format constraints, camera settings requirement, honest negative-condition handling, and condition-reference requirement

## Tasks / Subtasks

- [x] Task 1: Create Xcode project and add SPM dependency (AC: 1)
  - [x] 1.1 File > New > Project > iOS > App: Product Name "GoldenHour", Interface "SwiftUI", Storage "SwiftData", Language "Swift"; target iOS 18.0 minimum deployment
  - [x] 1.2 Add GoldenHourTests target if not auto-created (Unit Testing Bundle, host: GoldenHour)
  - [x] 1.3 Add Sunlight package: File > Add Package Dependencies, URL https://github.com/BinaryBirds/Sunlight, version "Up to Next Major" from 1.0.0; add Sunlight library to GoldenHour target
  - [x] 1.4 Verify clean build on iOS 18 simulator — fix any default template warnings before proceeding

- [x] Task 2: Set up APIKeys security layer (AC: 2, 3)
  - [x] 2.1 Create `GoldenHour/APIKeys.swift.template` with content as specified in Dev Notes; commit this file
  - [x] 2.2 Copy template to `GoldenHour/APIKeys.swift`; add `APIKeys.swift` to `.gitignore` (root-level)
  - [x] 2.3 Implement `GoldenHour/Services/KeychainService.swift` — `Sendable` struct, `write(key:value:)`, `read(key:) -> String?`, `delete(key:)`, `bootstrapIfNeeded()` — see Dev Notes for exact implementation pattern
  - [x] 2.4 Modify `GoldenHourApp.swift` entry point: call `KeychainService.shared.bootstrapIfNeeded()` in `init()`, before `WindowGroup`

- [x] Task 3: Define all core model types (AC: 4)
  - [x] 3.1 Create `GoldenHour/Models/ConditionState.swift` — enum `ConditionState: String, Codable, CaseIterable` with exactly 6 cases and canonical raw values
  - [x] 3.2 Create `GoldenHour/Models/LocationType.swift` — enum `LocationType: String, Codable, CaseIterable` with 6 cases
  - [x] 3.3 Create `GoldenHour/Models/GPSState.swift` — enum `GPSState` with 4 cases (no raw value needed)
  - [x] 3.4 Create `GoldenHour/Models/LightWindows.swift` — struct `LightWindows` with 4 `Date` fields and `currentWindow: WindowType` computed property; also define `enum WindowType: String` with cases `golden`, `blue`, `neither`
  - [x] 3.5 Create `GoldenHour/Models/WeatherCacheEntry.swift` — stub `Codable` struct with `fetchedAt: Date` and `isStale: Bool` computed property (full implementation in Story 1.3; stub needed here for AppState compile)

- [x] Task 4: Implement AppState (AC: 5)
  - [x] 4.1 Create `GoldenHour/AppState.swift` — `@Observable @MainActor final class AppState` with all 5 required properties at their initial values; see Dev Notes for exact declaration and injection pattern
  - [x] 4.2 Update `GoldenHourApp.swift` to hold `@State private var appState = AppState()` and inject via `.environment(appState)` (NOT `.environmentObject`)

- [x] Task 5: Implement all utility files (AC: 6)
  - [x] 5.1 Create `GoldenHour/Utilities/UserDefaultsKey.swift` — `enum UserDefaultsKey` with static string constants; `weatherCache = "weatherCache"`, `tipsCache = "tipsCache_"`
  - [x] 5.2 Create `GoldenHour/Utilities/Formatters.swift` — `enum Formatters` with `static let shortTime: DateFormatter` (timeStyle: .short, dateStyle: .none, locale: .current)
  - [x] 5.3 Create `GoldenHour/Utilities/WithTimeout.swift` — generic `func withTimeout<T: Sendable>(seconds: Double, operation: @Sendable () async -> T) async -> T?`
  - [x] 5.4 Create `GoldenHour/Utilities/Prompts.swift` — `enum Prompts` with `static let systemPrompt: String` — see Dev Notes for required content structure
  - [x] 5.5 Build and confirm all targets compile cleanly (no warnings, no errors)

- [x] Task 6: Verify all acceptance criteria (AC: 1–6)
  - [x] 6.1 Confirm `Sunlight` import compiles in a scratch file or empty test
  - [x] 6.2 Confirm `.gitignore` contains `APIKeys.swift`; confirm `APIKeys.swift.template` is tracked in git
  - [x] 6.3 Confirm `KeychainService.bootstrapIfNeeded()` is called before `WindowGroup` in `GoldenHourApp.init()`
  - [x] 6.4 Confirm all enum raw values match canonical strings exactly (case-sensitive)
  - [x] 6.5 Confirm `AppState` uses `@Observable` macro (not `ObservableObject`) and `.environment()` injection (not `.environmentObject()`)
  - [x] 6.6 Confirm `UserDefaultsKey` is the only place key strings are defined — grep codebase for `UserDefaults.standard.set` / `.object(forKey:)` to ensure no hardcoded strings elsewhere

### Review Findings

- [x] [Review][Patch] `withTimeout` force-unwrap crash — `group.next()!` crashes if parent task is cancelled before either child task completes; replace with `guard let result = await group.next() else { return nil }` [`GoldenHour/Utilities/WithTimeout.swift:12`]
- [x] [Review][Patch] `bootstrapIfNeeded` silently persists empty-string API keys — if `APIKeys.anthropic` or `APIKeys.maptiler` are empty (unfilled template), the nil-guard passes and `""` is written to Keychain; subsequent launches read `""` (not nil) so the key is never corrected without a reinstall; add `guard !value.isEmpty` before writing [`GoldenHour/Services/KeychainService.swift:52-58`]
- [x] [Review][Defer] `withTimeout` race at exact boundary — timeout task can win the `group.next()` race at the precise instant the operation also completes, returning nil spuriously; negligible probability on a 4s LLM timeout [`GoldenHour/Utilities/WithTimeout.swift`] — deferred, pre-existing
- [x] [Review][Defer] API keys compiled into binary — static string literals in `APIKeys.swift` are recoverable via binary inspection; intentional spec-mandated tradeoff for v1 personal tool — deferred, pre-existing
- [x] [Review][Defer] Key rotation requires reinstall — `bootstrapIfNeeded` only writes once; no update path without Keychain wipe — deferred, pre-existing
- [x] [Review][Defer] No `kSecAttrAccessible` on Keychain items — default allows iCloud/device backup inclusion; benign for solo personal tool — deferred, pre-existing
- [x] [Review][Defer] No Keychain access group — future extensions could read API keys; no extensions in v1 scope — deferred, pre-existing
- [x] [Review][Defer] `delete`-then-`add` pattern is non-atomic — standard iOS pattern; negligible failure risk in practice — deferred, pre-existing
- [x] [Review][Defer] `Formatters.shortTime` locale captured at init; stale after mid-session locale change — locale changes mid-session are rare for personal tool — deferred, pre-existing
- [x] [Review][Defer] `GoldenHourApp` uses `Item.self` placeholder in ModelContainer — by design; Story 2.1 replaces with `Spot.self` — deferred, pre-existing
- [x] [Review][Defer] `fatalError` on ModelContainer creation — reinstall is acceptable recovery path for personal tool — deferred, pre-existing
- [x] [Review][Defer] `handleLocationUpdate` spawns unstructured Task; concurrent refreshes may race — low practical risk; 500m filter reduces frequency — deferred, pre-existing
- [x] [Review][Defer] `LightWindows` no start < end validation — Sunlight library is reliable; polar edge cases out of v1 scope — deferred, pre-existing
- [x] [Review][Defer] `isStale` returns false on negative interval from clock skew — false-fresh safer than false-stale for personal tool — deferred, pre-existing
- [x] [Review][Defer] `refresh()` no-ops silently when no coordinate — correct guard; UI feedback for manual refresh is a later story concern — deferred, pre-existing

## Dev Notes

### Project Structure

Create the following directory hierarchy inside `GoldenHour/`:

```
GoldenHour/
├── GoldenHourApp.swift          (entry point — UPDATE)
├── ContentView.swift            (default; leave as placeholder)
├── APIKeys.swift                (gitignored; populate locally)
├── APIKeys.swift.template       (committed; empty placeholders)
├── AppState.swift               (NEW)
├── Models/
│   ├── ConditionState.swift     (NEW)
│   ├── LocationType.swift       (NEW)
│   ├── GPSState.swift           (NEW)
│   ├── LightWindows.swift       (NEW)
│   └── WeatherCacheEntry.swift  (NEW — stub only)
├── Services/
│   └── KeychainService.swift    (NEW)
└── Utilities/
    ├── UserDefaultsKey.swift    (NEW)
    ├── Formatters.swift         (NEW)
    ├── WithTimeout.swift        (NEW)
    └── Prompts.swift            (NEW)
```

Stories 1.2–1.4 will add to `Services/` and `Views/`. Do not create those directories now.

### Exact Enum Definitions

These raw values are load-bearing strings — they are used as dictionary keys in `StaticTips.json` (Story 2.3) and as UserDefaults cache keys. Get them right here.

```swift
// ConditionState.swift
enum ConditionState: String, Codable, CaseIterable {
    case brokenClouds = "broken_clouds"
    case clearSky     = "clear_sky"
    case overcast     = "overcast"
    case fog          = "fog"
    case rain         = "rain"
    case unknown      = "unknown"
}

// LocationType.swift
enum LocationType: String, Codable, CaseIterable {
    case coastal   = "coastal"
    case urban     = "urban"
    case forest    = "forest"
    case openField = "open_field"
    case elevated  = "elevated"
    case other     = "other"
}

// GPSState.swift
enum GPSState {
    case unresolved
    case granted
    case denied
    case restricted
}

// LightWindows.swift
enum WindowType: String {
    case golden  = "golden"
    case blue    = "blue"
    case neither = "neither"
}

struct LightWindows {
    let goldenStart: Date
    let goldenEnd:   Date
    let blueStart:   Date
    let blueEnd:     Date

    var currentWindow: WindowType {
        let now = Date.now
        if now >= goldenStart && now <= goldenEnd { return .golden }
        if now >= blueStart   && now <= blueEnd   { return .blue   }
        return .neither
    }
}
```

### WeatherCacheEntry Stub

Full implementation is in Story 1.3 (WeatherResponse + Codable storage). For now, create a minimal stub so `AppState` compiles:

```swift
// Models/WeatherCacheEntry.swift
struct WeatherCacheEntry: Codable {
    let fetchedAt: Date

    var isStale: Bool {
        Date.now.timeIntervalSince(fetchedAt) > 3600
    }
}
```

Story 1.3 will add `response: WeatherResponse` to this struct. That is a non-breaking additive change.

### AppState Declaration

```swift
// AppState.swift
import Observation
import Foundation

@Observable
@MainActor
final class AppState {
    var conditionState: ConditionState  = .unknown
    var lightWindows:   LightWindows?   = nil
    var verdict:        String          = ""
    var weatherEntry:   WeatherCacheEntry? = nil
    var gpsState:       GPSState        = .unresolved
}
```

**Injection (GoldenHourApp.swift):**
```swift
@main
struct GoldenHourApp: App {
    @State private var appState = AppState()

    init() {
        KeychainService.shared.bootstrapIfNeeded()
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(appState)   // ← NOT .environmentObject()
        }
    }
}
```

**Consuming in views:**
```swift
struct SomeView: View {
    @Environment(AppState.self) private var appState
    // ...
}
```

**Do NOT use `@EnvironmentObject` or `@ObservedObject` — these are the old pattern and will not wire up to `@Observable`.**

### KeychainService Implementation

`KeychainService` is a plain `Sendable` struct (no `@MainActor`). The Security framework calls are synchronous, fast, and safe to call from the main thread for bootstrap purposes.

```swift
// Services/KeychainService.swift
import Foundation
import Security

struct KeychainService: Sendable {
    static let shared = KeychainService()
    private let service: String

    private init(service: String = Bundle.main.bundleIdentifier ?? "com.golden-hour") {
        self.service = service
    }

    @discardableResult
    func write(key: String, value: String) -> Bool {
        guard let data = value.data(using: .utf8) else { return false }
        delete(key: key)                          // always delete first — avoids errSecDuplicateItem
        let query: [CFString: Any] = [
            kSecClass:       kSecClassGenericPassword,
            kSecAttrService: service,
            kSecAttrAccount: key,
            kSecValueData:   data
        ]
        return SecItemAdd(query as CFDictionary, nil) == errSecSuccess
    }

    func read(key: String) -> String? {
        let query: [CFString: Any] = [
            kSecClass:       kSecClassGenericPassword,
            kSecAttrService: service,
            kSecAttrAccount: key,
            kSecReturnData:  true,
            kSecMatchLimit:  kSecMatchLimitOne      // required — omitting causes errSecParam
        ]
        var result: CFTypeRef?
        guard SecItemCopyMatching(query as CFDictionary, &result) == errSecSuccess,
              let data = result as? Data,
              let string = String(data: data, encoding: .utf8) else { return nil }
        return string
    }

    @discardableResult
    func delete(key: String) -> Bool {
        let query: [CFString: Any] = [
            kSecClass:       kSecClassGenericPassword,
            kSecAttrService: service,
            kSecAttrAccount: key
        ]
        return SecItemDelete(query as CFDictionary) == errSecSuccess
    }

    // Called once at app launch before view hierarchy renders.
    // Reads from APIKeys.swift (compile-time constants) and writes to Keychain if not already present.
    func bootstrapIfNeeded() {
        if read(key: "anthropic_api_key") == nil {
            write(key: "anthropic_api_key", value: APIKeys.anthropic)
        }
        if read(key: "maptiler_api_key") == nil {
            write(key: "maptiler_api_key", value: APIKeys.maptiler)
        }
    }
}
```

**Keychain key strings:** `"anthropic_api_key"` and `"maptiler_api_key"`. These are the canonical strings used by all stories that read API keys. Do not invent new key strings.

### APIKeys Files

**`APIKeys.swift.template`** (commit this):
```swift
// APIKeys.swift.template
// DO NOT COMMIT APIKeys.swift — only commit this template.
// Setup: Copy this file to APIKeys.swift and fill in your actual keys.
// APIKeys.swift is gitignored.

struct APIKeys {
    static let anthropic = ""   // Your Anthropic API key
    static let maptiler  = ""   // Your Maptiler API key
}
```

**`APIKeys.swift`** (gitignored — create locally):
```swift
struct APIKeys {
    static let anthropic = "sk-ant-..."   // real key
    static let maptiler  = "..."          // real key
}
```

### Sunlight Library API (Story 1.2 preview — read now)

Story 1.2 will use Sunlight. Know these facts now so your type stubs are compatible:

- `import Sunlight` (product name in SPM)
- `Sunlight(coordinate: SunlightCoordinate, date: Date, timezone: TimeZone = .current)` — value type (struct), thread-safe
- `SunlightCoordinate(latitude: Double, longitude: Double)` — NOT `CLLocationCoordinate2D`; no CoreLocation import needed
- Computed properties are optional: `goldenHourEvening: SunlightPeriod?`, `blueHourEvening: SunlightPeriod?`
- `SunlightPeriod` has `.start: Date` and `.end: Date`
- Each property access recomputes — read into local `let` variables, don't access the same property twice
- `LightWindows` struct (defined above) maps directly from `SunlightPeriod` values

`SolarService` in Story 1.2 will call `Sunlight(coordinate:date:)` and map the result to `LightWindows`. The `LightWindows` type you define here must have `Date` fields, not `SunlightPeriod` — `SolarService` will unwrap the optionals.

### Utilities

**`UserDefaultsKey.swift`** — use a caseless enum (namespace pattern) to prevent accidental instantiation:
```swift
enum UserDefaultsKey {
    static let weatherCache = "weatherCache"
    static let tipsCache    = "tipsCache_"    // underscore suffix; Story 2.4 appends conditionState+locationType
}
```

**`Formatters.swift`** — same namespace pattern; single shared instance prevents repeated allocations:
```swift
enum Formatters {
    static let shortTime: DateFormatter = {
        let f = DateFormatter()
        f.timeStyle = .short
        f.dateStyle = .none
        f.locale    = .current
        return f
    }()
}
```

**`WithTimeout.swift`** — used by Story 2.4's LLM fetch to enforce the 4-second timeout:
```swift
func withTimeout<T: Sendable>(
    seconds: Double,
    operation: @escaping @Sendable () async -> T
) async -> T? {
    await withTaskGroup(of: T?.self) { group in
        group.addTask { await operation() }
        group.addTask {
            try? await Task.sleep(for: .seconds(seconds))
            return nil
        }
        let result = await group.next()!
        group.cancelAll()
        return result
    }
}
```

**`Prompts.swift`** — the canonical LLM prompt. This is a quality control mechanism; the exact wording matters for tip tone. Use this structure:

```swift
enum Prompts {
    static let systemPrompt = """
    You are a photographer's assistant helping with golden hour and blue hour shooting decisions. \
    Given the current light window, weather condition, location type, and any spot notes, provide \
    specific shooting guidance.

    Rules:
    - Respond in 2-4 sentences maximum.
    - Always include one specific camera setting recommendation (ISO, aperture, shutter speed).
    - Do not hedge or use phrases like "might" or "could be good". Be direct and confident.
    - For poor conditions (overcast, rain), reframe honestly — suggest what IS possible, not what isn't.
    - Reference the specific condition and location type in your response.
    - Write as if texting a friend who is already at the location.
    """
}
```

### Swift 6 / @Observable Anti-Patterns to Avoid

The dev agent on Story 1.4+ will consume AppState. Document these gotchas now:

- ❌ `.environmentObject(appState)` → ✅ `.environment(appState)`
- ❌ `@EnvironmentObject var appState: AppState` → ✅ `@Environment(AppState.self) var appState`
- ❌ `@Published var x` inside `@Observable` class → ✅ plain `var x` (macro handles it)
- ❌ `@ObservedObject` / `@StateObject` → ✅ `@State` for ownership, plain `let` for reference
- For bindings: `@Bindable var s = appState` then `$s.property` — only needed when a child view needs a `Binding<T>`

### Project Structure Notes

- iOS 18.0 minimum deployment target
- Swift 6 language mode (Xcode default for new projects targeting iOS 18)
- SwiftData storage selected at project creation — this creates the default `ModelContainer` setup; Story 2.1 will add the `Spot` model to it. Do NOT remove SwiftData from the project even though it's unused in this story.
- The default `ContentView.swift` created by Xcode is fine as a placeholder — leave it with its "Hello, world!" text for now. Story 1.4 replaces it.
- No test files need to be created in this story — unit tests start in Story 1.2 (`SolarServiceTests`).

### References

- [Source: epics.md, Story 1.1 Acceptance Criteria]
- [Source: architecture.md, Technical Stack and Implementation Sequence sections]
- [Source: Sunlight API research — `SunlightCoordinate`, `SunlightPeriod`, optional properties]
- [Source: Swift 6 Keychain pattern — `SecItemAdd`/`SecItemCopyMatching` via Security framework]
- [Source: @Observable gotchas — `.environment()` vs `.environmentObject()`]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

- Sunlight SPM package showed red/unresolved in Xcode after initial add; fixed with File > Packages > Reset Package Caches — not a code issue, cache failure during initial fetch.
- CLI xcodebuild unusable due to CommandLineTools/Xcode SDK conflict (MacOSX.sdk interfering with iPhoneSimulator SDK during clang module scanning). Build verified via Xcode GUI (Cmd+B) instead.

### Completion Notes List

- All 14 new Swift source files created matching Dev Notes specifications exactly.
- `GoldenHourApp.swift` updated: `@State private var appState`, `init()` with `bootstrapIfNeeded()`, `.environment(appState)` injection, SwiftData `ModelContainer` retained.
- `.gitignore` created at git root covering `APIKeys.swift`, Xcode artifacts, DerivedData, SPM `.build/`.
- `ConditionState` raw values are load-bearing strings — match exactly what will be used as keys in `StaticTips.json` (Story 2.3) and UserDefaults cache keys (Story 2.4).
- `WeatherCacheEntry` is a stub — Story 1.3 adds `response: WeatherResponse` as a non-breaking additive change.
- No unit tests added — per Dev Notes, tests begin in Story 1.2 (`SolarServiceTests`).
- Build confirmed clean in Xcode on iOS 26.4 simulator.

### File List

- `.gitignore` (new)
- `GoldenHour/GoldenHourApp.swift` (modified)
- `GoldenHour/APIKeys.swift.template` (new — committed)
- `GoldenHour/APIKeys.swift` (new — gitignored, local only)
- `GoldenHour/AppState.swift` (new)
- `GoldenHour/Models/ConditionState.swift` (new)
- `GoldenHour/Models/LocationType.swift` (new)
- `GoldenHour/Models/GPSState.swift` (new)
- `GoldenHour/Models/LightWindows.swift` (new)
- `GoldenHour/Models/WeatherCacheEntry.swift` (new — stub)
- `GoldenHour/Services/KeychainService.swift` (new)
- `GoldenHour/Utilities/UserDefaultsKey.swift` (new)
- `GoldenHour/Utilities/Formatters.swift` (new)
- `GoldenHour/Utilities/WithTimeout.swift` (new)
- `GoldenHour/Utilities/Prompts.swift` (new)

### Change Log

- 2026-05-10: Story 1.1 implemented — Xcode project initialized with Sunlight SPM dependency, APIKeys security layer with Keychain bootstrap, all core model types (ConditionState, LocationType, GPSState, LightWindows, WeatherCacheEntry stub), AppState @Observable root state, utility namespaces (UserDefaultsKey, Formatters, WithTimeout, Prompts), and .gitignore. Build verified clean in Xcode.
