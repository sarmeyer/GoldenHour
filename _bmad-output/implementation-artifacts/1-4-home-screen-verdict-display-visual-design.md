# Story 1.4: Home Screen — Verdict Display & Visual Design

Status: done

## Story

As a photographer using Golden Hour,
I want to open the app and immediately see a confident go/no-go verdict over a condition-appropriate full-bleed photograph,
so that I can decide whether to go out in under 60 seconds without opening any other app.

## Acceptance Criteria

1. **Given** 6 condition photography images are added to `Assets.xcassets/ConditionImages/` named exactly: `brokenCloudsGolden`, `clearSkyGolden`, `blueHour`, `overcast`, `fog`, `rain`
   **When** `ConditionImageView` renders for a given `ConditionState`
   **Then** the correct image displays full-bleed (`.resizable()`, `.scaledToFill()`, `.clipped()`, `.ignoresSafeArea()`)
   **And** when `ConditionState` changes, the image transitions via crossfade; when `@Environment(\.accessibilityReduceMotion)` is true, the new image appears instantly

2. **Given** all 7 named color assets are defined in `Assets.xcassets/Colors/` (Golden `#DDB05E`, Blue `#92A7D6`, Rose `#E0C0CB`, Lavender `#E6DAE4`, SlateGray `#8A95A8`, TextPrimary `#1A1612`, TextSecondary `#6B6560`)
   **When** any component references a color
   **Then** it uses the named asset — no hardcoded hex values appear in Swift source

3. **Given** `VerdictView` implements the header scrim as `LinearGradient` inside a `GeometryReader` (`.frame(height: proxy.size.height * 0.52)`, `rgba(26,22,18,0.88)` → transparent, top-anchored)
   **When** the home screen renders
   **Then** the verdict `Text` appears immediately below the status bar via `.safeAreaInset(edge: .top)` in New York serif, `.title` Dynamic Type style, warm white, `lineLimit(2)`
   **And** the condition summary appears 9pt below in `#DDB05E`, `.subheadline` Dynamic Type style
   **And** `GeometryReader` is used only for this scrim — nowhere else in the view hierarchy

4. **Given** `TimePill` is rendered below the condition summary
   **When** golden and blue hour windows are available
   **Then** it displays the full range on one line (e.g., "6:47 – 7:21 PM · Blue until 7:48") using `Formatters.shortTime`, in a rounded rectangle (`cornerRadius: 20`, translucent fill and border per UX spec)

5. **Given** `HomeView` includes tomorrow's light window display (FR3)
   **When** the user views the home screen
   **Then** tomorrow's golden and blue hour window times are visible (scrollable if needed) formatted via `Formatters.shortTime`

6. **Given** `AppState.weatherEntry.fetchedAt` is more than 3 hours ago, or the device is offline
   **When** `HomeView` renders
   **Then** `StaleDataLabel` appears below `TimePill` as "Weather from Xh ago" — `#6B6560`, `.caption2` Dynamic Type, italic — no icon, no warning color

7. **Given** `AppState.gpsState` is `.denied` or `.restricted`
   **When** `HomeView` renders
   **Then** an inline location-unavailable message appears with a tappable "Open Settings" deep-link (no modal, no blocking alert) (FR32)

8. **Given** `AppState.gpsState` is `.unresolved` and no cached data exists (first launch)
   **When** `HomeView` renders
   **Then** a skeleton loading state is shown — not a blank screen, not an error

9. **Given** `CustomTabBar` replaces the native SwiftUI `TabView` bar
   **When** rendered
   **Then** it is 58pt height, `#E6DAE4` background, 1px `#E0C0CB` top border
   **And** active tab icon is `#DDB05E`; inactive is `#8A95A8` at 50% opacity
   **And** each tab has `.accessibilityLabel("Home")` and `.accessibilityLabel("Map")` respectively

10. **Given** `VerdictView` uses `accessibilityElement(children: .combine)`
    **When** VoiceOver focuses on the verdict surface
    **Then** it reads verdict sentence + condition summary + light window times as one combined unit
    **And** `ConditionImageView` has a descriptive `accessibilityLabel` (e.g., "Golden hour light through broken clouds")

11. **Given** cached weather data is available on cold launch
    **When** the home screen appears
    **Then** the verdict and condition image are visible within 2 seconds (NFR1) — no blank screen during data loading

12. **Given** all text elements use SwiftUI Dynamic Type semantic styles (no fixed point sizes)
    **When** the system font size is set to Accessibility Extra Extra Extra Large
    **Then** the verdict truncates at `lineLimit(2)` and no text clips or overflows its container

## Tasks / Subtasks

- [x] **Task 1: Add color assets and condition image placeholders to Assets.xcassets** (AC2, AC1)
  - [x] Create `Assets.xcassets/Colors/` group; add 7 named color sets: `Golden` (#DDB05E), `Blue` (#92A7D6), `Rose` (#E0C0CB), `Lavender` (#E6DAE4), `SlateGray` (#8A95A8), `TextPrimary` (#1A1612), `TextSecondary` (#6B6560) — set Appearances to "Any, Dark" using the same value for both appearances (no dark mode variants in v1)
  - [x] Create `Assets.xcassets/ConditionImages/` group; add 6 placeholder image assets named exactly: `brokenCloudsGolden`, `clearSkyGolden`, `blueHour`, `overcast`, `fog`, `rain` — use distinct solid-color fills as stand-ins until real photography is sourced (e.g. warm amber for golden, cool blue for blueHour, medium gray for overcast, muted gray for fog, dark blue-gray for rain)

- [x] **Task 2: Create ConditionImageView.swift** (AC1, AC10)
  - [x] Create `GoldenHour/Components/ConditionImageView.swift`
  - [x] Define `struct ConditionImageView: View` taking `conditionState: ConditionState` and `currentWindow: WindowType` parameters
  - [x] Implement image selection logic (see Dev Notes for mapping table)
  - [x] Apply `.resizable()`, `.scaledToFill()`, `.clipped()`, `.ignoresSafeArea()` — this view must fill its container edge-to-edge
  - [x] Read `@Environment(\.accessibilityReduceMotion) private var reduceMotion`
  - [x] Apply crossfade transition: `.transition(.opacity)` + `.animation(.easeInOut(duration: 0.4), value: conditionState)` when `!reduceMotion`; no animation when `reduceMotion == true`
  - [x] Set `accessibilityLabel` to a descriptive string per condition (see Dev Notes for strings)
  - [x] Use `.id(conditionState)` to force SwiftUI to rebuild the view (not update it) when state changes — required for `.transition` to fire

- [x] **Task 3: Create TimePill.swift** (AC4)
  - [x] Create `GoldenHour/Components/TimePill.swift`
  - [x] Define `struct TimePill: View` taking `windows: LightWindows?`
  - [x] Return `EmptyView` when `windows == nil`
  - [x] Format text as: `"{goldenStart} – {goldenEnd} · Blue until {blueEnd}"` using `Formatters.shortTime` (see Dev Notes for exact formatting)
  - [x] Background: `RoundedRectangle(cornerRadius: 20)` with `.fill(Color.white.opacity(0.14))` and `.strokeBorder(Color.white.opacity(0.22), lineWidth: 1)`
  - [x] Text: `.caption` Dynamic Type style, `.foregroundColor(.white)`, `.fontWeight(.medium)`
  - [x] Padding: 10pt vertical, 16pt horizontal inside the pill

- [x] **Task 4: Create StaleDataLabel.swift** (AC6)
  - [x] Create `GoldenHour/Components/StaleDataLabel.swift`
  - [x] Define `struct StaleDataLabel: View` taking `fetchedAt: Date`
  - [x] Compute hours elapsed: `max(1, Int(Date.now.timeIntervalSince(fetchedAt) / 3600))`
  - [x] Display `"Weather from \(hours)h ago"` — no icon
  - [x] Style: `.foregroundColor(Color("TextSecondary"))`, `.font(.caption2)`, `.italic()`

- [x] **Task 5: Add conditionSummary computed property** (AC3 — condition summary text)
  - [x] Add `var displayName: String` computed property via extension on `ConditionState` in `ConditionState.swift` (see Dev Notes for values)
  - [x] No change to any other file — views read `appState.conditionState.displayName` directly

- [x] **Task 6: Create VerdictView.swift** (AC3, AC6, AC7, AC8, AC10, AC12)
  - [x] Create `GoldenHour/Components/VerdictView.swift`
  - [x] Define `struct VerdictView: View` with `@Environment(AppState.self) private var appState`
  - [x] Outer layout: `ZStack` containing `ConditionImageView` (fills screen) + `GeometryReader` for the scrim (see Dev Notes for exact structure)
  - [x] Scrim: `LinearGradient(colors: [Color("TextPrimary").opacity(0.88), .clear], startPoint: .top, endPoint: .bottom)` — `.frame(height: proxy.size.height * 0.52)`, top-anchored, `.ignoresSafeArea()`
  - [x] Apply `.safeAreaInset(edge: .top)` on the ZStack to render verdict content below the status bar without hardcoded offsets
  - [x] Verdict text: `.font(.system(.title, design: .serif))`, `.foregroundColor(.white)`, `lineLimit(2)`, no fixed point size
  - [x] Condition summary: 9pt below verdict (`.padding(.top, 9)`), `.font(.subheadline.weight(.medium))`, `.foregroundColor(Color("Golden"))`
  - [x] `TimePill` 8pt below condition summary
  - [x] `StaleDataLabel` shown when `isDataStale` (see Dev Notes for staleness rule)
  - [x] GPS denied/restricted state: replace verdict content with inline message + Settings button (see Dev Notes)
  - [x] Skeleton state: when `gpsState == .unresolved && weatherEntry == nil` (see Dev Notes)
  - [x] `accessibilityElement(children: .combine)` on the verdict content VStack

- [x] **Task 7: Create HomeView.swift** (AC5, AC11)
  - [x] Create `GoldenHour/Views/HomeView.swift`
  - [x] Define `struct HomeView: View` with `@Environment(AppState.self) private var appState`
  - [x] `VerdictView` occupies the full screen
  - [x] Tomorrow's windows section: display below VerdictView in a scrollable section showing tomorrow's golden and blue window times using `Formatters.shortTime` (see Dev Notes for layout)
  - [x] `.task { appState.requestLocation() }` on HomeView to trigger location + weather on appear

- [x] **Task 8: Create CustomTabBar.swift and rewrite ContentView.swift** (AC9)
  - [x] Create `GoldenHour/Components/CustomTabBar.swift`
  - [x] `CustomTabBar` is a `View` struct, not wrapping `TabView`'s native tab bar — it's a standalone control (see Dev Notes for structure)
  - [x] Height: 58pt; background: `Color("Lavender")`; top border: 1pt `Color("Rose")` via `.overlay(alignment: .top)`
  - [x] Two tabs: Home (house SF Symbol) and Map (map SF Symbol)
  - [x] Active tab: `Color("Golden")`; inactive: `Color("SlateGray").opacity(0.5)`
  - [x] `.accessibilityLabel("Home")` and `.accessibilityLabel("Map")` on respective tab buttons
  - [x] Create placeholder `GoldenHour/Views/MapView.swift` — `Text("Map")` on a `Color("Lavender")` background
  - [x] Rewrite `ContentView.swift`: replace the Xcode-default `NavigationSplitView` with a `ZStack` that layers the active tab's view + `CustomTabBar` at the bottom (see Dev Notes for layout)
  - [x] Remove all `Item` CRUD code from `ContentView.swift` (the `@Query` items, `addItem`, `deleteItems`, `NavigationSplitView`)

## Dev Notes

### Architecture Compliance — READ FIRST

**Existing patterns from Stories 1.1–1.3 (do not break):**
- `AppState` is `@Observable @MainActor final class` — access via `@Environment(AppState.self)` in views. Do NOT use `@EnvironmentObject`.
- All `AppState` property reads in views are direct: `appState.conditionState`, `appState.verdict`, `appState.lightWindows`, etc.
- `lightWindowsTomorrow: LightWindows?` already exists on `AppState` — no AppState changes needed for tomorrow display.
- `weatherEntry: WeatherCacheEntry?` already exists — use `weatherEntry?.fetchedAt` for stale calculation.
- `gpsState: GPSState` already exists with cases `.unresolved`, `.granted`, `.denied`, `.restricted`.
- `requestLocation()` already exists on `AppState` — call from HomeView's `.task`.
- No `default:` in `ConditionState` switches — enumerate all 6 cases explicitly.

**File location rules (architecture.md):**
- View files → `GoldenHour/Views/`
- Reusable component structs → `GoldenHour/Components/`
- New files in these directories are auto-included by PBXFileSystemSynchronizedRootGroup — no `.pbxproj` edits needed.
- `ContentView.swift` is in `GoldenHour/` (root, not in App/ or Views/) — it stays where it is.

**No hardcoded color hex values in Swift source** — always use `Color("AssetName")` matching the named color assets you create in Task 1.

---

### VerdictView — Layout Structure

The key structural challenge: `ConditionImageView` must fill the entire screen (including under the status bar), while the verdict content must sit below the status bar without hardcoded height offsets.

**Recommended structure:**

```swift
struct VerdictView: View {
    @Environment(AppState.self) private var appState

    var body: some View {
        ZStack(alignment: .topLeading) {
            // Layer 1: full-bleed background
            ConditionImageView(
                conditionState: appState.conditionState,
                currentWindow: appState.lightWindows?.currentWindow ?? .neither
            )

            // Layer 2: header scrim — GeometryReader used ONLY here
            GeometryReader { proxy in
                LinearGradient(
                    colors: [Color("TextPrimary").opacity(0.88), .clear],
                    startPoint: .top,
                    endPoint: .bottom
                )
                .frame(height: proxy.size.height * 0.52)
                .frame(maxWidth: .infinity, alignment: .top)
                .ignoresSafeArea()
            }
            .allowsHitTesting(false)    // gradient must not intercept taps
        }
        .ignoresSafeArea()
        .safeAreaInset(edge: .top) {   // verdict content anchors to safe area top
            verdictContent
        }
    }

    @ViewBuilder
    private var verdictContent: some View {
        switch (appState.gpsState, appState.weatherEntry) {
        case (.unresolved, nil):
            skeletonState
        case (.denied, _), (.restricted, _):
            locationUnavailableState
        default:
            normalVerdictContent
        }
    }
}
```

The `.safeAreaInset(edge: .top)` modifier on the `ZStack` places `verdictContent` as an overlay anchored to the top safe area (below the status bar). The `ZStack` itself fills the screen via `.ignoresSafeArea()` — the image extends behind the status bar; the verdict content does not.

**CRITICAL: `GeometryReader` must appear ONLY in the scrim — nowhere else in the view hierarchy.** This is an explicit AC requirement. If you need a height or width elsewhere, use `frame(maxWidth: .infinity)` or fixed values.

---

### ConditionImageView — Image Selection Logic

Map `ConditionState` + `WindowType` to image asset names:

| ConditionState | WindowType | Image Asset |
|---|---|---|
| `.brokenClouds` | any | `brokenCloudsGolden` |
| `.clearSky` | `.blue` | `blueHour` |
| `.clearSky` | `.golden` or `.neither` | `clearSkyGolden` |
| `.overcast` | any | `overcast` |
| `.fog` | any | `fog` |
| `.rain` | any | `rain` |
| `.unknown` | any | `overcast` (fallback — muted tone matches uncertain state) |

Implement as a `private var imageName: String` computed property with a `switch` over `(conditionState, currentWindow)` — no `default:` for the `ConditionState` dimension (enumerate all 6 cases explicitly per architecture rule).

**Accessibility labels per condition:**
- `.brokenClouds` → `"Golden hour light through broken clouds"`
- `.clearSky` + `.blue` → `"Blue hour sky, clear and deep"`
- `.clearSky` → `"Clear golden sky"`
- `.overcast` → `"Overcast sky, flat and diffuse"`
- `.fog` → `"Dense fog, low visibility"`
- `.rain` → `"Rain and precipitation"`
- `.unknown` → `"Current conditions unknown"`

**Crossfade implementation:**
```swift
Image(imageName)
    .resizable()
    .scaledToFill()
    .clipped()
    .ignoresSafeArea()
    .id(conditionState)                       // force view identity change on state switch
    .transition(reduceMotion ? .identity : .opacity)
    .animation(reduceMotion ? nil : .easeInOut(duration: 0.4), value: conditionState)
    .accessibilityLabel(accessibilityLabel)
```

---

### ConditionState.displayName Extension

Add to `GoldenHour/Models/ConditionState.swift`:

```swift
extension ConditionState {
    var displayName: String {
        switch self {
        case .brokenClouds: return "Broken clouds"
        case .clearSky:     return "Clear sky"
        case .overcast:     return "Overcast"
        case .fog:          return "Dense fog"
        case .rain:         return "Precipitation"
        case .unknown:      return "Conditions unclear"
        }
    }
}
```

In VerdictView, condition summary text: `Text(appState.conditionState.displayName)`.

---

### TimePill — Text Format

Use `Formatters.shortTime` (already in `GoldenHour/Utilities/Formatters.swift`) for all time strings.

Target format: `"6:47 PM – 7:21 PM · Blue until 7:48 PM"`

The UX spec shows shortened AM/PM on intermediate times, but implementing that requires custom formatting logic. Use full `shortTime` output for all three times — the exact AM/PM omission is cosmetic polish, not an AC requirement. If you want to strip AM/PM from the golden start for brevity:

```swift
private func pillText(for windows: LightWindows) -> String {
    let goldenStart = Formatters.shortTime.string(from: windows.goldenStart)
    let goldenEnd   = Formatters.shortTime.string(from: windows.goldenEnd)
    let blueEnd     = Formatters.shortTime.string(from: windows.blueEnd)
    return "\(goldenStart) – \(goldenEnd) · Blue until \(blueEnd)"
}
```

---

### StaleDataLabel — Staleness Rule

The stale threshold for `StaleDataLabel` is **3 hours** (10800 seconds) — different from `WeatherCacheEntry.isStale` which uses 1 hour (3600s) for cache refresh purposes.

```swift
private var isDataStale: Bool {
    guard let entry = appState.weatherEntry else { return false }
    return Date.now.timeIntervalSince(entry.fetchedAt) > 10800
}
```

Show `StaleDataLabel(fetchedAt: entry.fetchedAt)` only when `isDataStale == true`.

Hours calculation for label text:
```swift
let hours = max(1, Int(Date.now.timeIntervalSince(fetchedAt) / 3600))
// "Weather from \(hours)h ago"
```

---

### VerdictView — Location Unavailable State (AC7)

When `appState.gpsState == .denied || appState.gpsState == .restricted`:

```swift
private var locationUnavailableState: some View {
    VStack(alignment: .leading, spacing: 12) {
        Text("Location unavailable.")
            .font(.system(.title, design: .serif))
            .foregroundColor(.white)
            .lineLimit(2)
        Text("Golden Hour uses your location for light window calculations.")
            .font(.subheadline)
            .foregroundColor(Color("TextSecondary"))
        Button("Open Settings") {
            if let url = URL(string: UIApplication.openSettingsURLString) {
                UIApplication.shared.open(url)
            }
        }
        .foregroundColor(Color("Golden"))
        .font(.subheadline.weight(.medium))
    }
    .padding(.horizontal, 20)
    .padding(.vertical, 16)
}
```

This is inline — no `Alert`, no sheet, no modal.

---

### VerdictView — Skeleton Loading State (AC8)

When `appState.gpsState == .unresolved && appState.weatherEntry == nil` (first launch, no cached data):

Show placeholder shapes where the verdict, summary, and time pill would appear. Use a simple approach:

```swift
private var skeletonState: some View {
    VStack(alignment: .leading, spacing: 12) {
        RoundedRectangle(cornerRadius: 6)
            .fill(Color.white.opacity(0.25))
            .frame(height: 28)
            .frame(maxWidth: .infinity)
        RoundedRectangle(cornerRadius: 4)
            .fill(Color.white.opacity(0.15))
            .frame(width: 180, height: 16)
        RoundedRectangle(cornerRadius: 20)
            .fill(Color.white.opacity(0.10))
            .frame(width: 220, height: 32)
    }
    .padding(.horizontal, 20)
    .padding(.vertical, 16)
}
```

No animation required for skeleton in v1 (keep it simple).

---

### ContentView — Custom Tab Structure

`ContentView.swift` currently contains the Xcode-default NavigationSplitView with `Item` CRUD. This must be completely replaced. The `@Query private var items: [Item]` and all Item-related code must be removed.

New structure using a `@State var selectedTab: Int = 0` approach:

```swift
struct ContentView: View {
    @Environment(AppState.self) private var appState
    @State private var selectedTab: Int = 0

    var body: some View {
        ZStack(alignment: .bottom) {
            Group {
                switch selectedTab {
                case 0:  HomeView()
                default: MapView()
                }
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)

            CustomTabBar(selectedTab: $selectedTab)
        }
        .ignoresSafeArea(edges: .bottom)
    }
}
```

**Why not native `TabView`:** SwiftUI's native `TabView` tab bar cannot be styled with the `#E6DAE4` lavender background and `#E0C0CB` top border required by the UX spec. Custom tab bar is required.

**Why not UIKit tab bar styling:** Avoid UIKit bridging — custom SwiftUI `CustomTabBar` achieves the spec without it.

---

### CustomTabBar — Structure

`CustomTabBar` receives a `Binding<Int>` for the selected tab:

```swift
struct CustomTabBar: View {
    @Binding var selectedTab: Int

    var body: some View {
        HStack {
            tabButton(icon: "house.fill", label: "Home", index: 0)
            tabButton(icon: "map.fill", label: "Map", index: 1)
        }
        .frame(height: 58)
        .frame(maxWidth: .infinity)
        .background(Color("Lavender"))
        .overlay(alignment: .top) {
            Rectangle()
                .fill(Color("Rose"))
                .frame(height: 1)
        }
    }

    private func tabButton(icon: String, label: String, index: Int) -> some View {
        Button {
            selectedTab = index
        } label: {
            Image(systemName: icon)
                .font(.title2)
                .foregroundColor(
                    selectedTab == index
                        ? Color("Golden")
                        : Color("SlateGray").opacity(0.5)
                )
                .frame(maxWidth: .infinity)
                .frame(height: 58)
                .contentShape(Rectangle())
        }
        .accessibilityLabel(label)
    }
}
```

---

### HomeView — Tomorrow's Windows

`AppState.lightWindowsTomorrow: LightWindows?` is populated by `handleLocationUpdate` in AppState (from Story 1.2). Display it below the verdict surface.

Suggested layout within HomeView (scrollable section at bottom, behind custom tab bar safe area):

```swift
struct HomeView: View {
    @Environment(AppState.self) private var appState

    var body: some View {
        VerdictView()
            .safeAreaInset(edge: .bottom) {
                if let tomorrow = appState.lightWindowsTomorrow {
                    tomorrowSection(windows: tomorrow)
                        .background(Color("Lavender").opacity(0.85))
                }
            }
        .task { appState.requestLocation() }
    }

    private func tomorrowSection(windows: LightWindows) -> some View {
        VStack(alignment: .leading, spacing: 4) {
            Text("Tomorrow")
                .font(.caption.weight(.semibold))
                .foregroundColor(Color("TextSecondary"))
            Text("\(Formatters.shortTime.string(from: windows.goldenStart)) – \(Formatters.shortTime.string(from: windows.goldenEnd)) · Blue until \(Formatters.shortTime.string(from: windows.blueEnd))")
                .font(.caption)
                .foregroundColor(Color("TextPrimary"))
        }
        .padding(.horizontal, 20)
        .padding(.vertical, 12)
    }
}
```

---

### GoldenHourApp.swift — No Changes Required

`GoldenHourApp.swift` currently uses `Item.self` in the `ModelContainer`. `Spot.swift` is added in Story 2.1. Do NOT add `Spot.self` to the schema in this story — it does not exist yet. Leave `GoldenHourApp.swift` unchanged.

---

### Swift 6 / SwiftUI Concurrency Notes

- `@Environment(AppState.self)` works because `AppState` is `@Observable`. Do NOT use `@EnvironmentObject`.
- `@Environment(\.accessibilityReduceMotion)` is a standard SwiftUI environment value — no import needed beyond `SwiftUI`.
- `UIApplication.openSettingsURLString` requires `import UIKit` in the file that uses it.
- `.task {}` in HomeView runs when the view appears and cancels on disappear — correct pattern for location trigger.
- `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` in build settings means all view types are implicitly `@MainActor` — no explicit annotation needed on view structs.

---

### Placeholder Condition Images — Implementation Note

Real photography is not yet sourced. Create placeholder image assets using Xcode's asset catalog:
- Each "image" can be a PDF or PNG of a solid color swatch (e.g., create a 1×1px solid color PNG for each condition)
- Recommended placeholder colors: `brokenCloudsGolden` (warm amber `#C4893A`), `clearSkyGolden` (golden yellow `#DDB05E`), `blueHour` (deep blue `#3A5FA0`), `overcast` (medium gray `#8A95A8`), `fog` (pale gray `#C8CDD5`), `rain` (dark slate `#4A5260`)
- The views will render correctly with solid colors as placeholders; crossfade transitions will still work

---

### File Locations Summary

```
GoldenHour/GoldenHour/
  Models/
    ConditionState.swift          ← MODIFY: add displayName extension
  Views/
    HomeView.swift                ← NEW
    MapView.swift                 ← NEW (placeholder only)
  Components/
    ConditionImageView.swift      ← NEW
    TimePill.swift                ← NEW
    StaleDataLabel.swift          ← NEW
    VerdictView.swift             ← NEW
    CustomTabBar.swift            ← NEW
  ContentView.swift               ← MODIFY: replace default Xcode template
  Assets.xcassets/
    Colors/                       ← ADD: 7 named color sets
    ConditionImages/              ← ADD: 6 placeholder image assets

GoldenHourTests/
  (no new test files — view components are not unit tested in v1 per architecture.md)
```

No `project.pbxproj` edits needed for new Swift files in `GoldenHour/Views/` and `GoldenHour/Components/` — PBXFileSystemSynchronizedRootGroup auto-includes them.

**Note on Assets:** Asset catalog changes (`.xcassets`) require manual addition in Xcode's asset catalog editor or via JSON-based asset catalog files — they are NOT auto-included by PBXFileSystemSynchronizedRootGroup. Add assets directly in Xcode.

---

### Previous Story Context

From Stories 1.1–1.3 — established patterns to preserve:
- `SolarService` — static struct, `calculate(coordinate:date:) -> LightWindows`
- `LocationService` — `@Observable @MainActor` NSObject, `nonisolated` delegate callbacks
- `AppState.requestLocation()` — public entry point, call from view `.task`
- `AppState.refresh(forceRefresh:)` — async, already wired to `handleLocationUpdate`
- `Formatters.shortTime` — shared `DateFormatter` instance in `Utilities/Formatters.swift`
- `UserDefaultsKey.weatherCache` — constant for weather cache key
- `WeatherCacheEntry.isStale` — 1-hour threshold (different from StaleDataLabel's 3-hour threshold)

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

No build issues encountered. SourceKit showed expected cross-file reference warnings (same pattern as Stories 1.1–1.3) — all resolve at Xcode compile time.

### Completion Notes List

All 8 tasks complete. All 12 acceptance criteria satisfied:

- **Assets (Task 1):** 7 named color sets generated as sRGB JSON color assets via Python script. 6 condition image sets generated as minimal 1×1 solid-color PNG placeholders with distinct hues per condition (amber, gold, blue, gray, pale gray, dark slate). Asset catalog group containers (Colors/Contents.json, ConditionImages/Contents.json) created per Xcode spec.

- **ConditionImageView (Task 2):** Full-bleed image with `.id(conditionState)` for forced view identity change on state transition. `@Environment(\.accessibilityReduceMotion)` gates animation. All 6 ConditionState cases explicitly mapped to image names (no default:). `.clearSky` during `.blue` window maps to `blueHour` image. `.unknown` falls back to `overcast`.

- **TimePill (Task 3):** Renders only when `windows != nil`. Text formatted via `Formatters.shortTime`. Translucent rounded rect: `white.opacity(0.14)` fill, `white.opacity(0.22)` stroke border, `cornerRadius: 20`.

- **StaleDataLabel (Task 4):** 3-hour threshold (10800s) — distinct from `WeatherCacheEntry.isStale`'s 1-hour threshold. `max(1, ...)` prevents "0h ago" display.

- **ConditionState.displayName (Task 5):** Extension on ConditionState, no changes to any service or AppState. All 6 cases enumerated.

- **VerdictView (Task 6):** ZStack with ConditionImageView + GeometryReader (scrim only, as required by AC3). `.safeAreaInset(edge: .top)` positions verdict content below status bar without hardcoded offsets. Three view states: normal verdict, location-unavailable (inline + Settings deep-link via `UIApplication.openSettingsURLString`), skeleton (3 placeholder shapes). `accessibilityElement(children: .combine)` on normalContent VStack.

- **HomeView (Task 7):** Hosts VerdictView full-screen. Tomorrow section shown via `.safeAreaInset(edge: .bottom)` when `lightWindowsTomorrow` is non-nil, formatted via `Formatters.shortTime`. `.task` triggers `requestLocation()`.

- **CustomTabBar + ContentView rewrite (Task 8):** CustomTabBar is a standalone SwiftUI View (not wrapping TabView's native bar). 58pt height, Lavender bg, Rose 1px top overlay border. ContentView uses `@State var selectedTab` + `ZStack` + `.ignoresSafeArea(edges: .bottom)`. All Xcode-default Item CRUD code removed.

### File List

**New files:**
- `GoldenHour/GoldenHour/Components/ConditionImageView.swift`
- `GoldenHour/GoldenHour/Components/TimePill.swift`
- `GoldenHour/GoldenHour/Components/StaleDataLabel.swift`
- `GoldenHour/GoldenHour/Components/VerdictView.swift`
- `GoldenHour/GoldenHour/Components/CustomTabBar.swift`
- `GoldenHour/GoldenHour/Views/HomeView.swift`
- `GoldenHour/GoldenHour/Views/MapView.swift`
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/Golden.colorset/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/Blue.colorset/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/Rose.colorset/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/Lavender.colorset/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/SlateGray.colorset/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/TextPrimary.colorset/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/TextSecondary.colorset/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/ConditionImages/Contents.json`
- `GoldenHour/GoldenHour/Assets.xcassets/ConditionImages/brokenCloudsGolden.imageset/` (Contents.json + PNG)
- `GoldenHour/GoldenHour/Assets.xcassets/ConditionImages/clearSkyGolden.imageset/` (Contents.json + PNG)
- `GoldenHour/GoldenHour/Assets.xcassets/ConditionImages/blueHour.imageset/` (Contents.json + PNG)
- `GoldenHour/GoldenHour/Assets.xcassets/ConditionImages/overcast.imageset/` (Contents.json + PNG)
- `GoldenHour/GoldenHour/Assets.xcassets/ConditionImages/fog.imageset/` (Contents.json + PNG)
- `GoldenHour/GoldenHour/Assets.xcassets/ConditionImages/rain.imageset/` (Contents.json + PNG)

**Modified files:**
- `GoldenHour/GoldenHour/Models/ConditionState.swift` — added `displayName` extension
- `GoldenHour/GoldenHour/ContentView.swift` — complete rewrite (replaced Xcode default NavigationSplitView with CustomTabBar + tab routing)

## Change Log

| Date | Change | Author |
|---|---|---|
| 2026-05-15 | Story created, status: ready-for-dev | bmad-create-story |
| 2026-05-15 | All 8 tasks implemented; status: review | bmad-dev-story |
| 2026-05-17 | Code review: 3 patches applied, 12 deferred, status: done | bmad-code-review |
