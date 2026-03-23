---
name: Trip Activities & Mileage System
description: Trip activity system with auto-ETA cascade, Google Distance Matrix, PC Miler, Samsara mileage tracking, START/END points, TRUCK_PICK/DROP
type: project
---

Trip activity mileage and ETA system built March 2026.

**3 Mileage Sources:**
- Google Distance Matrix API → planning (auto-calculated)
- PC Miler API → billing (industry-standard, key provided by user via `PC_MILER_API_KEY` env var)
- Samsara → actual GPS odometer (manual/future integration, NOT for logbooks)
- OmniTrack → separate system for ELD/Logbook/HOS (no integration planned)

**New Activity Types:** START_POINT, END_POINT, TRUCK_PICK, TRUCK_DROP

**Auto-ETA Cascade:** User enters only Start Date-Time on first activity. System calls Google Distance Matrix API for each consecutive pair and cascades: ETA = prev departure + drive time + dwell.

**Key Files:**
- `src/lib/google-distance.ts` — Google Distance Matrix API with Haversine fallback
- `src/lib/pcmiler.ts` — PC*Miler API for billing distances
- `src/lib/trip-eta-calculator.ts` — Cascade ETA calculator
- `src/app/api/trips/[id]/calculate-route/route.ts` — POST triggers calc, GET returns mileage summary

**How to apply:** When working on trip planning, activities, or mileage features, reference these files. The `calculate-route` API is the entry point for route recalculation.
