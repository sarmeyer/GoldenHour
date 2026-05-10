---
title: "Product Brief: Golden Hour"
status: "complete"
created: "2026-05-09"
updated: "2026-05-09"
inputs: []
---

# Product Brief: Golden Hour

## What This Is

Golden Hour is a personal iOS app built for one photographer (and maybe a friend) who is tired of juggling three apps to answer a simple question: *is tonight's golden hour worth going out for, and what should I do when I get there?*

This is a personal tool, not a product going to market. There is no monetization, no growth strategy, no App Store launch plan. The goal is a well-built app that makes Saturday evening shoots more intentional and more rewarding.

## The Problem (Personal)

The light around sunrise and sunset — Golden Hour and Blue Hour — produces dramatically better photos. But acting on that knowledge involves too many steps:

- **When exactly?** Sunset times are easy to find; the actual golden/blue hour window isn't.
- **Worth going out?** Cloud cover matters, but not simply. Heavy overcast ruins a shoot; broken clouds produce the most dramatic light imaginable. There's no easy way to read conditions photographically without experience.
- **Where?** A great light moment at the wrong location is still a missed shot. Scouting requires its own research.
- **What to do when I get there?** Settings and composition depend on the actual conditions — and there's no one to ask in the moment.

Existing tools don't answer all four questions in one place. Simple timers give a countdown, nothing else. Pro suites (PhotoPills, TPE) are built for photographers planning Milky Way shoots months out — overkill for a Saturday evening decision.

## The Solution

A focused iOS app that answers the four questions in order:

1. **When** — Golden Hour and Blue Hour windows for the current GPS location, updated daily
2. **Worth it?** — Weather conditions interpreted photographically, not just meteorologically: *"Broken clouds tonight — this is the good kind. Expect dramatic color."*
3. **Where** — Map-based location scouting using OpenStreetMap data, with saveable personal spots
4. **What to do** — Contextual shooting tips and camera settings based on the specific combination of light window, weather, and location type

Example: a partly cloudy Blue Hour near open water → *"Shoot toward the water to capture the soft pastel reflection. Underexpose by 1 stop to preserve sky detail. Try ISO 400, f/8, 1/60s — bracket from there."*

A clear Golden Hour in an urban environment → *"Look for long shadows between buildings. Warm light hits vertical surfaces hard — position your subject facing the light source. Start at ISO 100, f/5.6, 1/250s."*

## What Makes This Worth Building

No existing app connects conditions to guidance for someone who isn't already an expert. The contextual tips — keyed to light window + weather + location type — are the part that makes this genuinely useful rather than just another calculator. They're also the part that makes this interesting to build.

## Success Criteria

Simple: does the app make it easier to decide whether to go out, where to go, and what to try? Success is a Saturday shoot that went better because of it. There are no metrics dashboards here — just whether it's actually used and useful.

## Scope

**Platform:** iOS only, no plans to expand.

**In for v1:**
- Golden Hour and Blue Hour calculations by GPS location
- Weather integration with photographer-oriented condition language
- Map-based location scouting via OpenStreetMap, with saveable spots
- Contextual shooting tips and camera settings by conditions
- Optional push notification: daily alert when golden hour conditions look promising

**Explicitly out:**
- Android version
- Social features or photo sharing
- Gear recommendations or purchases
- User accounts or cloud sync
- Localization (English only)
- Any monetization, ever

## If It Evolves

If the app turns out to be genuinely useful, natural next additions would be: more condition types (overcast portrait light, storm light), a personal location log (where did I go, what did I find), and smarter notifications that learn what kinds of conditions actually get the user out the door.
