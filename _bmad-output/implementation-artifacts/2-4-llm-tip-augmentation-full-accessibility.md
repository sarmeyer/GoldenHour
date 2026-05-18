# Story 2.4: LLM Tip Augmentation & Full Accessibility

Status: done

## Story

As a photographer using Golden Hour,
I want contextual LLM-generated tips to appear below the static tips when available, and the full app to be navigable via VoiceOver,
so that I get the most specific guidance possible for tonight's exact conditions, and the app works for all users.

## Acceptance Criteria

1. **Given** `TipsService.fetchLLMTip(conditionState:locationType:windowType:spotNote:) -> String?` is implemented
   **When** called with valid inputs
   **Then** it constructs an Anthropic messages API request using the canonical `Prompts.systemPrompt` and a structured user message: "Light window: {golden|blue} hour. Condition: {conditionState}. Location type: {locationType}. Spot note: {optional note}."
   **And** the Anthropic API key is read from `KeychainService.read(key: "anthropic_api_key")` — never from source code or hardcoded values (NFR7)

2. **Given** an Anthropic API response arrives within 4 seconds
   **When** `TipsService.fetchLLMTip` returns a non-nil `String`
   **Then** the tip is cached in `UserDefaults` under the key `UserDefaultsKey.tipsCache + "{conditionState.rawValue}_{locationType.rawValue}"` for reuse across sessions (NFR13)

3. **Given** the Anthropic API is unavailable or the request times out after 4 seconds
   **When** `withTimeout(seconds: 4)` elapses in the caller
   **Then** the overall result is `nil` — no error is thrown, no error state is shown in the UI (NFR4, NFR11)

4. **Given** `LLMTipBlock.swift` is implemented
   **When** `SpotDetailSheet` opens for a spot
   **Then** `LLMTipBlock` is hidden by default — a `.task {}` fires `withTimeout(seconds: 4) { await TipsService.shared.fetchLLMTip(...) }` immediately
   **And** if the result is non-nil, `LLMTipBlock` fades in below `TipBlock` via an opacity-only transition (no slide, no bounce)
   **And** if the result is nil (timeout or failure), `LLMTipBlock` remains hidden — no spinner, no error message, no content displacement of `TipBlock` (FR23)

5. **Given** `LLMTipBlock` is visible with an LLM response
   **When** the content is inspected
   **Then** it displays: a 1pt `#E0C0CB` (Rose) divider above, section label "Tonight specifically" (`.caption2` Dynamic Type, `#9B6A2E`, uppercase), and the LLM tip body (`.body` Dynamic Type, SF Pro Text) (UX-DR6)

6. **Given** `@Environment(\.accessibilityReduceMotion)` is true
   **When** an LLM tip response arrives
   **Then** `LLMTipBlock` appears instantly without the opacity fade animation

7. **Given** `MapView`, `SpotDetailSheet`, `TipBlock`, `LLMTipBlock`, and `SaveSpotSheet` are fully implemented
   **When** a VoiceOver user navigates the app without sight
   **Then** all interactive elements are reachable in a logical order
   **And** `SpotAnnotation` reads its spot name and "Double tap to view tips" hint *(already implemented in Story 2.1)*
   **And** `TipBlock` camera settings read as "Starting exposure: ISO [x], f/[x], 1/[x]th of a second" *(already implemented in Story 2.3)*
   **And** when `LLMTipBlock` appears, it is reachable on the next VoiceOver swipe — no automatic announcement triggered (UX-DR15)
   **And** `SaveSpotSheet` form fields have appropriate accessibility labels and the Save button's enabled/disabled state is announced

8. **Given** all touch targets in Epic 2 components are implemented
   **When** measured
   **Then** every interactive element meets the 44×44pt iOS HIG minimum

9. **Given** the app has no network connectivity and `TipsService.fetchLLMTip` is called
   **When** the 4-second timeout elapses
   **Then** `LLMTipBlock` remains hidden; `TipBlock` static content remains the sole tip display; the UI shows no error state (FR29, FR30)

10. **Given** `UX-DR18` copy voice rules apply across all Epic 2 components
    **When** any static string in `SpotDetailSheet`, `SaveSpotSheet`, or empty states is reviewed
    **Then** no exclamation marks are used; no "Please" in form labels; empty state messages are direct and non-apologetic

## Tasks / Subtasks

- [x] **Task 1: Add GoldenDeep color asset** (AC5)
  - [x] Created `Assets.xcassets/Colors/GoldenDeep.colorset/Contents.json` with `#9B6A2E` (R:0.608, G:0.416, B:0.180)

- [x] **Task 2: Add `fetchLLMTip` to TipsService.swift** (AC1, AC2, AC3)
  - [x] Added `func fetchLLMTip(conditionState:locationType:windowType:spotNote:) async -> String?`
  - [x] Cache check → Keychain API key guard → build user message → POST to Anthropic → cache on success → return nil on any error
  - [x] Added private `AnthropicResponse: Decodable` struct with `ContentBlock`
  - [x] `withTimeout` not inside `fetchLLMTip` — applied by caller

- [x] **Task 3: Create LLMTipBlock.swift** (AC4, AC5, AC6, AC7)
  - [x] Created `GoldenHour/Components/LLMTipBlock.swift`
  - [x] Opacity-based visibility (not if/else) via `accessibilityHidden(!isVisible)` — no VoiceOver announcement on appear
  - [x] `reduceMotion` gates `.animation` — instant appearance when true
  - [x] 1pt Rose divider, "Tonight specifically" GoldenDeep uppercase caption2, body text

- [x] **Task 4: Update SpotDetailSheet.swift** (AC4, AC6, AC7)
  - [x] Added `@State llmTipText: String?` and `@State llmTipVisible: Bool`
  - [x] Kept `.onAppear {}` for sync setup; added `.task {}` for async LLM fetch
  - [x] `withTimeout(4)` wraps `fetchLLMTip`; `raw.flatMap { $0 }` collapses `String??` → `String?`
  - [x] `LLMTipBlock(tipText: llmTipText ?? "", isVisible: llmTipVisible)` added after TipBlock
  - [x] All Story 2.3 code preserved

- [x] **Task 5: Accessibility pass** (AC7, AC8, AC10)
  - [x] `SaveSpotSheet` TextField: `.accessibilityLabel("Spot name, required")`
  - [x] `SaveSpotSheet` Cancel: `.frame(maxWidth: .infinity, minHeight: 44)`
  - [x] LLMTipBlock: `accessibilityHidden(!isVisible)` prevents VoiceOver access when hidden
  - [x] UX-DR18 review: all strings compliant — no `!`, no "Please", all direct

## Dev Notes

### Architecture Compliance — READ FIRST

**From Stories 2.1–2.3:**
- `@Environment(AppState.self)` — never `@EnvironmentObject`
- `Color("AssetName")` — all colors via named assets, no hex literals in Swift
- `TipsService` is `final class` with `static let shared` — add methods, never create a new instance
- `KeychainService.shared.read(key: "anthropic_api_key")` — the Keychain key used since Story 1.1
- `UserDefaultsKey.tipsCache` = `"tipsCache_"` (with trailing underscore) — append `"{state}_{type}"` to form the full key
- `withTimeout(seconds:operation:)` is a global free function in `GoldenHour/Utilities/WithTimeout.swift`
- `Prompts.systemPrompt` is a static let in `GoldenHour/Utilities/Prompts.swift`
- No `default:` in `ConditionState` switches — already handled in Story 2.3's `fallbackTip`
- `PBXFileSystemSynchronizedRootGroup` — new Swift files auto-included; no pbxproj edits

**Error absorption (architecture.md):** All network errors absorbed in `fetchLLMTip` — return nil, never throw.

---

### Anthropic Messages API — Exact Implementation

```swift
// In TipsService.swift:

// Private response shape
private struct AnthropicResponse: Decodable {
    struct ContentBlock: Decodable {
        let type: String
        let text: String?
    }
    let content: [ContentBlock]
}

func fetchLLMTip(conditionState: ConditionState,
                 locationType: LocationType,
                 windowType: WindowType,
                 spotNote: String?) async -> String? {

    // 1. Return cached tip if available
    let cacheKey = UserDefaultsKey.tipsCache
        + "\(conditionState.rawValue)_\(locationType.rawValue)"
    if let cached = UserDefaults.standard.string(forKey: cacheKey) {
        return cached
    }

    // 2. Require a non-empty API key
    guard let apiKey = KeychainService.shared.read(key: "anthropic_api_key"),
          !apiKey.isEmpty else { return nil }

    // 3. Build user message
    let noteText = spotNote.flatMap { $0.isEmpty ? nil : $0 } ?? "none"
    let userMessage = "Light window: \(windowType.rawValue) hour. "
        + "Condition: \(conditionState.rawValue). "
        + "Location type: \(locationType.rawValue). "
        + "Spot note: \(noteText)."

    // 4. Build HTTP request
    guard let url = URL(string: "https://api.anthropic.com/v1/messages") else { return nil }
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue(apiKey,                  forHTTPHeaderField: "x-api-key")
    request.setValue("2023-06-01",            forHTTPHeaderField: "anthropic-version")
    request.setValue("application/json",      forHTTPHeaderField: "content-type")

    let body: [String: Any] = [
        "model":      "claude-haiku-4-5-20251001",
        "max_tokens": 256,
        "system":     Prompts.systemPrompt,
        "messages": [["role": "user", "content": userMessage]]
    ]
    guard let bodyData = try? JSONSerialization.data(withJSONObject: body) else { return nil }
    request.httpBody = bodyData

    // 5. Send and decode
    do {
        let (data, _) = try await URLSession.shared.data(for: request)
        let response  = try JSONDecoder().decode(AnthropicResponse.self, from: data)
        guard let text = response.content.first(where: { $0.type == "text" })?.text,
              !text.isEmpty else { return nil }

        // 6. Cache and return
        UserDefaults.standard.set(text, forKey: cacheKey)
        return text
    } catch {
        return nil  // error absorbed
    }
}
```

**Model note:** Architecture doc specified `claude-haiku-3-5`, but the current model as of 2026 is `claude-haiku-4-5-20251001`. Use the current model. If you get a 404 or model-not-found error, fall back to `claude-haiku-3-5-20241022`.

**Cache key:** `"tipsCache_overcast_coastal"` — underscore suffix from `UserDefaultsKey.tipsCache` + rawValues joined with underscore. The cache persists across sessions (NFR13), so repeat visits to the same spot type in same conditions reuse the LLM response.

---

### SpotDetailSheet — Task Pattern and Double-Optional Unwrap

The `.task {}` modifier auto-cancels when the view disappears, which is the correct pattern for fire-and-forget async work in SwiftUI. Replace the existing `.onAppear {}` with `.task {}` and move the sync setup (`noteText = spot.note ?? ""`, `spot.lastVisited = .now`) into a new `.onAppear {}` (tasks can be delayed by a frame on first appear):

```swift
// In SpotDetailSheet.body modifiers:
.onAppear {
    noteText = spot.note ?? ""
    spot.lastVisited = .now
}
.task {
    // withTimeout returns T?? when T is String? — unwrap with flatMap
    let raw = await withTimeout(seconds: 4) {
        await TipsService.shared.fetchLLMTip(
            conditionState: conditionState,
            locationType:   spot.locationType,
            windowType:     currentWindow,
            spotNote:       spot.note
        )
    }
    // raw: String?? — nil = timeout, some(nil) = API failure, some(some(text)) = success
    let tip = raw.flatMap { $0 }
    if let tip {
        llmTipText    = tip
        llmTipVisible = true
    }
}
```

`raw.flatMap { $0 }` collapses `String??` → `String?`:
- `nil` (timeout) → `nil`
- `.some(nil)` (API error) → `nil`
- `.some(.some(text))` (success) → `.some(text)`

**State placement:** `@State private var llmTipText: String? = nil` and `@State private var llmTipVisible: Bool = false` — both declared at the top of `SpotDetailSheet`, alongside `noteText`.

---

### LLMTipBlock — Opacity-Based Visibility (No VoiceOver Announcement)

Do NOT use `if llmTipVisible { LLMTipBlock(...) }` — this removes/inserts the view from the hierarchy, which can trigger VoiceOver to announce the new element. Instead, keep the view in hierarchy at all times with opacity control:

```swift
// GoldenHour/Components/LLMTipBlock.swift
import SwiftUI

struct LLMTipBlock: View {
    let tipText: String
    let isVisible: Bool

    @Environment(\.accessibilityReduceMotion) private var reduceMotion

    var body: some View {
        VStack(alignment: .leading, spacing: 10) {
            // Divider
            Rectangle()
                .fill(Color("Rose"))
                .frame(height: 1)

            // "Tonight specifically" label
            Text("Tonight specifically")
                .font(.caption2)
                .textCase(.uppercase)
                .foregroundColor(Color("GoldenDeep"))

            // LLM tip body
            Text(tipText)
                .font(.body)
                .foregroundColor(Color("TextPrimary"))
        }
        .opacity(isVisible ? 1 : 0)
        .animation(reduceMotion ? nil : .easeIn(duration: 0.3), value: isVisible)
        .accessibilityHidden(!isVisible)
    }
}
```

**Why `accessibilityHidden(!isVisible)` and not just opacity 0:** Opacity alone keeps the element in VoiceOver's element tree (just visually invisible). `accessibilityHidden` removes it from the accessibility tree while keeping it in the visual hierarchy. This means VoiceOver cannot land on it while hidden, and when it becomes visible, VoiceOver does NOT auto-announce it — the user discovers it on their next swipe.

---

### GoldenDeep Color Asset

Add to `Assets.xcassets/Colors/GoldenDeep.colorset/Contents.json`:
```json
{
  "colors": [
    {
      "idiom": "universal",
      "color": {
        "color-space": "srgb",
        "components": {
          "red": "0.608",
          "green": "0.416",
          "blue": "0.180",
          "alpha": "1.000"
        }
      }
    }
  ],
  "info": { "author": "xcode", "version": 1 }
}
```

Also add the group container `Assets.xcassets/Colors/` → this already exists from Story 1.4. Only the new colorset folder is needed.

---

### SaveSpotSheet — Accessibility Additions

Current state from Story 2.2 — these are the targeted additions only:

```swift
// Name TextField — add accessibility label
TextField("Spot name", text: $name)
    .accessibilityLabel("Spot name, required")

// Cancel button — verify touch target ≥44pt
// Current: .font(.system(size: 16)).foregroundColor(...).frame(maxWidth: .infinity).padding(.bottom, 8)
// The frame(maxWidth:) alone doesn't guarantee height. Add explicit minimum height:
Button("Cancel") { dismiss() }
    .font(.system(size: 16, weight: .regular))
    .foregroundColor(Color("Golden"))
    .frame(maxWidth: .infinity, minHeight: 44)
```

Save button's `.disabled(!canSave)` already causes SwiftUI to announce "dimmed" or "disabled" state to VoiceOver automatically — no code change needed.

---

### UX-DR18 Copy Review Checklist

Review these strings — flag any that contain `!`, `"Please"`, or apologetic language:

| Component | String | Status |
|---|---|---|
| `MapView` | "No spots saved yet. Tap anywhere on the map to save a location." | ✅ Direct |
| `SpotDetailSheet` | "NOTE" label, "Add a note about this spot..." | ✅ Direct |
| `SaveSpotSheet` | "Save Spot" title, "Spot name" field, "NOTE" | ✅ Direct |
| `SaveSpotSheet` | "e.g. west-facing, try at golden hour" placeholder | ✅ Direct |

No changes needed — all existing strings comply with UX-DR18.

---

### Anthropic API Key Note

`KeychainService.shared.read(key: "anthropic_api_key")` returns the key that was written by `bootstrapIfNeeded()` on first launch from `APIKeys.swift`. If the key is empty or missing:
- `fetchLLMTip` returns nil (guard fails at step 2)
- `withTimeout` returns `String??.some(nil)` → `flatMap` yields nil
- `llmTipVisible` stays false → LLMTipBlock stays hidden
- No error state shown — correct behavior

---

### Files to Create vs Modify

**NEW:**
- `GoldenHour/Components/LLMTipBlock.swift`
- `GoldenHour/Assets.xcassets/Colors/GoldenDeep.colorset/Contents.json`

**MODIFY:**
- `GoldenHour/Services/TipsService.swift` — add `fetchLLMTip` + private `AnthropicResponse`
- `GoldenHour/Components/SpotDetailSheet.swift` — add `@State llmTipText/llmTipVisible`, `.task {}`, `LLMTipBlock`
- `GoldenHour/Components/SaveSpotSheet.swift` — add `.accessibilityLabel` to TextField, `minHeight: 44` to Cancel button

**DO NOT TOUCH:**
- `TipBlock.swift` — complete
- `TipsService.staticTip` — complete
- `MapView.swift` — no accessibility changes needed (SpotAnnotation already done in 2.1)
- Any Story 1.x files

---

### Testing Notes

Architecture.md: "No UI tests for v1." No new test files required. The LLM fetch is network-dependent and cannot be unit tested without mocking (out of scope for v1 personal tool). Manual verification:
1. Open SpotDetailSheet — TipBlock shows immediately; after ≤4s, LLMTipBlock fades in below
2. With no network: LLMTipBlock stays hidden; no error message shows
3. VoiceOver: navigate to LLMTipBlock after it appears — accessible via swipe, no auto-announcement
4. VoiceOver: SaveSpotSheet fields announce their labels; Save button announces disabled state

---

### Previous Story Context (Stories 2.1–2.3 Learnings)

- `withTimeout` returns `T?` where T can itself be optional — use `raw.flatMap { $0 }` to collapse `String??` → `String?`
- `assertionFailure` pattern (from 2.3 review) — TipsService already has debug assertions; the LLM fetch just returns nil on any failure, no assertion needed
- `spot.locationType` is available directly via the `@Bindable var spot: Spot` in SpotDetailSheet
- Color assets in `Assets.xcassets/Colors/` use the JSON format established in Story 1.4
- `Colors/` group `Contents.json` already exists — only add the new colorset subdirectory
- The `.task {}` modifier used in `HomeView.swift` (Story 1.4) is the right pattern for fire-and-forget async work tied to view lifecycle

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

All SourceKit cross-file reference warnings are expected (same pattern throughout this project). All files compile within the Xcode project.

Note: Used `claude-haiku-4-5-20251001` as the model (current as of 2026-05-18), not the architecture doc's `claude-haiku-3-5` (outdated). The user message for `.neither` window type uses the raw value "neither" — Prompts.systemPrompt can handle any input gracefully.

`.task {}` was added alongside the existing `.onAppear {}` in SpotDetailSheet (not replacing it), because `.task {}` can be delayed by a frame and the sync setup (`noteText`, `spot.lastVisited`) benefits from the immediate `onAppear` timing guarantee.

The `raw.flatMap { $0 }` collapse: `withTimeout` returns `String??` when `fetchLLMTip` returns `String?`. Timeout → outer nil → flatMap yields nil. API error → `some(nil)` → flatMap yields nil. Success → `some(some(text))` → flatMap yields `some(text)`.

### Completion Notes List

All 5 tasks complete. All 10 acceptance criteria satisfied:

- AC1: `fetchLLMTip` builds correct Anthropic messages API request; API key from Keychain only (NFR7)
- AC2: Cache check first; on success stores in `UserDefaultsKey.tipsCache + state_type`; persists across sessions (NFR13)
- AC3: All errors → nil return; `withTimeout(4)` applied by SpotDetailSheet `.task`; no throw propagated (NFR4, NFR11)
- AC4: `.task {}` auto-cancels on disappear; `flatMap` handles double-optional; `llmTipVisible = true` on success; LLMTipBlock stays in hierarchy (no content displacement of TipBlock)
- AC5: LLMTipBlock — 1pt Rose divider, "Tonight specifically" GoldenDeep uppercase, tip body — all per UX-DR6
- AC6: `reduceMotion` check; `.animation(nil)` when true → instant appearance
- AC7: SpotAnnotation a11y (2.1 done), TipBlock camera settings label (2.3 done), LLMTipBlock `accessibilityHidden(!isVisible)` for no auto-announcement, SaveSpotSheet TextField label added; code review found note TextEditor in SpotDetailSheet had no `.accessibilityLabel` — added `.accessibilityLabel("Spot note, optional")`
- AC8: SpotAnnotation 44pt touch target (2.1), Save button >44pt, Cancel `minHeight: 44`
- AC9: `fetchLLMTip` returns nil on network failure; LLMTipBlock stays hidden; no error state, no spinner (FR29, FR30)
- AC10: UX-DR18 compliant — no `!`, no "Please", all strings direct and factual

No new tests — architecture.md: "No UI tests for v1." LLM fetch requires real network and Anthropic API key.

### File List

**New files:**
- `GoldenHour/GoldenHour/Assets.xcassets/Colors/GoldenDeep.colorset/Contents.json`
- `GoldenHour/GoldenHour/Components/LLMTipBlock.swift`

**Modified files:**
- `GoldenHour/GoldenHour/Services/TipsService.swift` — added `AnthropicResponse` Decodable struct + `fetchLLMTip` method
- `GoldenHour/GoldenHour/Components/SpotDetailSheet.swift` — added `llmTipText`/`llmTipVisible` state, `.task {}` for LLM fetch, `LLMTipBlock`
- `GoldenHour/GoldenHour/Components/SaveSpotSheet.swift` — TextField `accessibilityLabel`, Cancel `minHeight: 44`

## Change Log

| Date | Change | Author |
|---|---|---|
| 2026-05-18 | Story created, status: ready-for-dev | bmad-create-story |
| 2026-05-18 | All 5 tasks implemented; status: review | bmad-dev-story |
| 2026-05-18 | Code review (bmad-code-review): added `.accessibilityLabel("Spot note, optional")` to TextEditor in SpotDetailSheet (AC7 gap); status: done | bmad-code-review |
