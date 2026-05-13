# PRD: Where's My Shoe?

## Problem Statement

Single shoes appear abandoned in the most random places — highways, gas stations, parks — with no explanation. There is no way to document, share, or collectively appreciate these mysterious sightings. People who notice them have nowhere to report them, and the internet has no map of this phenomenon.

## Solution

A web app where anyone can browse a world map of documented single-shoe sightings, and registered users can contribute new sightings by dropping a pin, adding a photo, and writing a short description. The tone is light and fun — this is a community curiosity project, not a utility.

## User Stories

1. As a visitor, I want to see a map with all reported shoe sightings, so that I can browse the phenomenon without needing to register.
2. As a visitor, I want to zoom and pan the map freely, so that I can explore sightings in specific areas.
3. As a visitor, I want clustered pins when many sightings are close together, so that the map remains readable at any zoom level.
4. As a visitor, I want to click a pin to open a modal with the sighting details, so that I can see the photo, description, and date of that sighting.
5. As a visitor, I want to see the shoe emoji as the pin icon, so that the map is immediately self-explanatory and fun.
6. As a visitor, I want to access the site on my phone browser, so that I can report a sighting in the moment I find a shoe.
7. As a visitor, I want to see a "buy me a coffee" link, so that I can support the creator if I enjoy the project.
8. As an unregistered user, I want to see a clear prompt to sign in when I try to add a pin, so that I understand why I can't post yet.
9. As a new user, I want to register using only my email address, so that I can start contributing without creating a password.
10. As a new user, I want to receive a magic link in my email to log in, so that I don't have to remember a password.
11. As a new user, I want to be prompted to choose a display name immediately after confirming my magic link, so that my contributions are attributed from the start.
12. As a registered user, I want to click anywhere on the map to place a new pin, so that I can report a shoe sighting at the exact location.
13. As a registered user, I want a modal to open after placing a pin, so that I can fill in the sighting details before saving.
14. As a registered user, I want the date field in the new sighting form to auto-fill to today, so that I don't have to type it manually.
15. As a registered user, I want to adjust the date in the form, so that I can report a sighting I saw earlier.
16. As a registered user, I want to optionally upload one photo of the shoe, so that others can see what it looked like.
17. As a registered user, I want to optionally add a short description (up to 280 characters), so that I can share context or humor about the sighting.
18. As a registered user, I want to confirm or cancel before saving a pin, so that I don't accidentally publish a sighting.
19. As a registered user, I want to see my new pin appear on the map immediately after saving, so that I get instant feedback that it was submitted.
20. As a registered user, I want to edit the description, date, or photo of my own sightings, so that I can correct mistakes after posting.
21. As a registered user, I want to delete my own sightings, so that I can remove something I posted by mistake.
22. As a registered user, I want to mark a sighting as "still there" when I pass by it, so that others know the shoe is still around.
23. As a registered user, I want to mark a sighting as "gone" when the shoe has disappeared, so that the community knows it has moved on.
24. As a registered user, I want to change my vote on a sighting at any time, so that I can update my report if the situation has changed.
25. As a registered user, I want to remain logged in across sessions, so that I don't have to re-authenticate every time I visit.
26. As a registered user, I want to log out, so that I can end my session on a shared device.

## Implementation Decisions

### Modules

**Map Module**
The core interactive map rendered with Leaflet and `react-leaflet`. Responsible for displaying tiles (OpenStreetMap), rendering all shoe pins with a shoe emoji icon, handling marker clustering via `leaflet.markercluster`, and capturing click events on the map canvas to initiate new sightings. Pin appearance reflects sighting status (active, gone missing, AWOL). Interface: receives a list of sightings and two callbacks (`onPinClick`, `onMapClick`) and renders accordingly.

**Sighting Module**
Encapsulates all data operations for shoe sightings: fetching all sightings, creating, updating, and soft-deleting a sighting. Backed by a Supabase Postgres table. Owns the data schema and is the single source of truth for sighting state. Interface: `getSightings()`, `getSighting(id)`, `createSighting(data)`, `updateSighting(id, data)`, `deleteSighting(id)`.

**Confirmation Module**
Handles the "still there" / "gone" voting mechanic. Each user holds one vote per sighting; pressing the opposite option flips the vote. Updates `sightings.gone_count` and `sightings.status` when thresholds are crossed. Interface: `getConfirmation(sightingId)`, `upsertConfirmation(sightingId, type)`, `deleteConfirmation(sightingId)`.

**Photo Upload Module**
Handles uploading a photo to Supabase Storage and returning a public URL. Decoupled from the sighting module — the sighting record stores only the URL. Interface: `uploadPhoto(file) -> url`.

**Auth Module**
Wraps Supabase Auth with magic link flow. Exposes current session state and `signIn(email)` / `signOut()` actions. After magic link confirmation, gates access behind a mandatory display name setup screen if the user has no profile yet. Used by the navbar and to gate the map click handler.

**Profile Module**
Manages user profile creation and reads. Creates a profile row on first login (after display name is set). Interface: `getProfile(userId)`, `createProfile(displayName)`.

**Modal Module**
Three modal variants sharing a common shell:
- *Sighting Detail Modal* — view of a pin's photo, description, date, status, and voting buttons. Opened by clicking an existing pin.
- *New Sighting Modal* — form with date (auto-filled), optional photo upload, optional description. Opened after placing a new pin. Handles validation (280-char limit) and submission.
- *Edit Sighting Modal* — pre-filled form for updating description, date, or photo. Location is not editable; the original pin position is permanent.

### Data Schema

**profiles table**
- `id` — uuid, primary key, foreign key to auth.users
- `display_name` — text, NOT NULL, unique (case-insensitive)
- `avatar_url` — text, nullable
- `is_admin` — boolean, default false
- `created_at` — timestamp

**sightings table**
- `id` — uuid, primary key
- `lat` — float, required
- `lng` — float, required
- `description` — text, nullable, max 280 chars
- `photo_url` — text, nullable
- `spotted_at` — date, required (user-adjustable, defaults to today)
- `status` — varchar, default `'active'` (`'active'`, `'gone_missing'`, `'awol'`)
- `gone_count` — integer, default 0 (net gone votes; updated by app logic on each confirmation change)
- `created_by` — uuid, foreign key to auth.users
- `created_at` — timestamp
- `deleted_at` — timestamp, nullable (soft delete)

**confirmations table**
- `id` — uuid, primary key
- `sighting_id` — uuid, foreign key to sightings
- `confirmed_by` — uuid, foreign key to auth.users
- `type` — varchar (`'still_there'`, `'gone'`)
- `created_at` — timestamp (first vote)
- `last_confirmed_at` — timestamp (last time this vote was re-affirmed)
- Unique on `(sighting_id, confirmed_by)`

**comments table** *(schema only — UI deferred)*
- `id` — uuid, primary key
- `sighting_id` — uuid, foreign key to sightings
- `author_id` — uuid, foreign key to auth.users
- `body` — text, max 500 chars
- `created_at` — timestamp
- `deleted_at` — timestamp, nullable (soft delete)

**reports table** *(schema only — UI deferred)*
- `id` — uuid, primary key
- `reported_by` — uuid, foreign key to auth.users
- `target_type` — varchar (`'sighting'`, `'comment'`)
- `target_id` — uuid
- `reason` — text, nullable
- `created_at` — timestamp

### Sighting Status Mechanic

Sighting status is driven by `gone_count`, which represents net gone votes (gone votes minus still-there votes). Thresholds are app-level constants, not hardcoded in the database.

| `gone_count` | `status`       | Map behaviour                              |
|-------------|----------------|--------------------------------------------|
| < 3          | `active`       | Normal shoe emoji pin                      |
| 3–4          | `gone_missing` | Altered pin style; community nudged to check |
| ≥ 5          | `awol`         | Pin marked AWOL; "Still there" button disabled |

Status is stored on the sighting row for fast map rendering. It is updated by application logic whenever a confirmation is inserted, updated, or deleted.

### Auth & Access Control
- Supabase Row Level Security (RLS): all users can SELECT sightings and confirmations; only authenticated users can INSERT sightings and confirmations; only the sighting owner can UPDATE or soft-delete their own sightings; `is_admin` users can soft-delete any sighting.
- Deleted sightings (`deleted_at IS NOT NULL`) are excluded from all SELECT policies for non-admin users.
- Photo storage bucket: public read, authenticated write.
- Magic link is the only auth method — no password, no OAuth.
- Display name is required before a user can interact with the map. It must be set in a one-time screen after magic link confirmation.

### Deployment
- Hosted on Vercel (Hobby plan). Environment variables managed via Vercel dashboard.
- Supabase project on free tier.
- No Vercel-specific features used — zero lock-in for future self-hosting migration.

### Monetisation
- Ko-fi (or equivalent) button/link in the footer. No backend involvement — external link only.

## Testing Decisions

Good tests for this project verify external behavior through public interfaces, not implementation details. They should not depend on component internals, Supabase client internals, or specific CSS classes.

**Sighting Module** — highest priority for tests. Pure data logic, no UI. Tests should cover: fetching returns correct shape, creating a sighting persists correctly, updating only allows description/date/photo changes, soft delete sets `deleted_at` and hides the row from non-admin queries, RLS blocks unauthenticated inserts.

**Confirmation Module** — test the vote mechanic: inserting a vote, flipping a vote, retracting a vote, and that `gone_count` and `status` on the sighting update correctly at each threshold (3 and 5).

**Photo Upload Module** — test that a valid file produces a public URL and that oversized/wrong-type files are rejected before upload.

**Auth Module** — test session state transitions: logged out → magic link sent → logged in → display name prompt → profile created → logged out.

**Profile Module** — test that display name uniqueness is enforced case-insensitively, and that a missing profile gates map interactions.

**Map Module** — lower priority. Focus on callbacks: does `onMapClick` fire with correct lat/lng? Does `onPinClick` fire with the correct sighting id?

**Modal Module** — test form validation (280-char description limit, 500-char comment limit, date required) and that submission calls the correct module functions with the right payload.

## v1 Scope

The following features ship in v1. Everything else is deferred.

- Map with all sightings, shoe emoji pins, clustering
- Magic link auth with mandatory display name setup
- Create a sighting (pin, photo, description, date)
- Edit own sighting (description, date, photo — location is immutable)
- Soft-delete own sighting
- "Still there" / "Gone" voting with three status states
- Sighting detail modal with voting buttons

## Out of Scope

- Native mobile app (iOS / Android)
- Multiple photos per sighting
- Structured fields (shoe type, condition, brand)
- Offline / PWA support
- Self-hosted deployment (deferred — start with Vercel)
- Linked sightings / "Shoe Journey" chains (named future feature — design separately alongside profiles and geocaching)

## Deferred (architected for, not built in v1)

- **User profile pages** — schema in place (`profiles` table); public profile UI, avatar, and stats deferred
- **Comments** — schema in place (`comments` table, flat, 500-char limit, soft delete); UI deferred
- **Reactions** — confirmations table is the foundation; additional reaction types (beyond `still_there`/`gone`) deferred
- **Moderation tooling** — schema in place (`reports` table, `is_admin` on profiles); admin dashboard UI deferred
- **Geocaching / gamification** — streaks, leaderboards, "shoe of the month"; depends on profiles shipping first

## Further Notes

- The project is intentionally silly. UI copy and empty states should reflect that tone (e.g. "No shoes lost here... yet.", "This shoe has gone AWOL.").
- The app name is **"Where's My Shoe?"**.
- When migrating from Vercel to self-hosted later: only `.env` file location and deploy process change. No code changes required.
- The 280-char description limit is a deliberate nod to Twitter/X — a shoe sighting shouldn't need an essay.
- Status thresholds (3 for "Gone Missing?", 5 for "AWOL") are app-level constants and can be tuned without schema changes.
- "Shoe Journey" (linking sightings of the same shoe across locations) is a named future feature. Do not add schema for it until the full mechanic is designed — the right data shape depends on unresolved questions (chains vs. networks, who can link, map visualisation).
