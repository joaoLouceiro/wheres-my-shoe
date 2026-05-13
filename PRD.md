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
11. As a registered user, I want to click anywhere on the map to place a new pin, so that I can report a shoe sighting at the exact location.
12. As a registered user, I want a modal to open after placing a pin, so that I can fill in the sighting details before saving.
13. As a registered user, I want the date field in the new sighting form to auto-fill to today, so that I don't have to type it manually.
14. As a registered user, I want to adjust the date in the form, so that I can report a sighting I saw earlier.
15. As a registered user, I want to optionally upload one photo of the shoe, so that others can see what it looked like.
16. As a registered user, I want to optionally add a short description (up to 280 characters), so that I can share context or humor about the sighting.
17. As a registered user, I want to confirm or cancel before saving a pin, so that I don't accidentally publish a sighting.
18. As a registered user, I want to see my new pin appear on the map immediately after saving, so that I get instant feedback that it was submitted.
19. As a registered user, I want to remain logged in across sessions, so that I don't have to re-authenticate every time I visit.
20. As a registered user, I want to log out, so that I can end my session on a shared device.

## Implementation Decisions

### Modules

**Map Module**
The core interactive map rendered with Leaflet and `react-leaflet`. Responsible for displaying tiles (OpenStreetMap), rendering all shoe pins with a shoe emoji icon, handling marker clustering via `leaflet.markercluster`, and capturing click events on the map canvas to initiate new sightings. This module has a simple interface: it receives a list of sightings and two callbacks (onPinClick, onMapClick) and renders accordingly.

**Sighting Module**
Encapsulates all data operations for shoe sightings: fetching all sightings, creating a new sighting, and reading a single sighting. Backed by a Supabase Postgres table. This is the deepest module — it owns the data schema and is the single source of truth for sighting state. Interface: `getSightings()`, `getSighting(id)`, `createSighting(data)`.

**Photo Upload Module**
Handles uploading a photo to Supabase Storage and returning a public URL. Decoupled from the sighting module — the sighting record stores only the URL. Interface: `uploadPhoto(file) -> url`.

**Auth Module**
Wraps Supabase Auth with magic link flow. Exposes current session state and `signIn(email)` / `signOut()` actions. Used by the navbar and to gate the map click handler.

**Modal Module**
Two modal variants sharing a common shell:
- *Sighting Detail Modal* — read-only view of a pin's photo, description, and date. Opened by clicking an existing pin.
- *New Sighting Modal* — form with date (auto-filled), optional photo upload, optional description. Opened after placing a new pin. Handles validation (280-char limit) and submission.

### Data Schema

**sightings table**
- `id` — uuid, primary key
- `lat` — float, required
- `lng` — float, required
- `description` — text, nullable, max 280 chars
- `photo_url` — text, nullable
- `spotted_at` — date, required (user-adjustable, defaults to today)
- `created_by` — uuid, foreign key to auth.users
- `created_at` — timestamp

### Auth & Access Control
- Supabase Row Level Security (RLS): all users can SELECT sightings, only authenticated users can INSERT.
- Photo storage bucket: public read, authenticated write.
- Magic link is the only auth method — no password, no OAuth.

### Deployment
- Hosted on Vercel (Hobby plan). Environment variables managed via Vercel dashboard.
- Supabase project on free tier.
- No Vercel-specific features used — zero lock-in for future self-hosting migration.

### Monetisation
- Ko-fi (or equivalent) button/link in the footer. No backend involvement — external link only.

## Testing Decisions

Good tests for this project verify external behavior through public interfaces, not implementation details. They should not depend on component internals, Supabase client internals, or specific CSS classes.

**Sighting Module** — highest priority for tests. Pure data logic, no UI. Tests should cover: fetching returns correct shape, creating a sighting persists correctly, RLS blocks unauthenticated inserts. Use Supabase's local dev stack or a test database.

**Photo Upload Module** — test that a valid file produces a public URL and that oversized/wrong-type files are rejected before upload.

**Auth Module** — test session state transitions: logged out → magic link sent → logged in → logged out.

**Map Module** — lower priority. The map renders a third-party library; testing rendering is low value. Focus on the callbacks: does `onMapClick` fire with correct lat/lng? Does `onPinClick` fire with the correct sighting id?

**Modal Module** — test form validation (280-char limit enforced, date required) and that submission calls `createSighting` with the correct payload.

## Out of Scope

- Native mobile app (iOS / Android)
- Multiple photos per sighting
- Editing or deleting sightings after submission
- Comments or reactions on sightings
- Structured fields (shoe type, condition, brand)
- User profiles or public attribution of sightings
- Offline / PWA support
- Self-hosted deployment (deferred — start with Vercel)
- Any form of moderation tooling

## Further Notes

- The project is intentionally silly. UI copy and empty states should reflect that tone (e.g. "No shoes lost here... yet.").
- The app name is **"Where's My Shoe?"**.
- When migrating from Vercel to self-hosted later: only `.env` file location and deploy process change. No code changes required.
- The 280-char description limit is a deliberate nod to Twitter/X — a shoe sighting shouldn't need an essay.
