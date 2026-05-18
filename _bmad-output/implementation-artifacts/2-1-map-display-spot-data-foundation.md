# Story 2.1: Map Display & Spot Data Foundation

Status: done

## Story

As a photographer using Golden Hour,
I want to see a map centered on my current location with my saved shooting spots marked on it,
so that I can visually locate and navigate to my personal spots.

## Acceptance Criteria

1. **Given** `Spot.swift` is implemented as a SwiftData `@Model` class
   **When** the model is inspected
   **Then** it has properties: `name: String`, `note: String?`, `locationType: LocationType`, `latitude: Double`, `longitude: Double`, `lastVisited: Date?`, `createdAt: Date`

2. **Given** `TipContent.swift` is implemented as a `Codable` struct
   **When** decoded from `StaticTips.json`
   **Then** it correctly maps `body: String` and `cameraSettings: String` fields

3. **Given** `ModelContainer(for: Spot.self)` is configured at `GoldenHourApp` entry point
   **When** the app launches
   **Then** SwiftData is initialized before any view that reads or writes `Spot` data
   **And** no iCloud sync is configured (local persistence only) (FR18)

4. **Given** `MapView.swift` is implemented using MapKit with a Maptiler raster tile overlay (`MKTileOverlay` URL template using the Maptiler API key from `KeychainService`)
   **When** the map tab is opened
   **Then** the map renders with Maptiler tiles and OSM attribution displayed per Maptiler terms (NFR2: within 3 seconds on standard connectivity)

5. **Given** `AppState.gpsState` is `.granted`
   **When** `MapView` appears
   **Then** the map is centered on the user's current GPS coordinates (FR11)
   **And** the user can pan and zoom freely to browse the map (FR12)

6. **Given** `SpotAnnotation.swift` is implemented as a custom MapKit annotation view
   **When** a `Spot` exists in SwiftData
   **Then** it appears on the map as a 16pt filled circle with a drop shadow
   **And** its unselected state is `#8A95A8` at 70% opacity; selected state is `#DDB05E` at 100% opacity with a slight scale-up
   **And** its touch target is at least 44×44pt
   **And** its `accessibilityLabel` equals the spot's `name` and `accessibilityHint` is "Double tap to view tips"

7. **Given** SwiftData contains no `Spot` records
   **When** the user opens the Map tab
   **Then** an inline empty state message is displayed: "No spots saved yet. Tap anywhere on the map to save a location." (UX-DR13)
   **And** no modal, no full-screen takeover

8. **Given** the device is offline
   **When** map tiles are not cached
   **Then** the saved spots list remains accessible (FR28, NFR12) — tile unavailability does not prevent spot access

## Tasks / Subtasks

- [x] **Task 1: Create Spot.swift** (AC1)
  - [ ] Create `GoldenHour/Models/Spot.swift`
  - [ ] Define `@Model final class Spot` with all 7 properties per AC1
  - [ ] `createdAt` defaults to `Date.now` in the memberwise initialiser — do NOT add a `@Attribute(.unique)` constraint (not needed for v1)
  - [ ] Import both `Foundation` and `SwiftData`

- [x] **Task 2: Create TipContent.swift** (AC2)
  - [x] Create `GoldenHour/Models/TipContent.swift`
  - [x] Define `struct TipContent: Codable` with `body: String` and `cameraSettings: String`
  - [x] No `CodingKeys` needed — property names match JSON keys exactly

- [x] **Task 3: Update GoldenHourApp.swift — ModelContainer** (AC3)
  - [x] Replace `Item.self` with `Spot.self` in the `Schema([...])` array
  - [x] Delete `GoldenHour/Item.swift` — it was the Xcode default placeholder and is no longer referenced
  - [x] Verify the app builds after deletion (PBXFileSystemSynchronizedRootGroup will auto-remove it from the build)

- [x] **Task 4: Add userCoordinate to AppState** (AC5)
  - [x] Add `var userCoordinate: CLLocationCoordinate2D? { lastCalculatedCoordinate }` as a public computed property to `AppState`
  - [x] This exposes the existing `@ObservationIgnored private var lastCalculatedCoordinate` to views without making it @Observable-tracked (no unnecessary re-renders)

- [x] **Task 5: Create SpotAnnotation.swift** (AC6)
  - [x] Create `GoldenHour/Components/SpotAnnotation.swift`
  - [x] Define `class SpotAnnotation: NSObject, MKAnnotation` holding a `Spot` reference
  - [x] Define `class SpotAnnotationView: MKAnnotationView` rendering the 16pt circle in a 44×44 frame
  - [x] Unselected fill: `UIColor(named: "SlateGray")?.withAlphaComponent(0.7)`
  - [x] Selected fill: `UIColor(named: "Golden")?.withAlphaComponent(1.0)` + scale transform 1.15
  - [x] Drop shadow: `layer.shadowColor`, `shadowOpacity: 0.3`, `shadowRadius: 3`, `shadowOffset: CGSize(width: 0, height: 2)`
  - [x] `accessibilityLabel` = spot name; `accessibilityHint` = "Double tap to view tips"
  - [x] `isAccessibilityElement = true` on the annotation view

- [x] **Task 6: Implement MapView.swift** (AC4, AC5, AC6, AC7, AC8)
  - [x] Replace the placeholder `MapView` with a full SwiftUI `View` that uses `@Query` for spots
  - [x] Embed a private `MapKitView: UIViewRepresentable` wrapping `MKMapView`
  - [x] Add `MKTileOverlay` with Maptiler URL template
  - [x] Set `canReplaceMapContent = true` to suppress Apple Maps base tiles
  - [x] Implement `MKMapViewDelegate` via `Coordinator` for tile rendering, annotation registration, and selection
  - [x] Center map on `AppState.userCoordinate` on first appearance — one-time only via `hasCentred` flag
  - [x] Display OSM attribution: `Text("© MapTiler © OpenStreetMap contributors")` overlay at bottom-trailing
  - [x] Show empty state overlay when `spots.isEmpty`
  - [x] Sync `spots` from `@Query` to `MKMapView` annotations via `ObjectIdentifier` diff

## Dev Notes

### Architecture Compliance — READ FIRST

**Patterns established in Stories 1.1–1.4 (do not break):**
- `@Environment(AppState.self)` for accessing AppState in views — NOT `@EnvironmentObject`
- `Color("AssetName")` for all color references — NO hardcoded hex values in Swift source
- Named color assets: `Golden` (#DDB05E), `SlateGray` (#8A95A8), `Lavender` (#E6DAE4), `Rose` (#E0C0CB), `TextPrimary` (#1A1612), `TextSecondary` (#6B6560)
- `PBXFileSystemSynchronizedRootGroup` — new Swift files in subdirs auto-included; no pbxproj edits needed
- No `default:` in `ConditionState` switches — enumerate all 6 cases
- Errors absorbed in services — never propagate `throws` to views
- `KeychainService.shared.read(key: "maptiler_api_key")` — this is how to retrieve the Maptiler API key

**Service boundary:**
- `MapView` reads spots directly via `@Query` — no AppState intermediary needed for SwiftData queries
- `MapView` reads `AppState.userCoordinate` for centering — does NOT call SolarService or WeatherService directly
- Story 2.2 will add save/delete operations; for now `MapView` is read-only

---

### Spot Model — Exact Implementation

```swift
// GoldenHour/Models/Spot.swift
import Foundation
import SwiftData

@Model
final class Spot {
    var name:         String
    var note:         String?
    var locationType: LocationType
    var latitude:     Double
    var longitude:    Double
    var lastVisited:  Date?
    var createdAt:    Date

    init(name: String,
         note: String? = nil,
         locationType: LocationType = .other,
         latitude: Double,
         longitude: Double,
         lastVisited: Date? = nil,
         createdAt: Date = .now) {
        self.name         = name
        self.note         = note
        self.locationType = locationType
        self.latitude     = latitude
        self.longitude    = longitude
        self.lastVisited  = lastVisited
        self.createdAt    = createdAt
    }
}
```

`LocationType` is already defined in `GoldenHour/Models/LocationType.swift` from Story 1.1:
```swift
enum LocationType: String, Codable, CaseIterable {
    case coastal, urban, forest, openField = "open_field", elevated, other
}
```

`@Model` synthesizes `Codable` conformance automatically for SwiftData persistence. Do NOT add `Codable` manually.

---

### GoldenHourApp — ModelContainer Update

```swift
var sharedModelContainer: ModelContainer = {
    let schema = Schema([
        Spot.self,           // ← replace Item.self with this
    ])
    let modelConfiguration = ModelConfiguration(schema: schema, isStoredInMemoryOnly: false)
    do {
        return try ModelContainer(for: schema, configurations: [modelConfiguration])
    } catch {
        fatalError("Could not create ModelContainer: \(error)")
    }
}()
```

After replacing `Item.self`, also **delete `GoldenHour/Item.swift`** — it's the Xcode new-project placeholder (a `@Model final class Item { var timestamp: Date }`). It's referenced nowhere except the old ModelContainer. Deleting the file is safe; the build system will stop including it automatically.

---

### SpotAnnotation — UIKit Annotation Pattern

MapKit annotations require UIKit, even in a SwiftUI app. The pattern is:
1. A data class conforming to `MKAnnotation` (holds the `Spot` reference + `coordinate`)
2. An `MKAnnotationView` subclass that draws the pin

```swift
// GoldenHour/Components/SpotAnnotation.swift
import MapKit
import UIKit

// MARK: - Data Object
final class SpotAnnotation: NSObject, MKAnnotation {
    let spot: Spot
    @objc dynamic var coordinate: CLLocationCoordinate2D

    init(spot: Spot) {
        self.spot = spot
        self.coordinate = CLLocationCoordinate2D(latitude: spot.latitude,
                                                 longitude: spot.longitude)
        super.init()
    }

    var title: String? { spot.name }
}

// MARK: - View
final class SpotAnnotationView: MKAnnotationView {
    static let reuseID = "SpotAnnotationView"

    private let circleSize: CGFloat = 16
    private let hitArea:    CGFloat = 44

    override init(annotation: MKAnnotation?, reuseIdentifier: String?) {
        super.init(annotation: annotation, reuseIdentifier: reuseIdentifier)
        frame = CGRect(x: 0, y: 0, width: hitArea, height: hitArea)
        backgroundColor = .clear
        isUserInteractionEnabled = true
        isAccessibilityElement = true
        accessibilityHint = "Double tap to view tips"

        // Drop shadow on the layer
        layer.shadowColor  = UIColor.black.cgColor
        layer.shadowOpacity = 0.3
        layer.shadowRadius  = 3
        layer.shadowOffset  = CGSize(width: 0, height: 2)
    }

    required init?(coder: NSCoder) { fatalError() }

    override func draw(_ rect: CGRect) {
        guard let ctx = UIGraphicsGetCurrentContext() else { return }
        let center = CGPoint(x: rect.midX, y: rect.midY)
        let circleRect = CGRect(x: center.x - circleSize / 2,
                                y: center.y - circleSize / 2,
                                width: circleSize,
                                height: circleSize)

        let isSelected = (annotation?.isEqual(nil) == false) && super.isSelected
        let fillColor: UIColor = isSelected
            ? (UIColor(named: "Golden") ?? .systemYellow)
            : (UIColor(named: "SlateGray")?.withAlphaComponent(0.7) ?? UIColor.gray.withAlphaComponent(0.7))

        ctx.setFillColor(fillColor.cgColor)
        ctx.fillEllipse(in: circleRect)
    }

    override var isSelected: Bool {
        didSet {
            setNeedsDisplay()
            let scale: CGFloat = isSelected ? 1.15 : 1.0
            UIView.animate(withDuration: 0.15) {
                self.transform = CGAffineTransform(scaleX: scale, y: scale)
            }
        }
    }

    override var accessibilityLabel: String? {
        get { (annotation as? SpotAnnotation)?.spot.name }
        set { }
    }
}
```

**Note:** `@objc dynamic var coordinate` is required by `MKAnnotation` — SwiftData `@Model` properties are not `@objc dynamic` by default, so we copy the coordinate into the annotation object rather than reading it from the model.

---

### MapView — Architecture

`MapView.swift` should be structured as:

```swift
// GoldenHour/Views/MapView.swift
import SwiftUI
import SwiftData
import MapKit

struct MapView: View {
    @Environment(AppState.self) private var appState
    @Query private var spots: [Spot]

    var body: some View {
        ZStack {
            MapKitView(spots: spots, userCoordinate: appState.userCoordinate)
                .ignoresSafeArea()

            // OSM attribution — required by Maptiler terms
            VStack {
                Spacer()
                HStack {
                    Spacer()
                    Text("© MapTiler © OpenStreetMap contributors")
                        .font(.system(size: 9))
                        .foregroundColor(Color("TextSecondary"))
                        .padding(6)
                        .background(Color.white.opacity(0.7))
                        .cornerRadius(4)
                        .padding(.trailing, 8)
                        .padding(.bottom, 8)
                }
            }

            // Empty state
            if spots.isEmpty {
                VStack {
                    Spacer()
                    Text("No spots saved yet. Tap anywhere on the map to save a location.")
                        .font(.subheadline)
                        .foregroundColor(Color("TextSecondary"))
                        .multilineTextAlignment(.center)
                        .padding(.horizontal, 32)
                        .padding(.bottom, 40)
                }
            }
        }
    }
}
```

---

### MapKitView — UIViewRepresentable

`MapKitView` is a private `UIViewRepresentable` inside `MapView.swift`:

```swift
private struct MapKitView: UIViewRepresentable {
    let spots: [Spot]
    let userCoordinate: CLLocationCoordinate2D?

    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        mapView.delegate = context.coordinator
        mapView.showsUserLocation = true

        // Register annotation view
        mapView.register(SpotAnnotationView.self,
                         forAnnotationViewWithReuseIdentifier: SpotAnnotationView.reuseID)

        // Maptiler tile overlay
        let apiKey = KeychainService.shared.read(key: "maptiler_api_key") ?? ""
        if !apiKey.isEmpty {
            let template = "https://api.maptiler.com/maps/streets/tiles/{z}/{x}/{y}.png?key=\(apiKey)"
            let overlay = MKTileOverlay(urlTemplate: template)
            overlay.canReplaceMapContent = true
            mapView.addOverlay(overlay, level: .aboveLabels)
        }

        return mapView
    }

    func updateUIView(_ mapView: MKMapView, context: Context) {
        // Center on user location once (first time coordinate becomes available)
        if let coord = userCoordinate, !context.coordinator.hasCentred {
            let region = MKCoordinateRegion(center: coord,
                                            latitudinalMeters: 2000,
                                            longitudinalMeters: 2000)
            mapView.setRegion(region, animated: false)
            context.coordinator.hasCentred = true
        }

        // Sync SwiftData spots → MKAnnotations using a diff
        syncAnnotations(on: mapView)
    }

    private func syncAnnotations(on mapView: MKMapView) {
        let existing = mapView.annotations.compactMap { $0 as? SpotAnnotation }
        let existingIDs = Set(existing.compactMap { $0.spot.persistentModelID })
        let desiredIDs  = Set(spots.compactMap { $0.persistentModelID })

        let toRemove = existing.filter { !desiredIDs.contains($0.spot.persistentModelID) }
        let toAdd    = spots.filter { !existingIDs.contains($0.persistentModelID) }

        mapView.removeAnnotations(toRemove)
        mapView.addAnnotations(toAdd.map { SpotAnnotation(spot: $0) })
    }

    func makeCoordinator() -> Coordinator {
        Coordinator()
    }

    final class Coordinator: NSObject, MKMapViewDelegate {
        var hasCentred = false

        func mapView(_ mapView: MKMapView, rendererFor overlay: MKOverlay) -> MKOverlayRenderer {
            if let tileOverlay = overlay as? MKTileOverlay {
                return MKTileOverlayRenderer(tileOverlay: tileOverlay)
            }
            return MKOverlayRenderer(overlay: overlay)
        }

        func mapView(_ mapView: MKMapView, viewFor annotation: MKAnnotation) -> MKAnnotationView? {
            guard annotation is SpotAnnotation else { return nil }
            let view = mapView.dequeueReusableAnnotationView(
                withIdentifier: SpotAnnotationView.reuseID,
                for: annotation
            )
            return view
        }
    }
}
```

**Key points:**
- `canReplaceMapContent = true` suppresses Apple's base map — Maptiler tiles fill the entire map
- `hasCentred` flag prevents re-centering every time `updateUIView` fires (which happens on every state change)
- Annotation sync uses a diff based on `persistentModelID` to avoid removing and re-adding unchanged annotations
- The coordinator returns `nil` for the user location annotation (the default blue dot is fine)

---

### Maptiler URL Template

The exact URL format for `MKTileOverlay`:
```
https://api.maptiler.com/maps/streets/tiles/{z}/{x}/{y}.png?key=YOUR_API_KEY
```

`MKTileOverlay` substitutes `{z}`, `{x}`, `{y}` automatically. No manual substitution needed.

**Style options** (all produce raster PNGs compatible with MKTileOverlay):
- `streets` — street map (recommended default)
- `outdoor` — topographic, good for nature/hiking spots
- `satellite` — aerial imagery

Use `streets` as default. The style can be changed by swapping the path segment.

**OSM Attribution requirement (Maptiler terms):**
Display "© MapTiler © OpenStreetMap contributors" on the map at all times. This is a legal requirement for the free tier, not optional.

---

### AppState — userCoordinate Addition

Add to `GoldenHour/AppState.swift`:

```swift
/// Exposes the last GPS-resolved coordinate for views that need it (e.g., MapView centering).
/// Read-only; not @Observable-tracked (reads from @ObservationIgnored storage).
var userCoordinate: CLLocationCoordinate2D? {
    lastCalculatedCoordinate
}
```

This is a minimal, safe change — it does not make `lastCalculatedCoordinate` tracked, so map centering does not trigger home screen re-renders.

---

### SwiftData Query in MapView

`@Query` in `MapView` fetches all spots sorted by creation date:

```swift
@Query(sort: \Spot.createdAt) private var spots: [Spot]
```

This works because `MapView` is embedded in the scene that has `.modelContainer(sharedModelContainer)` applied (in `GoldenHourApp.body`). No additional setup needed.

**Note:** `@Query` is a SwiftUI property wrapper, not available in `UIViewRepresentable`. This is why `MapKitView` receives `spots: [Spot]` as a plain `let` parameter — the SwiftUI wrapper does the querying.

---

### MapView + ContentView Integration

`ContentView.swift` already routes `selectedTab == 1` (default branch) to `MapView()` via:
```swift
default: MapView()
```

No changes to `ContentView.swift` are needed for this story.

The `MapView` SwiftUI view will have access to `AppState` via `.environment(appState)` which is injected at the `WindowGroup` level in `GoldenHourApp`.

---

### Offline Behavior (AC8)

When the device is offline:
- `MKTileOverlay` will fail to fetch tiles — the map shows a blank/gray background
- `@Query` spots are read from local SwiftData — fully available offline
- `SpotAnnotation` objects still render over the blank tile background
- Empty state overlay appears/disappears based on `spots.isEmpty` regardless of connectivity

No special offline detection code is needed. The architecture's "errors absorbed in services" principle applies here — tile fetch failures are silently handled by MapKit.

---

### Files to Create vs Modify

**NEW:**
- `GoldenHour/Models/Spot.swift`
- `GoldenHour/Models/TipContent.swift`
- `GoldenHour/Components/SpotAnnotation.swift`

**MODIFY:**
- `GoldenHour/Views/MapView.swift` — replace placeholder with full implementation
- `GoldenHour/AppState.swift` — add `userCoordinate` computed property
- `GoldenHour/GoldenHourApp.swift` — replace `Item.self` with `Spot.self`

**DELETE:**
- `GoldenHour/Item.swift` — Xcode placeholder, no longer in schema

No `project.pbxproj` edits needed — new files in correct directories are auto-included.

---

### Testing Notes

Architecture.md: "No UI tests for v1." No new test files are required for this story.

The `TipContent` struct is so simple (two String properties, no custom coding) that a test would add no value. The SwiftData `Spot` model is tested implicitly by the app running without crashing after the ModelContainer migration.

---

### Previous Story Context (Stories 1.1–1.4)

From Stories 1.1–1.4 — established patterns:
- `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` — all types implicitly `@MainActor` unless `nonisolated`
- UIKit views used via `UIViewRepresentable` — pattern established (implicitly, since MapKit requires UIKit)
- `KeychainService.shared.read(key: "maptiler_api_key")` — the key for Maptiler; `"anthropic_api_key"` for Anthropic
- `ContentView.swift` uses `selectedTab == 0 → HomeView`, `default → MapView`
- `MapView.swift` currently a placeholder (`Text("Map")` on Lavender background) — replace entirely

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

All SourceKit cross-file reference warnings are expected in this project (same pattern as Stories 1.1–1.4). All files compile correctly within the Xcode project.

Annotation sync: used `ObjectIdentifier($0.spot)` rather than `persistentModelID` because SwiftData's `@Model` instances in the same `ModelContext` return the same in-memory object for the same persistent record, making `ObjectIdentifier` a reliable diff key.

### Completion Notes List

All 6 tasks complete. All 8 acceptance criteria satisfied:

- AC1: `Spot.swift` — `@Model final class` with all 7 required properties and correct types. `createdAt` defaults to `.now`.
- AC2: `TipContent.swift` — `Codable` struct with `body` and `cameraSettings`. Property names match JSON keys; no custom CodingKeys needed.
- AC3: `GoldenHourApp.swift` updated — `Spot.self` in schema; `Item.swift` deleted.
- AC4: `MapKitView` adds `MKTileOverlay` with Maptiler URL template; `canReplaceMapContent = true` suppresses Apple base tiles; OSM attribution `Text` overlay at bottom-trailing.
- AC5: `hasCentred` flag in Coordinator ensures one-time centering on first non-nil `userCoordinate`; user can freely pan/zoom after that.
- AC6: `SpotAnnotationView` — 16pt `circleView` subview in 44pt hit-area frame; SlateGray 70% unselected; Golden 100% + 1.15× scale when selected; `accessibilityLabel` = spot name, `accessibilityHint` = "Double tap to view tips".
- AC7: Empty state `Text` overlay shown when `spots.isEmpty`; inline at bottom of map — no modal, no full-screen.
- AC8: `@Query` reads directly from local SwiftData; tile network failures silently handled by MapKit; spots remain visible.

No tests added — architecture.md explicitly states "No UI tests for v1."

### File List

**New files:**
- `GoldenHour/GoldenHour/Models/Spot.swift`
- `GoldenHour/GoldenHour/Models/TipContent.swift`
- `GoldenHour/GoldenHour/Components/SpotAnnotation.swift`

**Modified files:**
- `GoldenHour/GoldenHour/Views/MapView.swift` — replaced placeholder with full UIViewRepresentable + MKMapView implementation
- `GoldenHour/GoldenHour/AppState.swift` — added `userCoordinate` computed property
- `GoldenHour/GoldenHour/GoldenHourApp.swift` — replaced `Item.self` with `Spot.self` in ModelContainer schema

**Deleted files:**
- `GoldenHour/GoldenHour/Item.swift` — Xcode default placeholder, no longer referenced

## Change Log

| Date | Change | Author |
|---|---|---|
| 2026-05-17 | Story created, status: ready-for-dev | bmad-create-story |
| 2026-05-17 | All 6 tasks implemented; status: review | bmad-dev-story |
