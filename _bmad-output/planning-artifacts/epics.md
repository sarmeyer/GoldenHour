---
stepsCompleted: ["step-01-validate-prerequisites", "step-02-design-epics", "step-03-create-stories", "step-04-final-validation"]
inputDocuments:
  - "_bmad-output/planning-artifacts/prd.md"
  - "_bmad-output/planning-artifacts/architecture.md"
  - "_bmad-output/planning-artifacts/ux-design-specification.md"
---

# Golden Hour - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for Golden Hour, decomposing the requirements from the PRD, UX Design, and Architecture requirements into implementable stories.

## Requirements Inventory

### Functional Requirements

FR1: The app calculates the golden hour window (start time, end time, duration) for the user's current GPS location for today
FR2: The app calculates the blue hour window (start time, end time, duration) for the user's current GPS location for today
FR3: The app displays tomorrow's golden and blue hour window times for advance planning
FR4: Light window calculations update when the user's GPS location changes significantly between sessions
FR5: The app fetches current weather conditions from Open-Meteo for the user's GPS location
FR6: The app maps raw weather data (cloud cover, precipitation probability, visibility, weather code) to a photographer-specific condition state (broken clouds, clear sky, overcast, fog, precipitation)
FR7: The app displays a human-language condition summary using opinionated photographic framing — not raw meteorological data
FR8: The app displays a synthesized go/no-go verdict combining light window timing and condition quality into a single actionable sentence
FR9: The app displays last-successfully-fetched condition data with a visible timestamp when a fresh fetch is unavailable
FR10: The app communicates clearly when condition data is unavailable, distinguishing between offline and stale-but-present states
FR11: The app displays a map centered on the user's current GPS location
FR12: Users can browse the map to explore potential shooting locations
FR13: Users can save a map location as a personal shooting spot with a custom name
FR14: Users can add or edit a text note on a saved spot
FR15: Users can assign a location type category to a saved spot (coastal, urban, forest, open field, elevated, other) for use in tip generation
FR16: Users can view all saved spots on the map
FR17: Users can delete a saved spot
FR18: Saved spots persist locally on-device across app sessions with no cloud sync
FR19: The saved spot data model records a last-visited timestamp for each spot
FR20: The app displays shooting tips for a selected location keyed to the current light window type and condition state
FR21: The app displays camera settings (ISO, aperture, shutter speed) as starting-point recommendations within tip content
FR22: Tips display immediately using static fallback content when a location is selected, before any LLM response is available
FR23: The app upgrades displayed tips to LLM-generated content when an Anthropic API response arrives within a defined timeout
FR24: LLM-generated tips reference the specific light window type, condition state, and location type — not generic photography advice
FR25: The app selects the appropriate static fallback tip set based on current condition state and light window type
FR26: Tips adapt honestly to poor conditions — reframing (e.g., soft light for portraits) rather than overstating photographic potential
FR27: Light window calculations are fully available without network connectivity
FR28: Saved spots are fully available without network connectivity
FR29: The app displays a coherent, useful state when offline — static tips always available; weather and LLM content degrade with clear messaging
FR30: The app communicates which specific features are unavailable and why when connectivity is absent
FR31: The app requests location permission and uses GPS coordinates as the primary input for all location-dependent calculations
FR32: The app displays a location-unavailable state with a clear explanation and fallback option when location permission is denied or GPS is unavailable
FR33: The app stores the Anthropic API key in iOS Keychain — not in plaintext or hardcoded in the binary
FR34: Users can manually trigger a refresh of current weather conditions

### NonFunctional Requirements

NFR1: Home screen (light window times + condition summary) loads within 2 seconds from cold launch
NFR2: Map renders and saved spots display within 3 seconds on standard connectivity
NFR3: Static tips display within 200ms of location selection
NFR4: LLM tip requests time out after 4 seconds; app remains on static tips without showing an error state
NFR5: Weather data cached with minimum 60-minute TTL; fresh fetches not triggered more than once per hour per location
NFR6: Light window calculations complete in under 100ms
NFR7: The Anthropic API key is stored in iOS Keychain — not in the app bundle, source code, or any plaintext configuration file
NFR8: The app transmits user data only to Open-Meteo (GPS coordinates) and Anthropic (condition state + location type) — no other external data transmission
NFR9: The app collects no analytics, no external crash reporting, and no usage telemetry
NFR10: Weather fetch failure must not prevent the app from displaying cached conditions (with timestamp) or a clear unavailable state
NFR11: LLM API failure must not prevent the app from displaying static fallback tips — the tip section is never empty
NFR12: Map tile unavailability must not prevent access to the saved spots list view
NFR13: Open-Meteo and Anthropic API responses are cached to avoid redundant calls within a session
NFR14: All text scales with the user's iOS Dynamic Type setting — no fixed font sizes
NFR15: The app is navigable via VoiceOver through standard SwiftUI accessibility semantics

### Additional Requirements

- **Project initialization:** Xcode > File > New Project > iOS > App (SwiftUI interface, SwiftData storage, Swift language) is the first implementation story before any feature work
- **SPM dependency:** Add Sunlight (https://github.com/BinaryBirds/Sunlight) as a Swift Package dependency for solar calculations
- **Architecture pattern:** MVVM with @Observable macro; AppState @MainActor singleton injected via .environment; no Combine required
- **ConditionState enum must be defined first** — it is a build-time dependency for TipsService, VerdictView, ConditionImageView, and all LLM prompts; canonical raw values are load-bearing strings in UserDefaults keys and StaticTips.json
- **GPSState enum definition required** (cases: unresolved, granted, denied, restricted) — referenced by AppState but not formally defined in PRD; must be added in first story
- **Keychain bootstrap sequence:** GoldenHourApp.init() calls KeychainService.bootstrapIfNeeded() synchronously before the view hierarchy renders; reads APIKeys.swift constants on first launch, writes to Keychain, reads from Keychain thereafter
- **APIKeys.swift must be gitignored;** APIKeys.swift.template committed with placeholder values and setup instructions
- **Maptiler API key:** managed via same APIKeys.swift → Keychain flow alongside Anthropic key
- **Canonical LLM prompt template** must be defined in Utilities/Prompts.swift as a static constant — primary quality control mechanism for tip tone, camera settings format, and negative-condition handling; this is an important gap to resolve in the first implementation story
- **Static tips stored in Resources/StaticTips.json** keyed by "{conditionState.rawValue}_{windowType.rawValue}"; TipContent shape: body String + cameraSettings String; covers all 8 required combinations (5 conditions × golden/blue, with full golden set of 5 and blue set of 3 as specified in PRD)
- **UserDefaultsKey registry:** centralized in Utilities/UserDefaultsKey.swift — no agent should hardcode a key string
- **Error absorption pattern:** all services absorb errors internally, return optional/cached values; never propagate throws to the view layer; UI never shows an error state for network failures
- **Unit tests required** for SolarService (golden/blue hour window accuracy vs. known reference times), ConditionMapper (all weather code → ConditionState mappings), TipsService (correct static tip selection for all 8 valid condition × window combinations)
- **Implementation sequence** must respect build dependencies: ConditionState/LocationType/GPSState enums → AppState → KeychainService → SolarService → LocationService → WeatherService → HomeView → SwiftData Spot model → MapView → TipsService (static) → TipsService (LLM) → visual polish

### UX Design Requirements

UX-DR1: VerdictView — hero component for the home screen: full-bleed ConditionImageView background, top-anchored dark header scrim (LinearGradient rgba(26,22,18,0.88) → transparent at ~52% screen height via GeometryReader), verdict sentence (New York serif, .title Dynamic Type style, warm white, below status bar), condition summary (#DDB05E, .subheadline, SF Pro Medium, 9pt below verdict), TimePill (8pt below condition summary), StaleDataLabel (below TimePill when stale or offline); combined accessibilityElement reads all content as one unit

UX-DR2: ConditionImageView — background photography layer keyed to ConditionState enum; 6 images minimum bundled in Assets.xcassets/ConditionImages/ named exactly: brokenCloudsGolden, clearSkyGolden, blueHour, overcast, fog, rain; .resizable() + .scaledToFill() + .clipped() + .ignoresSafeArea(); crossfade transition when condition state changes between sessions; instant swap when Reduce Motion is enabled; descriptive accessibilityLabel per image (e.g. "Golden hour light through broken clouds")

UX-DR3: TimePill — rounded rectangle container (rgba(247,243,239,0.14) fill, rgba(247,243,239,0.22) border, cornerRadius: 20); single text line showing golden + blue hour range ("6:47 – 7:21 PM · Blue until 7:48"); .caption Dynamic Type style; two variants: dark-scrim (light text) and light-surface (dark text)

UX-DR4: SpotDetailSheet — bottom sheet for saved spot detail: .sheet with .presentationDetents([.medium, .large]), opens at .medium; surface #E6DAE4 at 96% opacity, cornerRadius: 30 top corners; handle (36×4pt, #E0C0CB, centered, 14pt from top); inner padding 24pt horizontal / 20pt vertical; spot name (.title3, SF Pro Medium), location type badge (#DDB05E tint, rounded label), TipBlock, optional LLMTipBlock, all wrapped in ScrollView

UX-DR5: TipBlock — static tip display: section label "For tonight's conditions" (10pt, #6B6560, uppercase, .caption2 Dynamic Type); tip body in SF Pro Text (.body Dynamic Type, conversational, present tense, direct address); camera settings row in SF Pro Mono (.caption Dynamic Type) with accessibilityLabel "Starting exposure: ISO [x], f/[x], 1/[x]th of a second"

UX-DR6: LLMTipBlock — LLM augmentation block: "Tonight specifically" section label (10pt, #9B6A2E, uppercase); 1pt #E0C0CB divider above; LLM tip body (SF Pro Text, .body); hidden by default; appears via opacity-only transition (no slide or bounce) when LLM response received; Reduce Motion: appears instantly; never replaces TipBlock content; remains hidden if no response within 4s; no spinner, no error message shown

UX-DR7: SpotAnnotation — custom MapKit annotation: 16pt filled circle + subtle drop shadow; unselected state: #8A95A8 at 70% opacity; selected state: #DDB05E at 100% opacity with slight scale-up; 44×44pt touch target via .contentShape; accessibilityLabel = spot name; accessibilityHint = "Double tap to view tips"

UX-DR8: SaveSpotSheet — spot creation form: .medium detent only; spot name TextField (required; Save button disabled when empty); note TextEditor (optional, ~3 lines, placeholder "e.g. west-facing, try at golden hour"); location type Picker (6 options matching LocationType enum cases); Primary Save button (#DDB05E background, #1A1612 label, cornerRadius: 12, 16pt SF Pro Semibold); Secondary Cancel button (no fill, #DDB05E label)

UX-DR9: StaleDataLabel — muted staleness indicator: single text line (11pt, #6B6560, italic, .caption2 Dynamic Type); shown when weather cache age > 3 hours or when offline: "Weather from Xh ago"; GPS-acquiring variant: "Using last location"; no icon, no warning color

UX-DR10: CustomTabBar — lavender-surface tab bar replacing SwiftUI native: 58pt fixed height, #E6DAE4 background, 1px #E0C0CB top border; two tabs (Home, Map); active icon #DDB05E, inactive #8A95A8 at 50% opacity; .accessibilityLabel("Home") / .accessibilityLabel("Map") on each tab item

UX-DR11: Color system — all named colors defined as Asset Catalog named color assets (with light/dark slots for future-proofing): Golden #DDB05E, Blue #92A7D6, Rose #E0C0CB, Lavender #E6DAE4, SlateGray #8A95A8, TextPrimary #1A1612, TextSecondary #6B6560; no hardcoded hex values in Swift source

UX-DR12: Typography system — all text uses SwiftUI Dynamic Type semantic styles with no fixed point sizes: verdict → .title + New York font + lineLimit(2); condition summary → .subheadline; time pill → .caption; tip body → .body; camera settings → .caption + monospaced font; metadata/labels → .caption2; verify text never clips at Accessibility XXL sizes

UX-DR13: Empty states (all inline, never modal or full-screen): no saved spots → "No spots saved yet. Tap anywhere on the map to save a location." with tap action to SaveSpotSheet; location unavailable → inline message with Settings deep-link ("Open Settings"); first launch skeleton state on home screen while data loads; no network + no cache → "No conditions data available. Light window times will appear when connected."

UX-DR14: Destructive action pattern for spot deletion: accessible via long-press on spot annotation or context menu; Alert with title "Delete [Spot Name]?", message "This can't be undone.", buttons "Delete" (destructive role) / "Cancel" (default/cancel role); on confirm: remove from SwiftData, remove annotation from map, dismiss sheet if open

UX-DR15: Accessibility implementation — color contrast: warm white on scrim ≥ 4.5:1 (tune scrim opacity per condition image, verify in Xcode Accessibility Inspector); #1A1612 on #E6DAE4 passes 4.5:1 for all text; #DDB05E on #E6DAE4 ≥ 3:1 for large text only; touch targets ≥ 44×44pt for all interactive elements; @Environment(\.accessibilityReduceMotion) checked for all animations; @Environment(\.dynamicTypeSize) used to adjust layout constraints at accessibility3+ sizes

UX-DR16: Header scrim and layout geometry — scrim as LinearGradient inside GeometryReader (.frame(height: proxy.size.height * 0.52)); avoid GeometryReader elsewhere in the view hierarchy; condition image uses .ignoresSafeArea(); verdict content respects .safeAreaInset(edge: .top); status bar set to light content (white/warm white)

UX-DR17: Feedback patterns — no alerts, no modals, no banners for degraded-but-functional states (stale weather, offline, LLM timeout, GPS acquiring); these states surface only as StaleDataLabel or hidden LLMTipBlock; Alerts reserved exclusively for irreversible actions (spot deletion); inline settings deep-link for location-unavailable state; no "Oops!" or apology language in any copy

UX-DR18: Copy voice rules — no exclamation marks; no "Please" in form labels; confident and declarative for verdicts; honest and direct for negative states; no hedging language; static tip copy in present tense, direct address, written for someone already on location

### FR Coverage Map

| FR | Epic | Description |
|---|---|---|
| FR1 | Epic 1 | Golden hour window calculation (today) |
| FR2 | Epic 1 | Blue hour window calculation (today) |
| FR3 | Epic 1 | Tomorrow's light window display |
| FR4 | Epic 1 | Location-change recalculation |
| FR5 | Epic 1 | Open-Meteo weather fetch |
| FR6 | Epic 1 | ConditionState mapping |
| FR7 | Epic 1 | Opinionated condition summary display |
| FR8 | Epic 1 | Synthesized go/no-go verdict |
| FR9 | Epic 1 | Stale condition data with timestamp |
| FR10 | Epic 1 | Offline/stale unavailability communication |
| FR11 | Epic 2 | GPS-centered map display |
| FR12 | Epic 2 | Map browsing |
| FR13 | Epic 2 | Save spot with custom name |
| FR14 | Epic 2 | Spot note add/edit |
| FR15 | Epic 2 | Location type assignment |
| FR16 | Epic 2 | View all saved spots on map |
| FR17 | Epic 2 | Delete saved spot |
| FR18 | Epic 2 | Local persistence, no cloud sync |
| FR19 | Epic 2 | lastVisited timestamp on spots |
| FR20 | Epic 2 | Shooting tips keyed to condition + light window |
| FR21 | Epic 2 | Camera settings in tips |
| FR22 | Epic 2 | Immediate static tip display |
| FR23 | Epic 2 | LLM tip upgrade within 4s |
| FR24 | Epic 2 | LLM tips reference specific context |
| FR25 | Epic 2 | Correct static tip selection logic |
| FR26 | Epic 2 | Honest negative-condition tips |
| FR27 | Epic 1 | Light window calculations offline-capable |
| FR28 | Epic 2 | Saved spots offline-capable |
| FR29 | Epic 1 & 2 | Coherent offline state (home in E1; tips/spots in E2) |
| FR30 | Epic 1 & 2 | Offline feature unavailability messaging (home in E1; tips in E2) |
| FR31 | Epic 1 | GPS permission + coordinates as primary input |
| FR32 | Epic 1 | Location-unavailable state + fallback |
| FR33 | Epic 1 | Anthropic API key in Keychain |
| FR34 | Epic 1 | Manual weather refresh |

## Epic List

### Epic 1: Project Foundation & Home Screen Verdict

Users can open the app and immediately read a photographic go/no-go verdict for tonight's conditions at their GPS location — including golden and blue hour window times — with graceful behavior when offline, when conditions are stale, or when location is unavailable. This epic establishes the complete project foundation and delivers the single most important user experience: the verdict.

**FRs covered:** FR1, FR2, FR3, FR4, FR5, FR6, FR7, FR8, FR9, FR10, FR27, FR29 (home screen), FR30 (home screen), FR31, FR32, FR33, FR34

**UX-DRs covered:** UX-DR1, UX-DR2, UX-DR3, UX-DR9, UX-DR10, UX-DR11, UX-DR12, UX-DR16, UX-DR17

**NFRs addressed:** NFR1, NFR5, NFR6, NFR7, NFR8, NFR9, NFR10, NFR14, NFR15

---

### Epic 2: Location Scouting & On-Location Shooting Tips

Users can browse the map, save personal shooting spots with notes and location types, and tap a saved spot to receive instant static shooting tips plus contextual LLM-generated tips specific to the current condition and light window. Spot management works fully offline. This epic delivers the complete on-location value: scouting, saving, and guidance.

**FRs covered:** FR11, FR12, FR13, FR14, FR15, FR16, FR17, FR18, FR19, FR20, FR21, FR22, FR23, FR24, FR25, FR26, FR28, FR29 (tips/spots), FR30 (tips)

**UX-DRs covered:** UX-DR4, UX-DR5, UX-DR6, UX-DR7, UX-DR8, UX-DR13, UX-DR14, UX-DR15, UX-DR18

**NFRs addressed:** NFR2, NFR3, NFR4, NFR11, NFR12, NFR13

---

## Epic 1: Project Foundation & Home Screen Verdict

Users can open the app and immediately read a photographic go/no-go verdict for tonight's conditions at their GPS location — including golden and blue hour window times — with graceful behavior when offline, when conditions are stale, or when location is unavailable.

### Story 1.1: Project Initialization & Security Foundation

As a developer building the Golden Hour app,
I want the Xcode project initialized with all core types, utilities, and the Keychain security layer in place,
So that all subsequent stories have a consistent, dependency-safe foundation to build upon.

**Acceptance Criteria:**

**Given** a new Xcode project is created (File > New Project > iOS > App, SwiftUI interface, SwiftData storage, Swift language) with targets `GoldenHour` and `GoldenHourTests`
**When** the Sunlight Swift Package (https://github.com/BinaryBirds/Sunlight) is added via SPM
**Then** the project builds successfully on an iOS 18 simulator with no errors

**Given** `APIKeys.swift.template` is committed with empty string placeholders and setup instructions
**When** a developer clones the repository
**Then** `APIKeys.swift.template` is present; `APIKeys.swift` is absent (listed in `.gitignore`)

**Given** `APIKeys.swift` exists locally with real values for `APIKeys.anthropic` and `APIKeys.maptiler`
**When** `GoldenHourApp.init()` executes before the view hierarchy renders
**Then** `KeychainService.bootstrapIfNeeded()` is called synchronously, reads both keys, and writes them to iOS Keychain if not already present
**And** on all subsequent launches, `KeychainService.read(key:)` retrieves keys from Keychain — `APIKeys.swift` is never referenced again

**Given** `ConditionState.swift` is implemented
**When** the enum is inspected
**Then** it has exactly 6 cases with canonical raw values: `brokenClouds="broken_clouds"`, `clearSky="clear_sky"`, `overcast="overcast"`, `fog="fog"`, `rain="rain"`, `unknown="unknown"`, conforming to `String, Codable`
**And** `LocationType` has 6 cases (`coastal`, `urban`, `forest`, `openField`, `elevated`, `other`) with matching snake_case raw values
**And** `GPSState` has 4 cases: `unresolved`, `granted`, `denied`, `restricted`
**And** `LightWindows` struct has `goldenStart`, `goldenEnd`, `blueStart`, `blueEnd` (`Date`) and `currentWindow: WindowType` computed property (returns `.golden`, `.blue`, or `.neither`)

**Given** `AppState.swift` is implemented as `@Observable @MainActor class AppState`
**When** injected at root via `.environment`
**Then** it exposes: `conditionState: ConditionState = .unknown`, `lightWindows: LightWindows?`, `verdict: String = ""`, `weatherEntry: WeatherCacheEntry?`, `gpsState: GPSState = .unresolved`

**Given** Utilities group is created
**When** `UserDefaultsKey.swift`, `Formatters.swift`, `WithTimeout.swift`, and `Prompts.swift` are implemented
**Then** `UserDefaultsKey.weatherCache = "weatherCache"` and `UserDefaultsKey.tipsCache = "tipsCache_"` are the only UserDefaults key strings in the codebase
**And** `Formatters.shortTime` is a single shared `DateFormatter` instance (`timeStyle: .short`, `dateStyle: .none`)
**And** `withTimeout(seconds:operation:) -> T?` returns nil if the async operation exceeds the given duration
**And** `Prompts.systemPrompt` is a non-empty `static let String` defining the photography assistant persona with: output format constraints, camera settings requirement, honest negative-condition handling, and condition-reference requirement

### Story 1.2: Solar Calculation & Location Services

As a photographer using Golden Hour,
I want the app to determine my GPS location and calculate accurate golden and blue hour windows for today and tomorrow,
So that I always know exactly when the light windows occur at my current location.

**Acceptance Criteria:**

**Given** `SolarService.swift` wraps the Sunlight library
**When** `calculate(coordinate:date:) -> LightWindows` is called with valid coordinates and a date
**Then** it returns golden hour and blue hour start/end times within ±2 minutes of actual observed values for that location and date
**And** the calculation completes in under 100ms (NFR6)
**And** the method accepts any `Date` so tomorrow's windows can be calculated (FR3)

**Given** `SolarService.calculate` is called with today's date and current time falls within the golden hour window
**When** `LightWindows.currentWindow` is evaluated
**Then** it returns `.golden`; blue hour active → `.blue`; outside both → `.neither`

**Given** `SolarServiceTests.swift` exists in `GoldenHourTests`
**When** tests run
**Then** golden and blue hour calculations are validated against at least 3 known reference locations/dates, covering standard timezone, a DST transition date, and a high-latitude case (e.g., Reykjavik)

**Given** `LocationService.swift` manages `CLLocationManager`
**When** the user has not yet been prompted for location permission
**Then** `AppState.gpsState` is `.unresolved` and no calculations fire

**Given** the user grants location permission and `CLLocationManager` delivers a coordinate
**When** `LocationService` publishes the coordinate
**Then** `AppState.gpsState` becomes `.granted`
**And** `AppState` calls `SolarService.calculate(coordinate:date:)` for today and tomorrow, storing results in `AppState.lightWindows`

**Given** the user denies location permission
**When** `LocationService` receives the denial callback
**Then** `AppState.gpsState` becomes `.denied`; no solar calculation is attempted

**Given** location is restricted (MDM or parental controls)
**When** `LocationService` detects the restricted state
**Then** `AppState.gpsState` becomes `.restricted`

**Given** the app has a granted location and the user's position changes by more than 500m since the last calculation
**When** `LocationService` publishes the new coordinate
**Then** `AppState` recalculates light windows with the updated coordinates (FR4)

**Given** the device has no network connectivity
**When** `SolarService.calculate` is called with a valid coordinate
**Then** it completes successfully — solar calculation requires no network access (FR27)

### Story 1.3: Weather Fetch, Condition Mapping & Verdict Logic

As a photographer using Golden Hour,
I want the app to fetch current weather and interpret it photographically,
So that I receive a confident go/no-go verdict in opinionated photographic language rather than raw meteorological data.

**Acceptance Criteria:**

**Given** `WeatherResponse.swift` is implemented as a `Decodable` struct
**When** decoded from Open-Meteo JSON
**Then** `cloud_cover`, `precipitation_probability`, `visibility`, `weather_code` map correctly to camelCase Swift properties via `CodingKeys`

**Given** `WeatherCacheEntry.swift` is implemented as a `Codable` struct
**When** `fetchedAt` is more than 3600 seconds ago
**Then** `isStale` computed property returns `true`

**Given** `WeatherService.fetch(latitude:longitude:)` is called and a cache entry with age < 60 minutes exists in `UserDefaults`
**When** no explicit refresh was requested
**Then** the cached `WeatherResponse` is returned without a network call (NFR5)

**Given** the cache is stale or absent and the Open-Meteo API request succeeds
**When** `WeatherService.fetch` completes
**Then** the decoded response is stored in `UserDefaults` under `UserDefaultsKey.weatherCache` and returned
**And** the method never throws to callers — errors are absorbed internally (NFR10)

**Given** the API request fails and a cached entry exists
**When** `WeatherService.fetch` completes
**Then** the stale cached entry is returned and `AppState.weatherEntry` reflects it with its original `fetchedAt` timestamp

**Given** `ConditionMapper.map(_ response: WeatherResponse) -> ConditionState` is implemented
**When** called with any valid `WeatherResponse`
**Then** it returns the correct `ConditionState` for that weather data combination
**And** every `switch` over `ConditionState` enumerates all 6 cases explicitly — no `default:` clause
**And** any unmapped input returns `.unknown` — never a silent fallthrough

**Given** `ConditionMapper.verdict(for:windows:) -> String` is implemented
**When** condition is positive (e.g., `.brokenClouds`) during an active light window
**Then** the returned string is a confident, non-hedging single sentence (e.g., "Go tonight. Broken clouds — this is the good kind.")

**Given** `ConditionMapper.verdict` is called with a negative condition (e.g., `.overcast`)
**When** conditions are poor for golden hour
**Then** the verdict is honest and includes a falsifiable photographic reason (e.g., "Skip tonight. Heavy overcast flattens everything.")

**Given** `ConditionMapperTests.swift` exists in `GoldenHourTests`
**When** tests run
**Then** all valid `weather_code` ranges map to their expected `ConditionState`, boundary values are tested, and `.unknown` is returned for any unrecognized input

**Given** `AppState` has valid GPS coordinates and calls `refresh()`
**When** `WeatherService.fetch` succeeds and `ConditionMapper.map` runs
**Then** `AppState.conditionState`, `AppState.verdict`, and `AppState.weatherEntry` are updated on `@MainActor`

**Given** the user triggers a manual weather refresh (FR34) via a UI control
**When** the refresh fires
**Then** `WeatherService` ignores the current cache TTL and makes a fresh Open-Meteo API call

### Story 1.4: Home Screen — Verdict Display & Visual Design

As a photographer using Golden Hour,
I want to open the app and immediately see a confident go/no-go verdict over a condition-appropriate full-bleed photograph,
So that I can decide whether to go out in under 60 seconds without opening any other app.

**Acceptance Criteria:**

**Given** 6 condition photography images are added to `Assets.xcassets/ConditionImages/` named exactly: `brokenCloudsGolden`, `clearSkyGolden`, `blueHour`, `overcast`, `fog`, `rain`
**When** `ConditionImageView` renders for a given `ConditionState`
**Then** the correct image displays full-bleed (`.resizable()`, `.scaledToFill()`, `.clipped()`, `.ignoresSafeArea()`)
**And** when `ConditionState` changes, the image transitions via crossfade; when `@Environment(\.accessibilityReduceMotion)` is true, the new image appears instantly

**Given** all 7 named color assets are defined in `Assets.xcassets/Colors/` (Golden `#DDB05E`, Blue `#92A7D6`, Rose `#E0C0CB`, Lavender `#E6DAE4`, SlateGray `#8A95A8`, TextPrimary `#1A1612`, TextSecondary `#6B6560`)
**When** any component references a color
**Then** it uses the named asset — no hardcoded hex values appear in Swift source

**Given** `VerdictView` implements the header scrim as `LinearGradient` inside a `GeometryReader` (`.frame(height: proxy.size.height * 0.52)`, `rgba(26,22,18,0.88)` → transparent, top-anchored)
**When** the home screen renders
**Then** the verdict `Text` appears immediately below the status bar via `.safeAreaInset(edge: .top)` in New York serif, `.title` Dynamic Type style, warm white, `lineLimit(2)`
**And** the condition summary appears 9pt below in `#DDB05E`, `.subheadline` Dynamic Type style
**And** `GeometryReader` is used only for this scrim — nowhere else in the view hierarchy

**Given** `TimePill` is rendered below the condition summary
**When** golden and blue hour windows are available
**Then** it displays the full range on one line (e.g., "6:47 – 7:21 PM · Blue until 7:48") using `Formatters.shortTime`, in a rounded rectangle (`cornerRadius: 20`, translucent fill and border per UX spec)

**Given** `HomeView` includes tomorrow's light window display (FR3)
**When** the user views the home screen
**Then** tomorrow's golden and blue hour window times are visible (scrollable if needed) formatted via `Formatters.shortTime`

**Given** `AppState.weatherEntry.fetchedAt` is more than 3 hours ago, or the device is offline
**When** `HomeView` renders
**Then** `StaleDataLabel` appears below `TimePill` as "Weather from Xh ago" — `#6B6560`, `.caption2` Dynamic Type, italic — no icon, no warning color

**Given** `AppState.gpsState` is `.denied` or `.restricted`
**When** `HomeView` renders
**Then** an inline location-unavailable message appears with a tappable "Open Settings" deep-link (no modal, no blocking alert) (FR32)

**Given** `AppState.gpsState` is `.unresolved` and no cached data exists (first launch)
**When** `HomeView` renders
**Then** a skeleton loading state is shown — not a blank screen, not an error

**Given** `CustomTabBar` replaces the native SwiftUI `TabView` bar
**When** rendered
**Then** it is 58pt height, `#E6DAE4` background, 1px `#E0C0CB` top border
**And** active tab icon is `#DDB05E`; inactive is `#8A95A8` at 50% opacity
**And** each tab has `.accessibilityLabel("Home")` and `.accessibilityLabel("Map")` respectively

**Given** `VerdictView` uses `accessibilityElement(children: .combine)`
**When** VoiceOver focuses on the verdict surface
**Then** it reads verdict sentence + condition summary + light window times as one combined unit
**And** `ConditionImageView` has a descriptive `accessibilityLabel` (e.g., "Golden hour light through broken clouds")

**Given** cached weather data is available on cold launch
**When** the home screen appears
**Then** the verdict and condition image are visible within 2 seconds (NFR1) — no blank screen during data loading

**Given** all text elements use SwiftUI Dynamic Type semantic styles (no fixed point sizes)
**When** the system font size is set to Accessibility Extra Extra Extra Large
**Then** the verdict truncates at `lineLimit(2)` and no text clips or overflows its container

---

## Epic 2: Location Scouting & On-Location Shooting Tips

Users can browse the map, save personal shooting spots with notes and location types, and tap a saved spot to receive instant static shooting tips plus contextual LLM-generated tips specific to the current condition and light window. Spot management works fully offline.

### Story 2.1: Map Display & Spot Data Foundation

As a photographer using Golden Hour,
I want to see a map centered on my current location with my saved shooting spots marked on it,
So that I can visually locate and navigate to my personal spots.

**Acceptance Criteria:**

**Given** `Spot.swift` is implemented as a SwiftData `@Model` class
**When** the model is inspected
**Then** it has properties: `name: String`, `note: String?`, `locationType: LocationType`, `latitude: Double`, `longitude: Double`, `lastVisited: Date?`, `createdAt: Date`

**Given** `TipContent.swift` is implemented as a `Codable` struct
**When** decoded from `StaticTips.json`
**Then** it correctly maps `body: String` and `cameraSettings: String` fields

**Given** `ModelContainer(for: Spot.self)` is configured at `GoldenHourApp` entry point
**When** the app launches
**Then** SwiftData is initialized before any view that reads or writes `Spot` data
**And** no iCloud sync is configured (local persistence only) (FR18)

**Given** `MapView.swift` is implemented using MapKit with a Maptiler raster tile overlay (`MKTileOverlay` URL template using the Maptiler API key from `KeychainService`)
**When** the map tab is opened
**Then** the map renders with Maptiler tiles and OSM attribution displayed per Maptiler terms (NFR2: within 3 seconds on standard connectivity)

**Given** `AppState.gpsState` is `.granted`
**When** `MapView` appears
**Then** the map is centered on the user's current GPS coordinates (FR11)
**And** the user can pan and zoom freely to browse the map (FR12)

**Given** `SpotAnnotation.swift` is implemented as a custom MapKit annotation view
**When** a `Spot` exists in SwiftData
**Then** it appears on the map as a 16pt filled circle with a drop shadow
**And** its unselected state is `#8A95A8` at 70% opacity; selected state is `#DDB05E` at 100% opacity with a slight scale-up
**And** its touch target is at least 44×44pt via `.contentShape`
**And** its `accessibilityLabel` equals the spot's `name` and `accessibilityHint` is "Double tap to view tips"

**Given** SwiftData contains no `Spot` records
**When** the user opens the Map tab
**Then** an inline empty state message is displayed: "No spots saved yet. Tap anywhere on the map to save a location." (UX-DR13)
**And** no modal, no full-screen takeover

**Given** the device is offline
**When** map tiles are not cached
**Then** the saved spots list remains accessible (FR28, NFR12) — tile unavailability does not prevent spot access

### Story 2.2: Spot Management — Save, Edit & Delete

As a photographer using Golden Hour,
I want to save shooting locations with a name, note, and location type, and be able to edit or delete them,
So that I can build a personal library of spots I return to.

**Acceptance Criteria:**

**Given** the user long-presses a location on the map
**When** the long-press gesture is recognized
**Then** `SaveSpotSheet` is presented at `.medium` detent only (FR13)

**Given** `SaveSpotSheet.swift` is implemented
**When** rendered
**Then** it contains: a name `TextField` (required), a note `TextEditor` (optional, ~3 lines, placeholder "e.g. west-facing, try at golden hour"), a `Picker` for location type (6 options matching `LocationType` enum cases exactly), a primary Save button and a secondary Cancel button
**And** the Save button is disabled when the name field is empty

**Given** the user enters a name and taps Save
**When** `SaveSpotSheet` dismisses
**Then** a new `Spot` is written to SwiftData with the entered name, note (if provided), selected `LocationType`, the tapped map coordinate, and `createdAt = Date.now`
**And** the spot's `SpotAnnotation` appears on the map immediately (FR16)
**And** the spot persists across app restarts (FR18) and is available offline (FR28)

**Given** a spot exists and the user wants to edit its note
**When** the user taps an existing spot annotation to open its detail sheet
**Then** the note field is editable inline in the sheet and changes are saved to SwiftData on dismissal (FR14)

**Given** a spot exists
**When** the user long-presses its annotation or accesses a context menu
**Then** a "Delete" option is presented
**And** tapping "Delete" shows an `Alert` with title "Delete [Spot Name]?", message "This can't be undone.", buttons "Delete" (destructive role) / "Cancel" (default/cancel role) (UX-DR14)

**Given** the user confirms deletion in the Alert
**When** the action completes
**Then** the `Spot` is removed from SwiftData, its annotation is removed from the map, and any open detail sheet for that spot is dismissed (FR17)

**Given** the user taps a saved spot annotation
**When** the spot detail sheet is presented
**Then** `Spot.lastVisited` is updated to `Date.now` and persisted to SwiftData (FR19)

**Given** `SaveSpotSheet` uses Primary/Secondary button styles per the UX spec
**When** rendered
**Then** the Save button has `#DDB05E` background, `#1A1612` label, `cornerRadius: 12`, 16pt SF Pro Semibold
**And** the Cancel button has no fill, `#DDB05E` label
**And** no more than one primary button is visible at a time

### Story 2.3: Static Shooting Tips

As a photographer using Golden Hour,
I want to tap a saved spot and instantly see shooting tips for the current conditions,
So that I arrive at my spot prepared with specific guidance rather than guessing camera settings.

**Acceptance Criteria:**

**Given** `Resources/StaticTips.json` is created
**When** its contents are inspected
**Then** it contains entries for all 8 required condition × window combinations, keyed as `"{conditionState.rawValue}_{windowType.rawValue}"` (e.g., `"broken_clouds_golden"`, `"overcast_blue"`)
**And** each entry has `body` (conversational, present-tense tip written for someone already on location) and `cameraSettings` (ISO, aperture, shutter speed starting point) fields
**And** no tip text appears hardcoded anywhere in Swift source

**Given** `TipsService.swift` is implemented
**When** initialized
**Then** it loads `StaticTips.json` once and holds the decoded dictionary in memory — no repeated file reads

**Given** `TipsService.staticTip(for:window:) -> TipContent` is called
**When** passed any valid `ConditionState` + `WindowType` combination covered by `StaticTips.json`
**Then** it returns the correct `TipContent` for that pairing (FR25)

**Given** `TipsService.staticTip` is called with `.unknown` condition or `.neither` window type
**When** no exact match exists
**Then** it returns a sensible fallback `TipContent` — the method never returns nil or throws

**Given** `TipsServiceTests.swift` exists in `GoldenHourTests`
**When** tests run
**Then** all 8 valid condition × window combinations return the correct `TipContent` from `StaticTips.json`
**And** the `.unknown` / `.neither` fallback path is tested

**Given** `SpotDetailSheet.swift` is implemented as a bottom sheet
**When** a user taps a saved spot annotation
**Then** the sheet rises with `.presentationDetents([.medium, .large])`, opening at `.medium`
**And** the sheet surface is `#E6DAE4` at 96% opacity, `cornerRadius: 30` on top corners
**And** the drag handle is 36×4pt, `#E0C0CB`, centered, 14pt from the top edge
**And** inner padding is 24pt horizontal, 20pt vertical (UX-DR4)

**Given** `SpotDetailSheet` is open for a saved spot
**When** the sheet renders
**Then** it displays: spot name (`.title3`, SF Pro Medium), location type badge (`#DDB05E` tint, rounded label), and `TipBlock` content
**And** all content is wrapped in a `ScrollView`

**Given** `TipBlock.swift` is implemented
**When** rendered with a `TipContent`
**Then** it displays the section label "For tonight's conditions" (`.caption2` Dynamic Type, `#6B6560`, uppercase), tip body (`.body` Dynamic Type, SF Pro Text), and camera settings row (`.caption` Dynamic Type, SF Pro Mono font) — e.g., "ISO 400 · f/8 · 1/60s"
**And** the camera settings row has an `accessibilityLabel` formatted as "Starting exposure: ISO [x], f/[x], 1/[x]th of a second"

**Given** a user taps a saved spot while the app is offline
**When** `SpotDetailSheet` opens
**Then** static tips display immediately from the in-memory `TipsService` dictionary — no network required (FR22, FR28, FR29)
**And** no error state, no spinner, no message about offline status in the tips area (FR30)

**Given** static tips display immediately at tap time (FR22)
**When** `SpotDetailSheet` is presented
**Then** `TipBlock` content is visible within 200ms of the tap (NFR3) — `TipsService.staticTip` is synchronous

**Given** the current condition state is negative (e.g., `.overcast`, `.rain`)
**When** `TipBlock` renders the corresponding static tip
**Then** the tip content honestly reframes the conditions (e.g., soft light for portraits) rather than overstating photographic potential (FR26)

### Story 2.4: LLM Tip Augmentation & Full Accessibility

As a photographer using Golden Hour,
I want contextual LLM-generated tips to appear below the static tips when available, and the full app to be navigable via VoiceOver,
So that I get the most specific guidance possible for tonight's exact conditions, and the app works for all users.

**Acceptance Criteria:**

**Given** `TipsService.fetchLLMTip(conditionState:locationType:windowType:spotNote:) -> String?` is implemented
**When** called with valid inputs
**Then** it constructs an Anthropic messages API request using the canonical `Prompts.systemPrompt` and a structured user message: "Light window: {golden|blue} hour. Condition: {conditionState}. Location type: {locationType}. Spot note: {optional note}."
**And** the Anthropic API key is read from `KeychainService.read(key:)` — never from source code or hardcoded values (NFR7)

**Given** an Anthropic API response arrives within 4 seconds
**When** `TipsService.fetchLLMTip` returns a non-nil `String`
**Then** the tip is cached in `UserDefaults` under the key `UserDefaultsKey.tipsCache + "{conditionState.rawValue}_{locationType.rawValue}"` for reuse across sessions (NFR13)

**Given** the Anthropic API is unavailable or the request times out after 4 seconds
**When** `withTimeout(seconds: 4)` elapses
**Then** `fetchLLMTip` returns `nil` — no error is thrown, no error state is shown in the UI (NFR4, NFR11)

**Given** `LLMTipBlock.swift` is implemented
**When** `SpotDetailSheet` opens for a spot
**Then** `LLMTipBlock` is hidden by default — a `Task { withTimeout(4) { await tipsService.fetchLLMTip(...) } }` fires immediately
**And** if the result is non-nil, `LLMTipBlock` fades in below `TipBlock` via an opacity-only transition (no slide, no bounce)
**And** if the result is nil (timeout or failure), `LLMTipBlock` remains hidden — no spinner, no error message, no content displacement of `TipBlock` (FR23)

**Given** `LLMTipBlock` is visible with an LLM response
**When** the content is inspected
**Then** it displays: a 1pt `#E0C0CB` divider above, section label "Tonight specifically" (`.caption2` Dynamic Type, `#9B6A2E`, uppercase), and the LLM tip body (`.body` Dynamic Type, SF Pro Text) (UX-DR6)

**Given** `@Environment(\.accessibilityReduceMotion)` is true
**When** an LLM tip response arrives
**Then** `LLMTipBlock` appears instantly without the opacity fade animation

**Given** the LLM tip references the specific light window type, condition state, and location type
**When** the tip is validated manually against the `Prompts.systemPrompt` constraints
**Then** no tip is generic (applicable to any conditions); each tip references the actual inputs (FR24)

**Given** `MapView`, `SpotDetailSheet`, `TipBlock`, `LLMTipBlock`, and `SaveSpotSheet` are fully implemented
**When** a VoiceOver user navigates the app without sight
**Then** all interactive elements are reachable in a logical order
**And** `SpotAnnotation` reads its spot name and "Double tap to view tips" hint
**And** `TipBlock` camera settings read as "Starting exposure: ISO [x], f/[x], 1/[x]th of a second"
**And** when `LLMTipBlock` appears, it is reachable on the next VoiceOver swipe — no automatic announcement triggered (UX-DR15)
**And** `SaveSpotSheet` form fields have appropriate labels and the Save button's enabled/disabled state is announced

**Given** all touch targets in Epic 2 components are implemented
**When** measured
**Then** every interactive element meets the 44×44pt iOS HIG minimum (spot annotations use `.contentShape` to achieve this despite their 16pt visual size) (UX-DR15)

**Given** the app has no network connectivity and `TipsService.fetchLLMTip` is called
**When** the 4-second timeout elapses
**Then** `LLMTipBlock` remains hidden; `TipBlock` static content remains the sole tip display; the UI shows no error state, no "offline" message in the tips area (FR29, FR30, UX-DR18)

**Given** `UX-DR18` copy voice rules apply across all Epic 2 components
**When** any static string in `SpotDetailSheet`, `SaveSpotSheet`, or empty states is reviewed
**Then** no exclamation marks are used; no "Please" appears in form labels; empty state messages are direct and non-apologetic; stale/unavailable states are factual and minimal
