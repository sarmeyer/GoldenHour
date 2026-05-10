---
stepsCompleted: ["step-01-init", "step-02-discovery", "step-02b-vision", "step-02c-executive-summary", "step-03-success", "step-04-journeys", "step-05-domain", "step-06-innovation", "step-07-project-type", "step-08-scoping", "step-09-functional", "step-10-nonfunctional", "step-11-polish"]
releaseMode: single-release
classification:
  projectType: mobile_app
  domain: general
  complexity: low
  projectContext: greenfield
inputDocuments:
  - "_bmad-output/planning-artifacts/product-brief-golden-hour.md"
  - "_bmad-output/planning-artifacts/product-brief-golden-hour-distillate.md"
briefCount: 2
researchCount: 0
brainstormingCount: 0
projectDocsCount: 0
workflowType: 'prd'
---

# Product Requirements Document - Golden Hour

**Author:** Sarah
**Date:** 2026-05-09

## Executive Summary

Golden Hour is a personal iOS application for a single photographer (and potentially one friend) that eliminates decision friction around golden hour and blue hour shoots. The app answers four questions in order: *when* are the light windows for the current GPS location, *is tonight worth going out for* given actual photographic conditions, *where nearby* should I go, and *what should I try* when I get there.

The primary user is a casual weekend photographer with ~90 minutes free on a Saturday evening. The core value is the go/no-go decision: reducing friction between "the light might be good tonight" and actually leaving the house with a camera. This is a personal tool — not a market product — with no monetization, no App Store launch plan, and no growth strategy.

**Project classification:** Native iOS mobile application · Consumer lifestyle / photography utility · Greenfield · Low complexity

### What Makes This Special

No existing photography app connects conditions to actionable guidance for non-expert users. Simple timer apps (Golden Hour One, Magic Hour) provide countdown clocks and nothing more. Pro suites (PhotoPills, The Photographer's Ephemeris) offer extensive celestial tools designed for Milky Way shoots planned months out — overwhelming for a casual Saturday evening decision.

Golden Hour's differentiator is a conditions-to-guidance layer: weather state and light window type are interpreted photographically (not meteorologically) and passed to an LLM at query time, generating specific, contextual shooting tips and camera settings for the exact combination of light window × weather × location type. The language is human and opinionated (*"broken clouds tonight — this is the good kind"*), not raw meteorological data.

Core insight: cloud cover percentage alone is a misleading signal for photographers. Broken clouds at golden hour produce dramatic light; a clear sky produces flat, hard light; heavy overcast kills golden hour entirely. The app interprets conditions photographically.

## Success Criteria

### User Success

- The go/no-go decision for a golden hour outing takes less than 60 seconds from opening the app — no cross-referencing multiple apps or weather sites
- The photographic weather interpretation is trusted enough to change behavior: the user leaves the house on a "broken clouds" evening they might otherwise skip, or stays home on a "clear sky" evening that would produce flat light
- Contextual tips feel specific to actual conditions, not generic photography advice applicable to any shoot
- The app is opened during active golden hour season more than it's ignored — signal: used at least once during a real shooting opportunity rather than bypassed for other tools
- The night-before planning workflow works as naturally as the same-evening workflow

### Business Success

Personal tool with no commercial success criteria. Success is personal utility:

- The builder uses it regularly during golden hour season as a primary decision tool — not a novelty that gets uninstalled after two weeks
- At least one real-world shooting session is meaningfully influenced by the app (location chosen via the map, or tips applied on-location)
- Tips remain useful as the primary user's conditions experience grows — educational, not just confirmatory

### Technical Success

- **Accuracy:** Golden hour and blue hour window calculations within ±2 minutes of actual observed conditions for the GPS location
- **Interpretation quality:** Weather state always maps to a defined photographic condition type — never falls through to a raw meteorological readout
- **Tips relevance:** LLM-generated tips reference the specific light window, weather state, and location type — not generic advice
- **Performance:** Home screen loads in under 2 seconds; map loads within 3 seconds on standard connectivity
- **Connectivity model:** Light window calculations and saved spots work fully offline; weather and LLM tips require connectivity, with clear UI state when unavailable
- **Location handling:** App gracefully degrades when GPS is unavailable or denied — clear messaging, fallback option

### Measurable Outcomes

Lightweight personal signals, not dashboard metrics:

- App opened at least once during each golden hour shooting opportunity for the first 3 months post-build
- At least one saved shooting spot added within the first month
- No instances of the interpretation layer producing meteorologically-correct but photographically-misleading output (e.g., flagging a clear sky as exciting)

## Product Scope

### Must-Have Capabilities

All core features ship together as a single v1 release. The features are interdependent — a partial release would not deliver the core value proposition.

| Capability | Notes |
|---|---|
| Light window calculation (golden + blue hour) by GPS | Foundational "when" answer; computed on-device |
| Synthesized go/no-go verdict | Single sentence above the fold combining light window + conditions: *"Go now. 34 min left, clouds are cooperating."* / *"Skip tonight. Overcast is flat."* |
| Photographic weather interpretation with opinionated language | Feeds the verdict and tips prompt context |
| Map with saved personal spots + notes | Plain map rendering; `lastVisited` timestamp on spot data model for future location log |
| LLM tips with static-first render | Static tips display immediately on spot selection; LLM tips replace within ~4s if available; offline → static tips permanently |
| Static fallback tips (~8 condition × light window sets) | First render for all tip requests; written for the person already on-location |
| Graceful offline states | Light window + saved spots fully offline; weather + LLM degrade with clear messaging |

**Nice-to-have (v1, ship if time permits):** Manual location entry · Next-day light window lookup

### Static Fallback Tips Coverage

| Light Window | Condition |
|---|---|
| Golden Hour | Broken clouds |
| Golden Hour | Clear sky |
| Golden Hour | Overcast |
| Golden Hour | Fog |
| Golden Hour | Rain |
| Blue Hour | Broken clouds |
| Blue Hour | Clear sky |
| Blue Hour | Overcast |

### Post-v1 Roadmap

- Push notifications — daily condition-aware alert (deferred to keep v1 scope tight)
- OSM POI filtering — photography-relevant location discovery (deferred; inconsistent tagging, plain map sufficient for 1-2 users)
- Personal location log — shooting history tied to real outcomes
- Additional condition types — overcast portrait light, storm light, blue-sky architecture
- Smarter notifications — learn which conditions actually get the user out the door
- Manual location entry (if not included in v1)

### Risk Mitigation

| Risk | Mitigation |
|---|---|
| Weather → condition mapping inconsistency | Single `ConditionState` enum; all display logic, LLM prompts, and fallback selection derive from it |
| LLM latency | Static tips render immediately; LLM silently upgrades within ~4s — users see content instantly |
| Solar calculation accuracy | Validate against known golden hour times; handle high-latitude edge cases and DST transitions |
| OSM tile server rate limits | Use Maptiler or Stadia Maps free tier; not tile.openstreetmap.org directly |
| Anthropic API key exposure | iOS Keychain; read from excluded local config at first launch, write to Keychain, read from Keychain thereafter |
| Open-Meteo reliability | Cache last successful response with TTL; show last-known conditions with timestamp on failure |

**Resource contingency:** Solo builder — if scope is too large, the synthesized verdict is the last thing to cut.

## User Journeys

### Journey 1: Saturday Evening, Same-Day Decision (Primary — Success Path)

**The Persona:** Sarah is a software engineer who shoots on weekends when the light is right. She has a mirrorless camera she actually uses, a few spots she likes, and a recurring problem: decision friction. Checking sunset time, cross-referencing a weather app, googling whether clouds help or hurt golden hour — by the time she's done, she's talked herself out of it.

**Opening Scene:** 5:30pm on a Saturday. Sarah has two hours before dinner. She opens Golden Hour.

**Rising Action:** The home screen loads immediately. The verdict is prominent: *"Go tonight. Broken clouds — this is the good kind. Expect color."* Golden hour is 6:47pm to 7:21pm; blue hour follows until 7:48pm. She doesn't need a cloud cover percentage. She knows what broken clouds mean now.

She opens the map. GPS is already centered. She taps the reservoir overlook she saved last month. Tips load instantly: *"Shoot toward the water to capture the reflection as the color builds. Underexpose by 1 stop to preserve sky detail. Try ISO 400, f/8, 1/60s — bracket from there."*

**Climax:** She grabs her bag. Decision made without friction, with enough specificity to feel prepared rather than anxious about settings when she arrives.

**Resolution:** She's at the reservoir by 6:30pm, set up before the window opens. Three frames she's genuinely happy with. The app made her go out. That's the whole job.

**Capabilities revealed:** Home screen light window display, synthesized go/no-go verdict, photographic condition summary, GPS-centered map, saved spots, location-specific LLM tips, camera settings in tips.

---

### Journey 2: Night-Before Planning (Primary — Alternate Path)

**Opening Scene:** Friday evening on the couch. Sarah has a free afternoon tomorrow and wonders if the light will be worth planning around.

**Rising Action:** She opens Golden Hour and checks tomorrow's window — golden hour is 6:52pm to 7:18pm. Real-time conditions won't be available until tomorrow, but she can scout now. She browses the map, finds an elevated park on the east side of town with an unobstructed western view she hasn't tried. She saves it: *"try tomorrow, west-facing."*

**Climax:** Scouting done, spot saved. She sets a calendar reminder for 6pm tomorrow — enough time to check the app and get out the door if conditions look good.

**Resolution:** Saturday afternoon she opens Golden Hour. *"Go tonight. Broken clouds — conditions look good."* The spot she found the night before is already there. She goes.

**Capabilities revealed:** Next-day light window lookup, map browsing and saving without current-conditions dependency, saved spots as planning artifacts.

---

### Journey 3: Conditions Say No (Primary — Edge Case / Correct Outcome)

**Opening Scene:** Tuesday evening. Meetings ended early, golden hour is in 90 minutes. Sarah opens Golden Hour hoping.

**Rising Action:** The verdict: *"Skip tonight. Heavy overcast — this kills golden hour. Flat, gray light."* She taps the map, selects a spot anyway. The tips: *"Overcast light is soft and diffused — good for portraits or street work. Not a golden hour shoot. If you go, focus on texture and shadow-free subjects rather than chasing color."* Honest. No pretense.

**Resolution:** She doesn't go. The right call, and the app helped her make it. Two weeks later, it says broken clouds — she goes without hesitation, because she trusted the read when it said no.

**Capabilities revealed:** Condition interpretation that communicates negative outcomes clearly, honest tips that reframe rather than hype poor conditions, trust built through accuracy over time.

---

### Journey Requirements Summary

| Capability | Required by Journey |
|---|---|
| Light window calculation (today + tomorrow) | 1, 2 |
| Synthesized go/no-go verdict | 1, 3 |
| Photographic condition summary (opinionated language) | 1, 3 |
| GPS-centered map | 1, 2 |
| Saved personal spots with notes | 1, 2 |
| LLM-generated tips keyed to conditions + location | 1, 3 |
| Camera settings in tip output | 1 |
| Honest negative-condition interpretation | 3 |
| Offline-capable light window display | 1, 2, 3 |

## Innovation & Novel Patterns

### Detected Innovation Areas

**LLM as a real-time interpretation layer for domain-specific contextual guidance**

Golden Hour uses a large language model not for chat or content generation — but as a live interpretation engine that synthesizes real-world sensor data into opinionated, domain-specific advice at the moment of need. Input is structured: light window type (golden/blue) + photographic condition state + location type. Output is a human-voice shooting guide with specific camera settings and composition direction, generated fresh for each query.

This differs from the two dominant approaches in consumer utility apps:
- **Raw data display** — show numbers, let the user interpret (weather apps, PhotoPills ephemeris)
- **Hand-authored static tips** — finite, expensive to maintain, can't cover the full permutation space

The LLM approach handles permutations dynamically. "Broken clouds + golden hour + coastal" produces different tips than "broken clouds + golden hour + urban canyon" without requiring manual authorship.

**Photographic reinterpretation of meteorological data**

The weather interpretation layer takes a raw API response (cloud cover %, precipitation probability, visibility, weather code) and maps it to a photographer-specific condition state with an opinionated narrative. "45% cloud cover, partly cloudy" becomes *"broken clouds tonight — this is the good kind."*

### Market Context

No existing photography utility app applies an LLM to real-time conditions for on-demand contextual guidance:
- Simple timers (Golden Hour One, Magic Hour): no weather, no tips
- Pro planning suites (PhotoPills, TPE): extensive data, zero interpretation for non-experts
- Weather apps: meteorological framing, no photographic translation

The LLM interpretation layer fills this gap — practical for a personal project only recently, given API-accessible LLMs at current pricing.

### Validation Approach

Innovation risk is quality consistency, not technical feasibility:

- **Prompt adherence:** Tips must reference specific input conditions — a tip applicable to any conditions is a failure
- **Camera settings specificity:** Concrete starting-point settings (ISO, aperture, shutter speed), not "adjust for available light"
- **Honest negative-condition handling:** Poor conditions get reframed honestly, not hyped
- **Tone consistency:** Knowledgeable friend, not photography textbook — prompt engineering is the primary control

Validation method: manual spot-checking across condition × location type combinations before ship.

### Risk Mitigation

| Risk | Mitigation |
|---|---|
| LLM API unavailable | Static fallback tips always display first; LLM unavailability means tips stay on static — no error state shown |
| LLM produces generic tips | Strict output format in system prompt; condition-reference requirement; tested before ship |
| Tips feel repetitive | Session variety instructions in prompt; vary emphasis across sessions |
| LLM hallucinates inappropriate settings | Photography physics constraints in system prompt; manual validation during development |
| API cost | At 2–4 sessions/month, negligible |

## Mobile App Specific Requirements

### Project-Type Overview

Golden Hour is a native iOS 18+ application installed directly on device (Xcode or TestFlight) — not App Store distributed. The app is foreground-first with no background capabilities in v1. All user data is stored locally on-device with no cloud sync or backend infrastructure.

### Technical Architecture

- **Platform:** iOS 18+, Swift, SwiftUI
- **Weather data:** Open-Meteo API — free, no API key required, JSON. Fields: `cloud_cover`, `precipitation_probability`, `visibility`, `weather_code`
- **LLM API:** Anthropic Claude API — query-time tip generation; API key stored in iOS Keychain
- **Maps:** MapKit with OSM raster tile overlay (Maptiler or Stadia Maps free tier); no POI filtering in v1
- **Light window calculations:** On-device astronomical computation; solar position library selection is architecture-phase decision
- **Local persistence:** SwiftData or CoreData for saved spots; no iCloud sync
- **Condition logic:** Single `ConditionState` enum drives all condition-dependent display, LLM prompt context, and fallback tip selection

### Platform Requirements

- **Minimum OS:** iOS 18
- **Target devices:** iPhone only; no iPad optimization
- **Distribution:** Personal device via Xcode or TestFlight; no App Store submission
- **Orientation:** Portrait primary; landscape not required
- **Accessibility:** Standard iOS Dynamic Type and VoiceOver via SwiftUI default behaviors

### Device Permissions

| Permission | Usage | Behavior if Denied |
|---|---|---|
| Location (When In Use) | GPS for light window calculation and map centering | Location-unavailable state shown; manual entry fallback |

No camera, microphone, photo library, notification, or contacts access required in v1.

### Offline Mode

| Feature | Offline Behavior |
|---|---|
| Light window calculation | Fully available — on-device computation |
| Saved spots | Fully available — local storage |
| Map tiles | Dependent on tile cache; degrades if not cached |
| Weather interpretation | Shows last-fetched condition with timestamp, or "conditions unavailable" |
| LLM tips | Static fallback tips display; no error state |

### Store Compliance

No App Store submission planned:
- No App Store review guidelines compliance required
- No privacy nutrition label required
- No in-app purchase framework needed
- Standard Apple developer account required for device provisioning

### Implementation Considerations

- **Open-Meteo:** REST, no auth; cache responses with ~1 hour TTL
- **Anthropic API:** REST with structured prompt; key in Keychain (read from gitignored local config at first launch, write to Keychain, read from Keychain thereafter); cache response per condition × location type per session
- **Map tiles:** Maptiler or Stadia Maps free tier; OSM attribution per provider terms; no POI filtering
- **Light window library:** Solar or SunCalc Swift port; validate against known golden hour times before ship

## Functional Requirements

**Note:** Capability contract for all downstream work. UX design, architecture, and epic breakdown cover only what is listed here.

### Light Window Calculation

- **FR1:** The app calculates the golden hour window (start time, end time, duration) for the user's current GPS location for today
- **FR2:** The app calculates the blue hour window (start time, end time, duration) for the user's current GPS location for today
- **FR3:** The app displays tomorrow's golden and blue hour window times for advance planning
- **FR4:** Light window calculations update when the user's GPS location changes significantly between sessions

### Condition Assessment

- **FR5:** The app fetches current weather conditions from Open-Meteo for the user's GPS location
- **FR6:** The app maps raw weather data (cloud cover, precipitation probability, visibility, weather code) to a photographer-specific condition state (broken clouds, clear sky, overcast, fog, precipitation)
- **FR7:** The app displays a human-language condition summary using opinionated photographic framing — not raw meteorological data
- **FR8:** The app displays a synthesized go/no-go verdict combining light window timing and condition quality into a single actionable sentence
- **FR9:** The app displays last-successfully-fetched condition data with a visible timestamp when a fresh fetch is unavailable
- **FR10:** The app communicates clearly when condition data is unavailable, distinguishing between offline and stale-but-present states

### Location & Spots

- **FR11:** The app displays a map centered on the user's current GPS location
- **FR12:** Users can browse the map to explore potential shooting locations
- **FR13:** Users can save a map location as a personal shooting spot with a custom name
- **FR14:** Users can add or edit a text note on a saved spot
- **FR15:** Users can assign a location type category to a saved spot (e.g., coastal, urban, forest, open field) for use in tip generation
- **FR16:** Users can view all saved spots on the map
- **FR17:** Users can delete a saved spot
- **FR18:** Saved spots persist locally on-device across app sessions with no cloud sync
- **FR19:** The saved spot data model records a last-visited timestamp for each spot

### Shooting Guidance

- **FR20:** The app displays shooting tips for a selected location keyed to the current light window type and condition state
- **FR21:** The app displays camera settings (ISO, aperture, shutter speed) as starting-point recommendations within tip content
- **FR22:** Tips display immediately using static fallback content when a location is selected, before any LLM response is available
- **FR23:** The app upgrades displayed tips to LLM-generated content when an Anthropic API response arrives within a defined timeout
- **FR24:** LLM-generated tips reference the specific light window type, condition state, and location type — not generic photography advice
- **FR25:** The app selects the appropriate static fallback tip set based on current condition state and light window type
- **FR26:** Tips adapt honestly to poor conditions — reframing (e.g., soft light for portraits) rather than overstating photographic potential

### Offline & Connectivity

- **FR27:** Light window calculations are fully available without network connectivity
- **FR28:** Saved spots are fully available without network connectivity
- **FR29:** The app displays a coherent, useful state when offline — static tips always available; weather and LLM content degrade with clear messaging
- **FR30:** The app communicates which specific features are unavailable and why when connectivity is absent

### App Configuration & Location Services

- **FR31:** The app requests location permission and uses GPS coordinates as the primary input for all location-dependent calculations
- **FR32:** The app displays a location-unavailable state with a clear explanation and fallback option when location permission is denied or GPS is unavailable
- **FR33:** The app stores the Anthropic API key in iOS Keychain — not in plaintext or hardcoded in the binary
- **FR34:** Users can manually trigger a refresh of current weather conditions

## Non-Functional Requirements

### Performance

- **NFR1:** Home screen (light window times + condition summary) loads within 2 seconds from cold launch
- **NFR2:** Map renders and saved spots display within 3 seconds on standard connectivity
- **NFR3:** Static tips display within 200ms of location selection
- **NFR4:** LLM tip requests time out after 4 seconds; app remains on static tips without showing an error state
- **NFR5:** Weather data cached with minimum 60-minute TTL; fresh fetches not triggered more than once per hour per location
- **NFR6:** Light window calculations complete in under 100ms

### Security

- **NFR7:** The Anthropic API key is stored in iOS Keychain — not in the app bundle, source code, or any plaintext configuration file
- **NFR8:** The app transmits user data only to: Open-Meteo (GPS coordinates) and Anthropic (condition state + location type) — no other external data transmission
- **NFR9:** The app collects no analytics, no external crash reporting, and no usage telemetry

### Integration Resilience

- **NFR10:** Weather fetch failure must not prevent the app from displaying cached conditions (with timestamp) or a clear unavailable state
- **NFR11:** LLM API failure must not prevent the app from displaying static fallback tips — the tip section is never empty
- **NFR12:** Map tile unavailability must not prevent access to the saved spots list view
- **NFR13:** Open-Meteo and Anthropic API responses are cached to avoid redundant calls within a session

### Accessibility

- **NFR14:** All text scales with the user's iOS Dynamic Type setting — no fixed font sizes
- **NFR15:** The app is navigable via VoiceOver through standard SwiftUI accessibility semantics
