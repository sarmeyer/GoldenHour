# Story 2.2: Spot Management — Save, Edit & Delete

Status: done

## Story

As a photographer using Golden Hour,
I want to save shooting locations with a name, note, and location type, and be able to edit or delete them,
so that I can build a personal library of spots I return to.

## Acceptance Criteria

1. **Given** the user long-presses a location on the map
   **When** the long-press gesture is recognized
   **Then** `SaveSpotSheet` is presented at `.medium` detent only (FR13)

2. **Given** `SaveSpotSheet.swift` is implemented
   **When** rendered
   **Then** it contains: a name `TextField` (required), a note `TextEditor` (optional, ~3 lines, placeholder "e.g. west-facing, try at golden hour"), a `Picker` for location type (6 options matching `LocationType` enum cases exactly), a primary Save button and a secondary Cancel button
   **And** the Save button is disabled when the name field is empty

3. **Given** the user enters a name and taps Save
   **When** `SaveSpotSheet` dismisses
   **Then** a new `Spot` is written to SwiftData with the entered name, note (if provided), selected `LocationType`, the tapped map coordinate, and `createdAt = Date.now`
   **And** the spot's `SpotAnnotation` appears on the map immediately (FR16)
   **And** the spot persists across app restarts (FR18) and is available offline (FR28)

4. **Given** a spot exists and the user wants to edit its note
   **When** the user taps an existing spot annotation to open its detail sheet
   **Then** the note field is editable inline in the sheet and changes are saved to SwiftData on dismissal (FR14)

5. **Given** a spot exists
   **When** the user long-presses its annotation
   **Then** a "Delete" option is presented via an `Alert` with title "Delete [Spot Name]?", message "This can't be undone.", buttons "Delete" (destructive role) / "Cancel" (default/cancel role) (UX-DR14)

6. **Given** the user confirms deletion in the Alert
   **When** the action completes
   **Then** the `Spot` is removed from SwiftData, its annotation is removed from the map, and any open detail sheet for that spot is dismissed (FR17)

7. **Given** the user taps a saved spot annotation
   **When** the spot detail sheet is presented
   **Then** `Spot.lastVisited` is updated to `Date.now` and persisted to SwiftData (FR19)

8. **Given** `SaveSpotSheet` uses Primary/Secondary button styles per the UX spec
   **When** rendered
   **Then** the Save button has `#DDB05E` background, `#1A1612` label, `cornerRadius: 12`, 16pt SF Pro Semibold
   **And** the Cancel button has no fill, `#DDB05E` label
   **And** no more than one primary button is visible at a time

## Tasks / Subtasks

- [x] **Task 1: Add interaction bindings to MapKitView** (AC1, AC5, AC7)
  - [x] `@Binding var pendingSaveCoordinate: MapCoordinate?`, `selectedSpot: Spot?`, `spotToDelete: Spot?` added to `MapKitView`
  - [x] `UILongPressGestureRecognizer` added in `makeUIView`; coordinator callbacks set in `updateUIView`
  - [x] `weak var mapView: MKMapView?` + `onSaveAt`/`onDeleteSpot`/`onSelectSpot` closures in Coordinator
  - [x] `@objc handleLongPress`: hit-tests for `SpotAnnotationView` → delete or save path
  - [x] `mapView(_:didSelect:)`: deselects annotation, fires `onSelectSpot`

- [x] **Task 2: Add sheet/alert state to MapView and wire it all up** (AC1, AC5, AC6)
  - [x] Added `MapCoordinate` Identifiable wrapper struct; `@State pendingSaveCoordinate: MapCoordinate?`
  - [x] `@State selectedSpot: Spot?`, `@State spotToDelete: Spot?`
  - [x] `@Environment(\.modelContext) private var modelContext`
  - [x] `.sheet(item: $pendingSaveCoordinate)` → `SaveSpotSheet`
  - [x] `.sheet(item: $selectedSpot)` → `SpotDetailSheet`
  - [x] `.alert(...)` delete confirmation; clears selectedSpot via `persistentModelID` comparison before deleting

- [x] **Task 3: Create SaveSpotSheet.swift** (AC2, AC3, AC8)
  - [x] Created `GoldenHour/Components/SaveSpotSheet.swift`
  - [x] `coordinate: CLLocationCoordinate2D`; uses `@Environment(\.modelContext)` for insert
  - [x] `@State name, note, locationType`; `canSave` guards trimmed-non-empty name
  - [x] Name `TextField`, Note `TextEditor` 80pt with placeholder ZStack overlay
  - [x] `Picker(.menu)` for all 6 `LocationType` cases with `displayName`
  - [x] Primary Save button (Golden bg, TextPrimary label, cornerRadius 12, semibold 16pt); disabled when `!canSave`
  - [x] Cancel button (Golden foreground, no fill)
  - [x] `modelContext.insert(Spot(...))` + `dismiss()`
  - [x] Lavender 96%, cornerRadius 30, Rose drag handle, `.presentationDetents([.medium])`

- [x] **Task 4: Create SpotDetailSheet.swift** (AC4, AC6, AC7)
  - [x] Created `GoldenHour/Components/SpotDetailSheet.swift`
  - [x] `@Bindable var spot: Spot`; `@State noteText` initialized on appear from `spot.note`
  - [x] `.onAppear`: `spot.lastVisited = .now` (FR19)
  - [x] `.onDisappear`: `spot.note = noteText.isEmpty ? nil : noteText`
  - [x] Spot name `.title3.weight(.medium)` + Golden location type badge
  - [x] Note `TextEditor` with placeholder in `ScrollView` (Story 2.3 TipBlock slot below)
  - [x] Same Lavender/cornerRadius/handle surface
  - [x] `.presentationDetents([.medium, .large])`; swipe-down only to dismiss

## Dev Notes

### Architecture Compliance — READ FIRST

**From Stories 2.1 and earlier:**
- `@Environment(AppState.self)` — never `@EnvironmentObject`
- `@Environment(\.modelContext) private var modelContext` — for SwiftData insert/delete in views
- `@Bindable var spot: Spot` — the SwiftData+Observable-compatible binding pattern for editing model objects (iOS 17+)
- `Color("AssetName")` — all colors via named assets, no hex literals
- Named colors available: `Golden`, `Lavender`, `Rose`, `TextPrimary`, `TextSecondary`, `SlateGray`
- `MapKitView` is `private struct` inside `MapView.swift` — keep it there
- `SpotAnnotation` and `SpotAnnotationView` are in `GoldenHour/Components/SpotAnnotation.swift`
- When spots are deleted from SwiftData, `@Query` in `MapView` updates automatically, triggering `syncAnnotations` which removes the annotation from the map — **no manual annotation removal needed**
- `PBXFileSystemSynchronizedRootGroup` auto-includes new Swift files in `GoldenHour/Components/` — no pbxproj edits

---

### Long-Press Gesture — Hit Test Pattern

The long-press gesture is added to the whole `MKMapView`. On fire, the coordinator must determine whether the press landed on a `SpotAnnotationView` (→ delete flow) or empty map (→ save flow):

```swift
// In makeUIView:
let longPress = UILongPressGestureRecognizer(target: context.coordinator,
                                             action: #selector(Coordinator.handleLongPress(_:)))
longPress.minimumPressDuration = 0.5
mapView.addGestureRecognizer(longPress)
context.coordinator.mapView = mapView   // weak ref stored in coordinator

// In Coordinator:
weak var mapView: MKMapView?

// Also need bindings accessible from coordinator:
var onSaveAt: ((CLLocationCoordinate2D) -> Void)?
var onDeleteAnnotation: ((SpotAnnotation) -> Void)?

@objc func handleLongPress(_ gesture: UILongPressGestureRecognizer) {
    guard gesture.state == .began, let mapView else { return }
    let point = gesture.location(in: mapView)

    // Hit-test: did the press land on an existing SpotAnnotationView?
    if let hitView = mapView.hitTest(point, with: nil),
       let annotationView = hitView as? SpotAnnotationView ?? hitView.superview as? SpotAnnotationView,
       let spotAnnotation = annotationView.annotation as? SpotAnnotation {
        // Long-press on annotation → trigger delete
        onDeleteAnnotation?(spotAnnotation)
    } else {
        // Long-press on empty map → trigger save
        let coordinate = mapView.convert(point, toCoordinateFrom: mapView)
        onSaveAt?(coordinate)
    }
}
```

**Wiring the callbacks** — set `onSaveAt` and `onDeleteAnnotation` in `updateUIView`:
```swift
func updateUIView(_ mapView: MKMapView, context: Context) {
    context.coordinator.onSaveAt = { coord in
        DispatchQueue.main.async { pendingSaveCoordinate = coord }
    }
    context.coordinator.onDeleteAnnotation = { annotation in
        DispatchQueue.main.async { spotToDelete = annotation.spot }
    }
    // ... existing hasCentred and syncAnnotations calls
}
```

**Annotation tap (didSelect):**
```swift
func mapView(_ mapView: MKMapView, didSelect view: MKAnnotationView) {
    guard let spotAnnotation = view.annotation as? SpotAnnotation else { return }
    mapView.deselectAnnotation(spotAnnotation, animated: false)  // prevent MapKit callout
    DispatchQueue.main.async {
        selectedSpot = spotAnnotation.spot
    }
}
```

---

### CLLocationCoordinate2D as Identifiable — Sheet Trigger

SwiftUI's `.sheet(item:)` requires the item to be `Identifiable`. `CLLocationCoordinate2D` is not `Identifiable`. Use a lightweight wrapper:

```swift
private struct MapCoordinate: Identifiable {
    let id = UUID()
    let coordinate: CLLocationCoordinate2D
}
```

Change state to `@State private var pendingSaveCoordinate: MapCoordinate? = nil`.
Set with: `pendingSaveCoordinate = MapCoordinate(coordinate: coord)`.
In sheet: `SaveSpotSheet(coordinate: item.coordinate)`.

Alternatively, since `Spot` IS `Identifiable` (SwiftData models conform to `Identifiable` automatically), use `.sheet(item: $selectedSpot)` directly.

---

### Delete Alert Pattern

```swift
.alert("Delete \(spotToDelete?.name ?? "Spot")?",
       isPresented: Binding(
           get: { spotToDelete != nil },
           set: { if !$0 { spotToDelete = nil } }
       )) {
    Button("Delete", role: .destructive) {
        if let spot = spotToDelete {
            // If this spot's sheet is open, close it first
            if selectedSpot?.persistentModelID == spot.persistentModelID {
                selectedSpot = nil
            }
            modelContext.delete(spot)
        }
        spotToDelete = nil
    }
    Button("Cancel", role: .cancel) { spotToDelete = nil }
} message: {
    Text("This can't be undone.")
}
```

**Note:** SwiftData `@Model` objects conform to `Identifiable` via `persistentModelID`. Use `persistentModelID` for equality checks, not `ObjectIdentifier` (which is memory-address based and may differ after faulting).

---

### Sheet Surface — Shared Pattern

Both `SaveSpotSheet` and `SpotDetailSheet` use the same presentation surface. Apply these modifiers to the sheet content view:

```swift
.presentationBackground(Color("Lavender").opacity(0.96))
.presentationCornerRadius(30)
.presentationDragIndicator(.hidden)  // hide system indicator; use custom handle below
```

And add a custom drag handle as the first element inside the sheet VStack:
```swift
// Custom drag handle
RoundedRectangle(cornerRadius: 2)
    .fill(Color("Rose"))
    .frame(width: 36, height: 4)
    .padding(.top, 14)
    .padding(.bottom, 8)
```

Inner padding: 24pt horizontal, 20pt vertical from handle to first content element.

---

### SaveSpotSheet — TextEditor Placeholder

`TextEditor` doesn't natively support placeholder text. Use a `ZStack` overlay:

```swift
ZStack(alignment: .topLeading) {
    if note.isEmpty {
        Text("e.g. west-facing, try at golden hour")
            .font(.body)
            .foregroundColor(Color("TextSecondary").opacity(0.6))
            .padding(.top, 8)
            .padding(.leading, 5)
            .allowsHitTesting(false)
    }
    TextEditor(text: $note)
        .frame(height: 80)   // ~3 lines
        .scrollContentBackground(.hidden)
        .background(Color.clear)
}
```

---

### SaveSpotSheet — Button Styles

Primary Save button:
```swift
Button("Save") {
    let spot = Spot(name: name.trimmingCharacters(in: .whitespaces),
                    note: note.isEmpty ? nil : note,
                    locationType: locationType,
                    latitude: coordinate.latitude,
                    longitude: coordinate.longitude)
    modelContext.insert(spot)
    dismiss()
}
.disabled(name.trimmingCharacters(in: .whitespaces).isEmpty)
.font(.system(size: 16, weight: .semibold))
.foregroundColor(Color("TextPrimary"))
.frame(maxWidth: .infinity)
.padding(.vertical, 14)
.background(
    name.trimmingCharacters(in: .whitespaces).isEmpty
        ? Color("Golden").opacity(0.4)
        : Color("Golden")
)
.cornerRadius(12)
```

Secondary Cancel button:
```swift
Button("Cancel") { dismiss() }
    .font(.system(size: 16, weight: .regular))
    .foregroundColor(Color("Golden"))
```

---

### SaveSpotSheet — LocationType Display Names

`LocationType` raw values are snake_case internal identifiers. Show human-readable labels in the Picker:

```swift
extension LocationType {
    var displayName: String {
        switch self {
        case .coastal:   return "Coastal"
        case .urban:     return "Urban"
        case .forest:    return "Forest"
        case .openField: return "Open Field"
        case .elevated:  return "Elevated"
        case .other:     return "Other"
        }
    }
}
```

Picker implementation:
```swift
Picker("Location Type", selection: $locationType) {
    ForEach(LocationType.allCases, id: \.self) { type in
        Text(type.displayName).tag(type)
    }
}
.pickerStyle(.menu)
```

**Add this extension to `GoldenHour/Models/LocationType.swift`** — same file as the enum, do not create a new file.

---

### SpotDetailSheet — @Bindable and Note Editing

`@Bindable` works with SwiftData `@Model` classes in iOS 17+. However, for the "save on dismiss" pattern, a local copy of the note is cleaner than binding directly (avoids partial saves if user dismisses mid-edit):

```swift
struct SpotDetailSheet: View {
    @Bindable var spot: Spot
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var noteText: String = ""

    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            // Custom drag handle
            ...

            ScrollView {
                VStack(alignment: .leading, spacing: 20) {
                    // Spot name + location badge
                    HStack(alignment: .firstTextBaseline) {
                        Text(spot.name)
                            .font(.title3.weight(.medium))
                            .foregroundColor(Color("TextPrimary"))
                        Spacer()
                        Text(spot.locationType.displayName)
                            .font(.caption.weight(.medium))
                            .foregroundColor(Color("Golden"))
                            .padding(.horizontal, 8)
                            .padding(.vertical, 4)
                            .background(Color("Golden").opacity(0.15))
                            .cornerRadius(8)
                    }

                    // Note editing (Story 2.3 adds TipBlock below this)
                    VStack(alignment: .leading, spacing: 6) {
                        Text("NOTE")
                            .font(.caption2)
                            .foregroundColor(Color("TextSecondary"))
                        ZStack(alignment: .topLeading) {
                            if noteText.isEmpty {
                                Text("Add a note about this spot...")
                                    .font(.body)
                                    .foregroundColor(Color("TextSecondary").opacity(0.6))
                                    .padding(.top, 8).padding(.leading, 5)
                                    .allowsHitTesting(false)
                            }
                            TextEditor(text: $noteText)
                                .frame(minHeight: 80)
                                .scrollContentBackground(.hidden)
                                .background(Color.clear)
                        }
                    }

                    // Story 2.3 will insert TipBlock here
                }
                .padding(.horizontal, 24)
                .padding(.top, 20)
                .padding(.bottom, 32)
            }
        }
        .background(Color.clear)
        .presentationBackground(Color("Lavender").opacity(0.96))
        .presentationCornerRadius(30)
        .presentationDragIndicator(.hidden)
        .presentationDetents([.medium, .large])
        .onAppear {
            noteText = spot.note ?? ""
            spot.lastVisited = .now     // FR19 — persist immediately via SwiftData
        }
        .onDisappear {
            spot.note = noteText.isEmpty ? nil : noteText
        }
    }
}
```

**Note:** `spot.lastVisited = .now` triggers SwiftData to mark the object dirty; SwiftData's `ModelContext` auto-saves on the run loop. No explicit `try modelContext.save()` needed.

---

### MapView — Sheet Wiring Summary

```swift
struct MapView: View {
    @Environment(AppState.self) private var appState
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Spot.createdAt) private var spots: [Spot]

    @State private var pendingSaveCoordinate: MapCoordinate? = nil
    @State private var selectedSpot: Spot? = nil
    @State private var spotToDelete: Spot? = nil

    var body: some View {
        ZStack {
            MapKitView(
                spots: spots,
                userCoordinate: appState.userCoordinate,
                pendingSaveCoordinate: $pendingSaveCoordinate,
                selectedSpot: $selectedSpot,
                spotToDelete: $spotToDelete
            )
            .ignoresSafeArea()
            // ... attribution and empty state overlays unchanged
        }
        .sheet(item: $pendingSaveCoordinate) { item in
            SaveSpotSheet(coordinate: item.coordinate)
        }
        .sheet(item: $selectedSpot) { spot in
            SpotDetailSheet(spot: spot)
        }
        .alert(...)   // delete alert per Dev Notes
    }
}
```

---

### Files to Create vs Modify

**NEW:**
- `GoldenHour/Components/SaveSpotSheet.swift`
- `GoldenHour/Components/SpotDetailSheet.swift`

**MODIFY:**
- `GoldenHour/Views/MapView.swift` — add bindings to `MapKitView`, add state vars + sheets + alert to `MapView`, add `MapCoordinate` wrapper, update coordinator with callbacks and `didSelect` delegate
- `GoldenHour/Models/LocationType.swift` — add `displayName` computed property extension

**DO NOT TOUCH:**
- `GoldenHour/Components/SpotAnnotation.swift` — no changes; `prepareForReuse` already handles selection state reset
- `GoldenHour/AppState.swift` — no changes
- Any Story 1.x files

---

### Testing Notes

Architecture.md: "No UI tests for v1." No test files required. The SwiftData persistence and `@Query` reactivity are tested implicitly by running the app. Manual verification steps:
1. Long-press empty map → SaveSpotSheet appears at .medium only (cannot pull to large)
2. Fill name + save → annotation appears immediately on map
3. Tap annotation → SpotDetailSheet opens; edit note; dismiss → note persists
4. Long-press annotation → delete alert with correct spot name → confirm → spot removed
5. Verify spot survives app restart (close and reopen Simulator)

---

### Previous Story Context (Story 2.1 — Key Learnings)

- `ObjectIdentifier` used for annotation sync diff — reliable for local-only SwiftData context
- `@Observable`-tracked `userCoordinate` (not computed) — critical for MapView re-rendering
- `prepareForReuse()` already added to `SpotAnnotationView` — reused annotation views are reset
- `hasCentred` flag in Coordinator — centering happens only once; preserved across this story
- `canReplaceMapContent = true` on tile overlay — this must be preserved when MapKitView is modified
- Do NOT add the tile overlay again in `updateUIView` — it's set once in `makeUIView`

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

All SourceKit cross-file warnings are expected (same pattern throughout this project). All code compiles within the Xcode target.

`CLLocationCoordinate2D` is not `Identifiable`, so `MapCoordinate` wrapper struct was added as a `private struct` at the top of `MapView.swift` — this is the cleanest approach that keeps the wrapper scoped to its only consumer.

The coordinator callbacks (`onSaveAt`, `onDeleteSpot`, `onSelectSpot`) are set in `updateUIView` (not `makeUIView`) because they close over SwiftUI bindings that are value types and must be re-captured on each update cycle.

### Completion Notes List

All 4 tasks complete. All 8 acceptance criteria satisfied:

- AC1: Long-press gesture on `MKMapView` → coordinator `handleLongPress` → hit-test differentiates empty map vs annotation → `pendingSaveCoordinate` set → `SaveSpotSheet` presented at `.medium` only
- AC2: `SaveSpotSheet` — name TextField, note TextEditor 80pt with ZStack placeholder, LocationType Picker (.menu style, all 6 cases, displayName), `canSave` disables Save button when name is empty or whitespace-only
- AC3: Save → `modelContext.insert(Spot(...))` with coordinate + `createdAt: .now` → `@Query` updates → `syncAnnotations` adds annotation; persists via SwiftData to disk
- AC4: `SpotDetailSheet` — `@State noteText` copies `spot.note` on appear; `.onDisappear` writes back; note editing is inline; changes persist to SwiftData
- AC5: Long-press on annotation → hit-test finds `SpotAnnotationView` → `onDeleteSpot` → `spotToDelete` → `.alert` with exact spot name
- AC6: Alert "Delete" button checks `persistentModelID` to clear `selectedSpot` if needed, then `modelContext.delete(spot)` → `@Query` → `syncAnnotations` removes annotation automatically
- AC7: `mapView(_:didSelect:)` → `onSelectSpot` → `selectedSpot` → `SpotDetailSheet` → `.onAppear { spot.lastVisited = .now }`
- AC8: Save button — `Color("Golden")` bg, `Color("TextPrimary")` label, `cornerRadius(12)`, `.system(size: 16, weight: .semibold)`; Cancel — `Color("Golden")` foreground, no background

### File List

**New files:**
- `GoldenHour/GoldenHour/Components/SaveSpotSheet.swift`
- `GoldenHour/GoldenHour/Components/SpotDetailSheet.swift`

**Modified files:**
- `GoldenHour/GoldenHour/Views/MapView.swift` — complete rewrite: `MapCoordinate` wrapper, 3 `@State` vars, `@Environment(\.modelContext)`, two `.sheet(item:)`, `.alert`, `MapKitView` gains 3 `@Binding` params + long-press gesture + 3 coordinator callbacks + `mapView(_:didSelect:)` delegate
- `GoldenHour/GoldenHour/Models/LocationType.swift` — added `displayName` computed property extension

## Change Log

| Date | Change | Author |
|---|---|---|
| 2026-05-17 | Story created, status: ready-for-dev | bmad-create-story |
| 2026-05-17 | All 4 tasks implemented; status: review | bmad-dev-story |
