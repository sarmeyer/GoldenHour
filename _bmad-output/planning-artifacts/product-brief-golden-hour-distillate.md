---
title: "Product Brief Distillate: golden-hour"
type: llm-distillate
source: "product-brief-golden-hour.md"
created: "2026-05-09"
purpose: "Token-efficient context for downstream PRD creation"
---

# Product Brief Distillate: Golden Hour

## Project Character

- Personal iOS app for 1-2 users (builder + one friend), not a product going to market
- No App Store launch plan, no growth strategy, no monetization — ever
- Goal: a well-built personal tool that makes golden hour shoots more intentional
- Builder is the primary user — dogfooding is the primary feedback loop

## Core User Scenario

- Casual weekend photographer with ~90 min free on a Saturday evening
- Decision flow: open app → see golden/blue hour window + weather interpretation → browse nearby spots on map → read contextual tips → go out
- Also used the night before to plan ahead
- Inherently low-frequency use: 2–4 meaningful shooting sessions per month

## The Four-Question Flow (core UX model)

The app answers these four questions in order — this is the fundamental design contract:

1. **When** — what are the exact golden/blue hour windows today for my location?
2. **Worth it?** — should I actually go out, given today's conditions?
3. **Where** — where nearby should I go?
4. **What to do** — what should I actually try when I get there?

Every feature exists to answer one of these four questions. Anything that doesn't serve the flow is out of scope.

## Weather Interpretation (critical product decision)

- Cloud cover % alone is an inadequate and misleading signal for photographers
- The app must interpret conditions photographically, not meteorologically:
  - Broken clouds at golden hour = the best possible light (dramatic color, texture) — app should flag this positively
  - Clear sky at golden hour = flat, hard light — less exciting than it sounds
  - Heavy overcast = kills golden hour entirely but can be good for portraits (soft, diffused)
  - Fog = high-interest atmospheric conditions worth flagging as a special case
- Weather language should be human and opinionated: *"Broken clouds tonight — this is the good kind. Expect dramatic color."* Not: *"Cloud cover: 45%"*
- This interpretation layer is a core differentiator, not a cosmetic choice

## Contextual Tips Engine (the interesting part to build)

- The headline differentiator: conditions → specific, actionable shooting guidance
- Tips are keyed to the combination of: light window type (golden/blue) × weather state × location type
- Tips include both camera settings (as starting points, not prescriptions) AND composition/subject guidance
- Concrete example — partly cloudy Blue Hour near water:
  - *"Shoot toward the water to capture the soft pastel reflection. Underexpose 1 stop to preserve sky detail. Try ISO 400, f/8, 1/60s — bracket from there."*
- Concrete example — clear Golden Hour in urban environment:
  - *"Look for long shadows between buildings. Warm light hits vertical surfaces hard — position your subject facing the light source. Start at ISO 100, f/5.6, 1/250s."*
- Implementation approach for tips engine: **not resolved** — could be rules-based matrix, hand-authored content keyed by condition state, or LLM-generated at query time. PRD should decide.
- Condition permutation scope: **not resolved** — need to define how many location types, weather states, and light window combinations to handle at v1

## Technical Constraints & Platform Decisions

- **iOS only** — no Android, no web, no desktop, ever
- **OpenStreetMap** selected as the map/location data source (explicitly chosen over proprietary map APIs)
- **GPS-based location** — app centers on the user's current location; manual location entry is a nice-to-have, not a requirement
- **No backend, no user accounts, no cloud sync** — all data local to the device
- **Weather API**: not yet selected — needs real-time conditions data including cloud cover, precipitation, visibility, and ideally fog/atmospheric data
- **Push notifications**: optional daily alert when golden hour conditions look promising — iOS permission required; daily cadence, timing TBD
- **Location permissions**: core functionality requires location access; degraded experience if denied

## Scope Decisions (explicit)

**Confirmed in for v1:**
- Golden Hour + Blue Hour window calculation by GPS
- Photographer-oriented weather interpretation
- Map with browseable nearby locations (OpenStreetMap)
- Saveable personal shooting spots
- Contextual tips + camera settings by conditions
- Optional daily push notification

**Confirmed out (permanently):**
- Monetization of any kind
- Android version
- Social features / photo sharing
- Gear recommendations or affiliate links
- User accounts or cloud sync
- Localization (English only)

**Confirmed out for v1, possible later:**
- Personal location log (where did I go, what did I find)
- Additional condition types: overcast portrait light, storm light, blue-sky architecture
- Smarter/personalized notifications based on usage patterns
- "Shot of the day" daily prompt (surfaced during review, not committed to)

## Competitive Context (for requirements grounding, not for differentiation messaging)

- **PhotoPills** (~$10.99 one-time): gold standard for pros — AR sun/moon, Milky Way planner, DOF calculators. Overwhelming for casual users. No contextual guidance.
- **The Photographer's Ephemeris (TPE)**: map-first sun/moon positioning. Strong on where the light falls, silent on what to do with it. No weather integration.
- **Sun Surveyor**: AR utility tool. No planning, no tips.
- **Golden Hour One / Magic Hour**: simple countdown timers. One trick, nothing else.
- **PlanIt! for Photographers**: comprehensive (maps, weather, celestial), another power-user suite.
- **Common gap across all**: none connect conditions to actionable guidance for a non-expert user. That's the space Golden Hour fills.

## Rejected Ideas (do not re-propose)

- **Monetization / freemium tier**: explicitly rejected — personal project, free forever by design
- **Generic educational tips**: rejected in favor of condition-specific contextual guidance; generic tips are everywhere already
- **Social / community features**: out of scope for a 1-2 user personal tool
- **WAU/retention metrics dashboards**: irrelevant for a personal project; success is subjective ("did this help me get a better shot?")
- **App Store optimization / launch strategy**: not applicable

## Open Questions for PRD

- How does the contextual tips engine work technically? Rules-based matrix vs. LLM at query time vs. hand-authored content?
- What are the defined location types for tip generation? (e.g., coastal, urban, forest, open field, mountain — how granular?)
- Which weather API? What data fields are required beyond basic cloud cover?
- How many condition combinations need tip coverage at v1 launch?
- What is the notification timing logic? Fixed time daily, or calculated relative to the golden hour window?
- What happens when GPS is unavailable or denied? Graceful fallback needed.
- How is OpenStreetMap POI data filtered to photography-relevant locations? (Not every OSM point is a shooting spot)
