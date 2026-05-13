# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

Pre-implementation. The PRD is the source of truth. No build tooling, dependencies, or code exist yet. When scaffolding the project, use the stack below.

## Stack

- **Frontend**: React, TypeScript
- **Map**: Leaflet + `react-leaflet`, `leaflet.markercluster`
- **Tiles**: OpenStreetMap (no API key required)
- **Backend/DB**: Supabase (Postgres + Auth + Storage)
- **Hosting**: Vercel (Hobby plan) — no Vercel-specific features; keep it portable

## Module Architecture

The codebase is split into six modules. Each is a pure interface — no cross-module internal coupling.

| Module | Responsibility | Public interface |
|---|---|---|
| **Map** | Leaflet map, shoe emoji pins, clustering, click capture | props: `sightings`, `onPinClick(id)`, `onMapClick(lat, lng)` |
| **Sighting** | All CRUD on the `sightings` table | `getSightings()`, `getSighting(id)`, `createSighting(data)`, `updateSighting(id, data)`, `deleteSighting(id)` |
| **Confirmation** | Still-there / gone voting; updates `gone_count` and `status` on the parent sighting | `getConfirmation(sightingId)`, `upsertConfirmation(sightingId, type)`, `deleteConfirmation(sightingId)` |
| **Photo Upload** | Upload to Supabase Storage, return public URL | `uploadPhoto(file) → url` |
| **Auth** | Magic link flow; gates display-name setup screen on first login | `signIn(email)`, `signOut()`, session state |
| **Profile** | Profile row creation and reads | `getProfile(userId)`, `createProfile(displayName)` |
| **Modal** | Three modal variants sharing a shell: Sighting Detail, New Sighting (form), Edit Sighting (pre-filled form) | — |

## Data Schema

See `PRD.md` for the full column list. Key invariants:

- **`sightings.status`** is always derived from `gone_count`; it is stored for read performance, not as ground truth.
- **Status thresholds are app-level constants** — never hardcode `3` or `5` in logic, always reference named constants (e.g. `GONE_MISSING_THRESHOLD = 3`, `AWOL_THRESHOLD = 5`).
- **Soft delete** everywhere: `deleted_at` on `sightings` and `comments`. Non-admin SELECT policies exclude rows where `deleted_at IS NOT NULL`.
- **Location is immutable**: `lat`/`lng` on a sighting cannot be edited after creation.
- **`confirmations`** is unique on `(sighting_id, confirmed_by)`. Use upsert — don't insert a second row to flip a vote.
- **`display_name`** is NOT NULL and unique case-insensitively. Enforce at the DB level.
- `comments` and `reports` tables ship with schema only — no UI in v1.

## Auth Rules

- Magic link is the only auth method — no password, no OAuth.
- After magic link confirmation, gate everything behind a mandatory display-name setup screen if `profiles` row doesn't exist yet.
- RLS: SELECT is public; INSERT/UPDATE/DELETE requires authentication; UPDATE and soft-delete of sightings restricted to owner or `is_admin = true`.

## Testing Priorities

Tests verify external behavior through public module interfaces — not Supabase client internals or CSS classes.

1. **Sighting Module** — highest priority: CRUD shape, soft-delete visibility, RLS blocking unauthenticated insert.
2. **Confirmation Module** — vote insert, flip, retract; `gone_count` and `status` transition at thresholds 3 and 5.
3. **Photo Upload Module** — valid file → URL; oversized / wrong-type files rejected before upload.
4. **Auth Module** — session state transitions through the full magic-link + display-name flow.
5. **Profile Module** — case-insensitive uniqueness enforcement; missing profile gates map interaction.
6. **Map Module** — lower priority; `onMapClick` and `onPinClick` callback correctness.
7. **Modal Module** — form validation: 280-char description limit, date required, submission payload.

## Tone

The project is intentionally silly. UI copy and empty states should match that tone (e.g. "No shoes lost here... yet.", "This shoe has gone AWOL."). The app name is **"Where's My Shoe?"**.
