# CLAUDE.md — Box Packer

> Read this file at the start of every session before writing any code.
> It explains what this app is, how it works, what decisions have been made and why,
> and what invariants must never be violated.

---

## What this app is

Box Packer is a moving inventory PWA. The user photographs boxes and standalone items,
Claude vision AI generates an itemized list with insurance replacement values, the user
reviews and edits, and the app writes the inventory to Supabase (source of truth) and
Google Sheets (secondary backup/export). Physical labels with QR codes are printed to a
**Nelko PL70e thermal printer** (4×6 portrait, monochrome, 203 DPI) via the Android
Web Share API.

The immediate use case is Marco and Geetha Knauf's Chicago-to-Aachen international
relocation (container ships ~mid-May 2026, departure June 29 2026). This is not a
hypothetical app — it is being actively used for a real move with a hard deadline.
Data loss bugs before the container seals are serious.

---

## Current technical state

- Single HTML file (~115KB): `index.html` deployed at https://marcoknauf82.github.io/box-packer/
- PWA with manifest and service worker (installable on Android)
- No backend — all API calls go directly from browser to Anthropic, Google, and Supabase
- No user accounts yet — single-user, single-move hardcoded
- Current version: **v20**

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

**boxes** (v20)
- `id` uuid (PK)
- `move_id` uuid (FK → moves)
- `box_number` integer — e.g. `14` — UNIQUE per move (v20: globally unique integer, replaces `box_code`)
- `box_code` text — LEGACY, kept nullable for rollback safety, not written by v20
- `tier` char(1) — must be 'A', 'B', 'C', 'D', or 'E' (enforced by DB check constraint)
- `description` text — longer narrative
- `short_summary` text — ≤5 words, prints bold on label (v20 new)
- `destination_room` text — optional, may be set later during unpacking
- `origin_home` text — "Chicago Home" or "Holiday Home" (v20 new)
- `origin_room` text — e.g. "Basement" (v20 new)
- `box_size` text — "Small"/"Medium"/"Large" (v20 new)
- `weight_kg` decimal(5,1) — optional (v20 new)
- `is_fragile` boolean
- `packed_by` text — "Marco" or "Geetha"
- `packed_at` date
- `est_value_cents` integer — ALWAYS stored as integer cents, never dollars
- `item_count` integer
- `drive_folder_url` text
- `qr_url` text

**box_items**
- Unchanged from v19.
- `id` uuid (PK), `box_id`, `move_id`, `item_number` (UNIQUE per box),
- `category`, `description`, `quantity`,
- `est_value_cents` integer, `value_source`, `value_source_type`.

**standalone_items** (v20)
- `id` uuid (PK)
- `move_id` uuid (FK → moves)
- `item_code` text — e.g. "I-01" — UNIQUE per move (kept as text format, visually distinct from boxes)
- `tier` char(1) — must be 'A', 'B', 'C', 'D', or 'E'
- Same value/source fields as box_items
- `dimensions` text
- `short_summary`, `origin_home`, `origin_room`, `weight_kg` (v20 new)

**origin_rooms** (v20 new)
- `id` uuid (PK)
- `move_id` uuid (FK → moves)
- `origin_home` text — "Chicago Home" or "Holiday Home"
- `room_name` text
- UNIQUE (move_id, origin_home, room_name)
- Seeded via SQL migration with starter rooms per home; extensible via UI.

### Critical invariants — never violate these

1. **Values are always integer cents.** $45.00 is stored as 4500. Never store 45.00 or "45".
   Divide by 100 only at display time.

2. **Box numbers are unique per move, globally (v20).** Boxes are numbered 1, 2, 3…
   regardless of tier. The DB has `unique (move_id, box_number)`.
   `getNextBoxNumber()` queries Supabase for `MAX(box_number)` before assigning the next.
   This is async — it must await the DB query. Prior v19 format (`A-14`) is retired but
   the `box_code` column remains nullable for rollback safety.

3. **Standalone items keep `I-NN` format.** `item_code` is still text-formatted with
   zero-padded sequence. This distinguishes standalone items from boxes visually
   ("I-03" vs "3") and in the Box Registry.

4. **Tier must be A, B, C, D, or E (v20).** The DB check constraint will reject
   anything else. Marco's specific categories:
   - **A**: Unpack on arrival (toiletries, bedding, basic kitchen, kids' essentials)
   - **B**: Unpack first months (rest of clothes, books, hobby items)
   - **C**: Post-rental 2027+ (owned furniture, kitchenware for unfurnished home)
   - **D**: Long-term storage (memorabilia, photos, sentimental keepsakes)
   - **E**: Karting — early 2027 (karting gear, tools, suits, helmets)
   Note: these labels are Marco's specific categories. The Phase 2 design will make
   tiers user-definable per move — see the deferred section below.

5. **value_source_type must be exactly 'ai_estimate', 'manual', or 'not_set'.** The DB
   check constraint enforces this. Do not add new values without updating the constraint.

6. **All Supabase writes must be awaited and error-checked.** A failed write that isn't
   caught means the user thinks data was saved when it wasn't. `saveBoxToSupabase`
   returns a boolean; `confirmBox` surfaces failures as `S.sheetResult = "supa-fail"`
   with a red error bar on the label screen.

7. **Every entry point to the packing flow must reset all PER-BOX packing state.**
   In v20, SESSION-level fields (`tier`, `originHome`, `originRoom`, `boxSize`,
   `packedBy`, `mode`) are INTENTIONALLY preserved across `nextBox()` and the menu
   "Pack a Box" button. Per-box fields (`desc`, `shortSummary`, `dest`, `fragile`,
   `photos`, `items`, `weightKg`, `boxId`, `boxNumber`, `packedAt`, etc.) are cleared
   every time.

8. **`S.packedAt` is set once at confirm time** (ISO timestamp). Do not overwrite on
   re-render. It becomes the authoritative timestamp for that box.

### Google Sheets (backup/export — secondary)

Sheet ID: `1PJWbuRemzJIMXFgyxln4256B6Z4Sy7nnyI7-g7gLtHw`

Three tabs:
- `Box Registry` — one row per box. v20 row shape (14 columns):
  Box ID, Tier, Tier Name, Short Summary, Description, Packed From, Destination,
  Size, Weight (kg), # Items, Fragile?, Date Packed, Inventory QR URL, Photos (Drive)
- `Item Detail` — one row per item within boxes
- `Item Registry` — one row per standalone item

Sheets writes are secondary to Supabase. If a Sheets write fails, log it and show a
non-fatal warning — do not block the flow or lose Supabase data. Supabase is the
source of truth. Will be deprecated in v21 in favor of on-demand CSV export.

---

## Label design (v20)

The printed label is 4" × 6" portrait, rendered on canvas at 203 DPI (812 × 1218 px),
pure black-and-white (the Nelko PL70e is monochrome thermal). The design is laid out
top-to-bottom as:

1. **Header row**: Large box number (integer, e.g. "14") on the left; tier badge on
   the right — a solid black rectangle containing the tier letter (A–E) in huge type
   with a two-line action-oriented subtitle underneath (e.g. "UNPACK ON / ARRIVAL").
2. **Short summary**: 5-word centered bold line (e.g. "Winter clothes and boots").
3. **Packed-from band**: Solid black horizontal band, white text, home · room
   in ALL CAPS (e.g. "CHICAGO HOME · BASEMENT").
4. **Fragile stripe** (conditional): Bordered box with hatched flanks and centered
   "FRAGILE" + "HANDLE WITH CARE" text. Caution-triangle icons in both flanks.
5. **Contents**: Two-column list of up to 8 items, with "+N more items" overflow.
6. **Size/weight row**: Box size on the left, weight (kg) on the right.
7. **QR + packed-by footer**: QR code (150px) on the left linking to the Supabase
   row; packed-by name + timestamp (e.g. "Apr 19, 2026 · 14:32") on the right.
8. **Deliver-to footer**: Small-print recipient address for lost-box recovery.
   Hardcoded in DELIVER_TO constant: Marco Knauf, Am Beverbach 2, 52066 Aachen.

The canvas renderer is a clean rewrite — do not try to surgically patch it.
If layout needs to change, regenerate `drawLabelOnCanvas` from the spec above.

---

## Key architectural decisions and why

### Why values are stored as integer cents
Floating point arithmetic is unreliable for currency. `0.1 + 0.2` in JavaScript gives
`0.30000000000000004`. Storing 4500 instead of 45.00 means all arithmetic is integer
arithmetic, which is exact. Divide by 100 only when displaying to the user.

### Why boxes are now numbered globally (not A-14 style) — v20
User feedback: "inverting the order to 14-A makes it clear each number is only assigned
to one box and we can sort by box number." Taken further to drop the letter entirely,
since the tier badge on the label shows the letter prominently anyway. The integer is
stored as `box_number` in Supabase; the display ID is `String(box_number)`. Reclassifying
a box to a different tier no longer changes its number.

### Why the move ID is hardcoded
Multi-move support (Phase 2) requires user accounts and auth, which haven't been built yet.
The hardcoded constant is explicitly labelled as a known limitation.

### Why the Supabase key is in client-side code
This is a known security shortcut acceptable for the dogfooding phase with RLS enabled.
The key is the Supabase "publishable" (anon) key — it is not a secret in the same way
a server API key is. RLS policies limit what it can do. Moving it to a backend proxy
is a Phase 2 task, not Phase 1.

### Why Google Drive is organised as `Aachen Move / [Box ID] / photos`
This structure makes photos findable by box code without querying the app. If the app
were unavailable, Marco could still find the photos for box 15 by navigating the Drive
folder structure directly.

### Why Sheets write is kept alongside Supabase
Google Sheets provides a human-readable, instantly shareable view of the inventory that
doesn't require opening the app. Useful for insurance companies, customs brokers, and
family members helping unpack. It is not a database replacement — it is an export format.
Will be retired in v21 in favor of on-demand CSV extract.

### Why origin_home and origin_room are in the schema
User packs from two physical locations (Chicago Home + Holiday Home in Hudson, IL).
Even though the container ships from Hudson only, knowing which home/room a box came
from is invaluable when unpacking in Aachen and checking "wait, was this the item from
the Chicago basement or the Hudson guest room?"

### Why origin_rooms is a separate table
Predefined + extensible per home. Seeded via SQL migration. Allows UI chip-style picker
that remembers rooms per home. User can add new rooms which persist to the table.

### Why the label is monochrome portrait at 203 DPI
Nelko PL70e is a thermal direct printer — it can only print black. 4×6 portrait is its
preferred paper size (the same as UPS standard shipping labels). 203 DPI is its native
resolution. Matching these exactly avoids any scaling artifacts.

### Why "PACKED FROM" label was removed but address retained
User preference: the prominent black band should show origin visibly, the word "PACKED
FROM" is redundant context since the address format is already unambiguous. The full
deliver-to address is kept small-print at the bottom specifically for lost-box recovery.

---

## What has been deliberately deferred (do not build yet)

- **User accounts / auth** — Phase 2.
- **Multi-move support** — Phase 2. Move ID is hardcoded.
- **User-defined tiers** — Phase 2. Currently hardcoded to A–E with Marco's labels.
  Phase 2 design should introduce a `tiers` table per move.
- **Offline queue for failed writes** — Phase 1 (April priority), not yet built.
- **Anthropic API key on backend** — Phase 2.
- **React migration** — Phase 2.
- **Server-side box_number generation** — Phase 2. Currently a read-then-insert which
  is vulnerable to race conditions if two clients pack at the same time (DB unique
  constraint catches it, but user sees an error).
- **PDF export matching EOS Form #1180** — Phase 1 (v21, critical for container pickup).
- **CSV extract** — Phase 1 (v21). Replaces live Sheets write.
- **Delivery check-in / unpacking mode** — Phase 1 (post-container-ship window).

---

## Known issues (open as of v20)

- Two dead constants in script: `ITEM_COUNTER = {}` and `counters = {A:0,B:0,C:1}`.
  Unused; leaving to avoid needless edits. Remove in v21 cleanup.
- `getNextBoxNumber` + INSERT is not atomic. Race acceptable for single-user Phase 1
  (Unique constraint catches it; user sees supa-fail on second submit).
- Orphan risk: if box insert succeeds but items insert fails, a box row exists with no
  items. Pre-existing from v19. User sees error and can manually resolve in Supabase UI.
- Sheet formatting may still race on first write — pre-existing from v19.
- Google OAuth re-consent occasionally required after scope upgrade.
- Label PNG QR depends on api.qrserver.com availability — both in the preview canvas
  (#qr-cv) and in the rendered full label PNG. Same service as v19.

## Visual theme (as of v20)

App chrome unchanged from v19:
- Font: Inter (Google Fonts)
- Background: `#F7F7F5` warm off-white; cards `#FFFFFF` with `1px #E8E7E2` border
- Primary / CTA: `#0D0E12` near-black
- Accent: `#F5D800` yellow — now only on the "Print to Nelko" section button
- Tier picker: monochrome, selected state uses near-black (`--text`) background
- Label preview: pure B&W matching the thermal output

---

## Development approach

- Marco reviews, tests, and makes all product decisions. Claude writes first drafts.
- Test on Android phone immediately after any UI change — desktop Chrome is not a
  reliable proxy.
- Prefer changes testable on device over elaborate local dev setups.
- If a module gets messy after several patches, regenerate it from a clean spec.
  (The v20 canvas renderer is a clean rewrite from v19's smaller 4×2.4 version.)
- Flag irreversible decisions explicitly before implementing them.
- Monetary values: always integer cents in the DB, always divide by 100 for display.
- Before adding a feature, ask: does this need to work before the container ships
  (~mid-May)? If yes, it is critical path. If no, it can wait.

---

## The move timeline (for prioritisation)

- **Now → mid-May 2026:** Intensive packing. Everything needed to document the move
  must exist. Critical: PDF export, offline queue, all packing features stable.
- **Mid-May → June 29 2026:** Container shipped. Build arrival-day features:
  delivery check-in (QR scan), unpacking mode, damage documentation.
- **June 29 2026:** Marco and Geetha fly to Aachen.
- **~July/August 2026:** Container arrives in Aachen. Delivery check-in and unpacking
  mode used for real.
- **Phase 2 (July–October 2026):** Backend, user accounts, multi-device. Box Packer
  gets ~30% of available time; LiveYourMoney (financial planning app) takes priority.

---

*Last updated: April 19, 2026 (v20 — Nelko label rewrite, 5 tiers, new fields, integer
box_number). Update this file whenever a significant architectural decision is made,
a known issue is resolved, or a deferred item moves to active development.*
