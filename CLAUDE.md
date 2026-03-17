# CLAUDE.md — Box Packer

> Read this file at the start of every session before writing any code.
> It explains what this app is, how it works, what decisions have been made and why,
> and what invariants must never be violated.

---

## What this app is

Box Packer is a moving inventory PWA. The user photographs boxes and standalone items,
Claude vision AI generates an itemized list with insurance replacement values, the user
reviews and edits, and the app writes the inventory to Supabase (source of truth) and
Google Sheets (backup/export). Physical labels with QR codes are printed to a Brother
QL label printer via Android Web Share API.

The immediate use case is Marco and Geetha Knauf's Chicago-to-Aachen international
relocation (container ships ~mid-May 2026, departure June 29 2026). This is not a
hypothetical app — it is being actively used for a real move with a hard deadline.
Data loss bugs before the container seals are serious.

---

## Current technical state

- Single HTML file (~82KB): `index.html` deployed at https://marcoknauf82.github.io/box-packer/
- PWA with manifest and service worker (installable on Android)
- No backend — all API calls go directly from browser to Anthropic, Google, and Supabase
- No user accounts yet — single-user, single-move hardcoded
- Current version: v18

---

## The data model — this is the most important section

Everything downstream depends on getting this right. Never change column names or types
without checking all the places they are read and written.

### Supabase tables (source of truth)

**moves**
- `id` uuid (PK)
- `name` text — e.g. "Chicago → Aachen 2026"
- `departure_date` date
- Move ID is hardcoded as a constant in the app. Single move only until Phase 2.

**boxes**
- `id` uuid (PK)
- `move_id` uuid (FK → moves)
- `box_code` text — e.g. "A-01", "B-03" — UNIQUE per move
- `tier` char(1) — must be 'A', 'B', or 'C' (enforced by DB check constraint)
- `description` text
- `destination_room` text
- `is_fragile` boolean
- `packed_by` text — "Marco" or "Geetha"
- `packed_at` date
- `est_value_cents` integer — ALWAYS stored as integer cents, never dollars
- `item_count` integer
- `drive_folder_url` text
- `qr_url` text

**box_items**
- `id` uuid (PK)
- `box_id` uuid (FK → boxes)
- `move_id` uuid (FK → moves)
- `item_number` integer — UNIQUE per box
- `category` text
- `description` text
- `quantity` integer
- `est_value_cents` integer — ALWAYS integer cents
- `value_source` text — human-readable explanation, e.g. "Current Amazon price"
- `value_source_type` text — must be 'ai_estimate', 'manual', or 'not_set'

**standalone_items**
- `id` uuid (PK)
- `move_id` uuid (FK → moves)
- `item_code` text — e.g. "I-01" — UNIQUE per move
- `tier` char(1) — must be 'A', 'B', or 'C'
- Same value/source fields as box_items
- `dimensions` text

### Critical invariants — never violate these

1. **Values are always integer cents.** $45.00 is stored as 4500. Never store 45.00 or "45".
   Divide by 100 only at display time.

2. **Box codes are unique per move.** The DB has a `unique (move_id, box_code)` constraint.
   Box code generation must query Supabase for the current maximum before assigning the next one.
   This is why `getNextBoxCode()` is async — it must await the DB query.

3. **Tier must be A, B, or C.** The DB check constraint will reject anything else.
   - Tier A: daily life — unpack on arrival in Aachen
   - Tier B: karting gear — unpack early 2027
   - Tier C: long-term storage — needed post-2027
   - Note: these labels are Marco's specific categories. The Phase 2 design will make
     tiers user-definable per move — see the deferred section below.

4. **value_source_type must be exactly 'ai_estimate', 'manual', or 'not_set'.** The DB check
   constraint enforces this. Do not add new values without updating the constraint.

5. **All Supabase writes must be awaited and error-checked.** A failed write that isn't caught
   means the user thinks data was saved when it wasn't. Always surface failures visibly.

### Google Sheets (backup/export — secondary)

Sheet ID: `1PJWbuRemzJIMXFgyxln4256B6Z4Sy7nnyI7-g7gLtHw`

Three tabs:
- `Box Registry` — one row per box
- `Item Detail` — one row per item within boxes
- `Item Registry` — one row per standalone item

Sheets writes are secondary to Supabase. If a Sheets write fails, log it and show a
non-fatal warning — do not block the flow or lose Supabase data. Supabase is the
source of truth.

---

## Key architectural decisions and why

### Why values are stored as integer cents
Floating point arithmetic is unreliable for currency. `0.1 + 0.2` in JavaScript gives
`0.30000000000000004`. Storing 4500 instead of 45.00 means all arithmetic is integer
arithmetic, which is exact. Divide by 100 only when displaying to the user.

### Why the move ID is hardcoded
Multi-move support (Phase 2) requires user accounts and auth, which haven't been built yet.
The hardcoded constant is explicitly labelled as a known limitation. Do not try to work
around it before Phase 2 — it would create complexity without the auth layer to support it.

### Why the Supabase key is in client-side code
This is a known security shortcut acceptable for the dogfooding phase with RLS enabled.
The key is the Supabase "publishable" (anon) key — it is not a secret in the same way
a server API key is. RLS policies limit what it can do. Moving it to a backend proxy
is a Phase 2 task, not Phase 1.

### Why Google Drive is organised as `Aachen Move / [Box ID] / photos`
This structure makes photos findable by box code without querying the app. If the app
were unavailable, Marco could still find the photos for "A-15" by navigating the Drive
folder structure directly.

### Why Sheets write is kept alongside Supabase
Google Sheets provides a human-readable, instantly shareable view of the inventory that
doesn't require opening the app. Useful for insurance companies, customs brokers, and
family members helping unpack. It is not a database replacement — it is an export format.

### Why box code generation queries Supabase rather than incrementing a local counter
Local counters reset on browser refresh. Supabase is persistent. The `getNextBoxCode()`
function queries `SELECT MAX(box_code) FROM boxes WHERE move_id = X AND tier = Y` to
find the current highest code, then increments. This is why the numbering survives
session resets.

### Why labels are 4×2.4 inches
This matches the Brother DK-11204 and DK-11209 label rolls Marco owns. The canvas
renderer targets 174 DPI at this size. Do not change dimensions without checking
physical label size compatibility.

---

## What has been deliberately deferred (do not build yet)

These are known gaps that will be addressed in Phase 2 or later. Do not attempt to
solve them now — the complexity is not justified at this stage:

- **User accounts / auth** — Phase 2. Single user only right now.
- **Multi-move support** — Phase 2. Move ID is hardcoded.
- **User-defined tiers** — Phase 2. Currently hardcoded to A/B/C with Marco's labels
  (daily life / karting / storage). Other users will want different categories. The Phase 2
  design should introduce a `tiers` table per move:
  ```sql
  create table tiers (
    id uuid default gen_random_uuid() primary key,
    move_id uuid not null references moves(id),
    code char(1) not null,        -- 'A', 'B', 'C' or user-defined
    label text not null,          -- e.g. "Daily life", "Karting gear", "Storage"
    description text,
    unpack_priority integer,      -- 1 = first to unpack
    color text,                   -- for UI color coding
    unique (move_id, code)
  );
  ```
  The `tier` check constraint on `boxes` and `standalone_items` would be replaced with
  a FK reference to `tiers`. Requires user accounts to be in place first (whose tiers
  are these?), so genuinely a Phase 2 dependency.
- **Offline queue for failed writes** — Phase 1 (April priority), but not yet built.
  Currently: if you're offline, writes fail visibly.
- **Anthropic API key on backend** — Phase 2. Currently in localStorage (client-side).
- **React migration** — Phase 2. Currently vanilla JS in a single HTML file.
- **PDF export** — Phase 1 (critical, not yet built).
- **Delivery check-in / unpacking mode** — Phase 1 (post-container-ship window).

---

## Known issues (open as of v18)

- Sheet formatting inconsistent on first box write — race condition in batchUpdate calls
- Session registry lost on browser refresh (numbering correct via Supabase, but UI resets)
- Label PNG QR depends on api.qrserver.com availability
- Google OAuth re-consent occasionally required after scope upgrade
- Move ID hardcoded to single move

---

## Development approach

- Marco reviews, tests, and makes all product decisions. Claude writes first drafts.
- Test on Android phone immediately after any UI change — desktop Chrome is not a reliable proxy.
- Prefer changes testable on device over elaborate local dev setups.
- If a module gets messy after several patches, regenerate it from a clean spec.
- Flag irreversible decisions explicitly before implementing them.
- Monetary values: always integer cents in the DB, always divide by 100 for display.
- Before adding a feature, ask: does this need to work before the container ships (~mid-May)?
  If yes, it is critical path. If no, it can wait.

---

## The move timeline (for prioritisation)

- **Now → mid-May 2026:** Intensive packing. Everything needed to document the move must exist.
  Critical: PDF export, offline queue, all packing features stable.
- **Mid-May → June 29 2026:** Container shipped. Build arrival-day features:
  delivery check-in (QR scan), unpacking mode, damage documentation.
- **June 29 2026:** Marco and Geetha fly to Aachen.
- **~July/August 2026:** Container arrives in Aachen. Delivery check-in and unpacking
  mode used for real.
- **Phase 2 (July–October 2026):** Backend, user accounts, multi-device. Box Packer
  gets ~30% of available time; LiveYourMoney (financial planning app) takes priority.

---

*Last updated: March 17, 2026. Update this file whenever a significant architectural decision
is made, a known issue is resolved, or a deferred item moves to active development.*
