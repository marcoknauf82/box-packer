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
relocation (container picks up Friday, May 29 2026; Marco departs June 29 2026). This
is not a hypothetical app — it is being actively used for a real move with a hard
deadline.
Data loss bugs before the container seals are serious.

---

## Current technical state

- Single HTML file (~146KB): `index.html` deployed at https://marcoknauf82.github.io/box-packer/
- PWA with manifest and service worker (installable on Android)
- No backend — all API calls go directly from browser to Anthropic, Google, and Supabase
- No user accounts yet — single-user, single-move hardcoded
- In-flight packing state survives app backgrounding via localStorage snapshot (v20.1)
- Canvas labels rendered at 2× supersampled resolution for thermal-print sharpness (v20.1.1)
- Photos compressed client-side to 1600px / 0.85q before upload (v20.2)
- QR codes on labels link directly to the app (`?box=N` triggers detail view) (v20.2.1)
- Drive folder URL persisted back to Supabase after upload so QR scans show photo links (v20.2.3)
- Canvas labels wrap summary + items to 2 lines with dynamic content budget; tested against all 23 real boxes (v20.2.4)
- OAuth expiry surfaces visibly in top bar + label screen; per-photo upload retry (3 attempts) with visible "N of N" counter (v20.2.5)
- 23 boxes packed in first real session (Holiday Home, May 9, 2026); session 2 (Hudson) started May 12
- Current version: **v20.2.5**

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
- `box_size` text — must be one of 'Extra Small' / 'Small' / 'Medium' / 'Large' / 'Extra Large' / 'Imperfect Box' / 'Oversized' (v20.2: expanded from 3 to 7 values)
- `dimensions` text — L×W×H string. Auto-populated for standard sizes from `BOX_SIZES` lookup; manually entered for XS/XL/Oversized (v20.2 new)
- `weight_kg` decimal(5,1) — optional (v20 new)
- `owners` text — comma-separated list of owners from OWNERS_LIST, e.g. "Marco, Geetha" (v20.1 new)
- `is_fragile` boolean
- `packed_by` text — "Marco" or "Geetha"
- `packed_at` date
- `est_value_cents` integer — ALWAYS stored as integer cents, never dollars
- `item_count` integer
- `drive_folder_url` text — populated by writeToSheet after Drive upload succeeds (v20.2.3)
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
- `owners` text — comma-separated list of owners (v20.1 new)

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
   `photos`, `items`, `weightKg`, `boxId`, `boxNumber`, `packedAt`, `owners`,
   etc.) are cleared every time. Note: `owners` is per-box (changes per container),
   not session (like home/room), because which family member's stuff is in a
   given box varies.

8. **`S.packedAt` is set once at confirm time** (ISO timestamp). Do not overwrite on
   re-render. It becomes the authoritative timestamp for that box.

9. **Session persistence via localStorage under `SESSION_KEY` (v20.1).** `saveSession()`
   is called from every `r()`. The saved snapshot is a whitelist of fields (not the
   full `S` spread) — `registry`, `detailBox`, `detailItems`, `editingItem`, and
   `originRoomsByHome` are deliberately excluded because they're re-fetched from
   Supabase on boot. `clearSession()` fires on: successful save, menu "Pack a Box",
   `nextBox`, and back-to-menu. Restore is only attempted when the saved screen is
   `"pack"` and step is 0–2 with meaningful progress.

10. **Owners values must come from OWNERS_LIST.** The UI enforces this via hardcoded
    buttons (Marco, Geetha, Arun, Maya). If you add owners programmatically, pull from
    OWNERS_LIST. Stored as comma-separated text, not an array, for simplicity.

11. **Box sizes must be one of seven values from `BOX_SIZES` (v20.2).** The DB check
    constraint allows: 'Extra Small', 'Small', 'Medium', 'Large', 'Extra Large',
    'Imperfect Box', 'Oversized'. The frontend `BOX_SIZES` constant is the source of
    truth — it stores the dimension string for each standard size (Small=17×11×12 in,
    Medium=22×13×15 in, Large=27×15×17 in, Imperfect Box=22×15×10.5 in). XS, XL, and
    Oversized have `needsPrompt: true` and require user-entered dimensions. When
    refactoring, do not split the size enum from this constant — they must stay in sync.

12. **QR codes link to the app, not Sheets (v20.2.1).** Format: `${APP_URL}?box=${id}`.
    Scanning the QR opens the PWA which reads `?box=N` from the URL on boot and calls
    `loadBoxDetail(N)` automatically. Labels printed BEFORE v20.2.1 point to the Google
    Sheet and won't scan-to-lookup correctly — they'll need re-printing if scan-to-lookup
    becomes critical (e.g. for damage documentation). For Marco's move, only test boxes
    were printed pre-v20.2.1, so this isn't a real concern.

13. **Skip-photos flow (v20.2.1).** Step 1 has a "Skip — enter manually" button that
    sets `S.step = 2` directly. Items array stays empty until user adds them via the
    review screen's "+" button. `value_source_type` defaults to "not_set" or "manual"
    depending on whether the user enters a value. This intentionally bypasses
    `analyze()` entirely — no AI call, no API cost, no Drive folder created (since
    there are no photos to upload). Box save still happens normally.

### Google Sheets (backup/export — secondary)

Sheet ID: `1PJWbuRemzJIMXFgyxln4256B6Z4Sy7nnyI7-g7gLtHw`

Three tabs:
- `Box Registry` — one row per box. v20.1 row shape (15 columns):
  Box ID, Tier, Tier Name, Short Summary, Description, Packed From, Destination,
  Size, Weight (kg), Belongs To, # Items, Fragile?, Date Packed, Inventory QR URL,
  Photos (Drive)
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

### Why multi-owner is per-box, not session (v20.1)
The packing session is a single human doing the work, so `packed_by` is session-level.
But which family member's stuff is in each box changes per container — one box might
be all Marco's work clothes, the next might be mixed kids' toys. So `owners` resets
on every new box, unlike `packedBy`.

### Why session persistence uses localStorage (not IndexedDB) — v20.1
IndexedDB is the "right" choice for photo-sized blobs, but it's async-only and
significantly more code. localStorage works synchronously, fits everything except the
biggest photo sets, and has a clean quota-exceeded fallback (drop photos, flag the
session so the restore toast tells the user). Phase 2 can move to IndexedDB when
migrating to React.

### Why owners is a comma-separated text field (not an array)
Supabase/Postgres supports text[] natively, and it would be the cleaner data model.
But the UI enforces values via hardcoded buttons, the write is always a join(","),
and the read is always a split(",") or raw display — so we'd be serializing to array
and back to string anyway. Text is simpler and keeps the schema homogeneous with the
other text fields. Phase 2 with accounts will revisit this when owners become
first-class user references.

### Why box sizes are categorical strings + a separate dimensions text field (v20.2)
EOS Form #1180 needs cubic measurement per box (cubic feet or cubic inches). Marco
buys boxes from Walmart in three standard sizes plus Imperfect Foods grocery boxes,
so 4 of his boxes have known dimensions and the remaining 3 generic sizes (XS, XL,
Oversized) get measured at packing time. The `BOX_SIZES` frontend constant maps
each name to its known dimension string OR flags `needsPrompt: true` to surface a
manual input. The `dimensions text` column on `boxes` stores whatever lands there —
either auto-filled from the lookup or typed by hand. We could have stored cubic
inches as a separate decimal column for cleaner aggregation, but Marco doesn't
actually need running cubic-foot totals during packing, only at PDF-export time —
so a text field is fine and we parse it later when needed.

### Why QR codes link to the app instead of Google Sheets (v20.2.1)
Original v19/v20 QR pointed to the Sheet URL with `&box=N` query params, which
Sheets ignores. Functionally useless beyond "opens the spreadsheet." Switching to
`APP_URL?box=N` lets the app intercept the param on boot and navigate to the box's
detail screen — photos, items, full metadata. This is the foundation for delivery
check-in (scan box on arrival in Aachen → app marks it received) and damage
documentation (scan damaged box → see original packing photos for comparison).
Trade-off: labels printed before v20.2.1 won't scan-to-lookup. Acceptable since
no production labels were printed pre-v20.2.1.

### Why skip-photos exists (v20.2.1)
Some boxes don't benefit from AI analysis — sealed boxes of paperwork, identical
restocks of consumables, items where the box label is the only identifier needed.
Forcing a photo upload + AI call adds time and cost without value in those cases.
Skip-photos jumps straight to manual review where the user types description and
items by hand. `value_source_type` lands as "not_set" rather than "ai_estimate"
which keeps audit trail honest about which boxes had AI involvement.

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

## Known issues (open as of v20.2.5)

**Network / offline (remaining v21 work)**

- No offline write queue for Supabase. If WiFi drops between `getNextBoxNumber` and `INSERT`, the user still sees a red "supa-fail" bar and has to retry. Data isn't corrupted — Supabase is source of truth — but state is awkward and there's no retry-on-reconnect.
- AI photo analysis fetch (to api.anthropic.com) is still single-attempt. Network blip → "Failed to fetch" toast → user must manually retry or skip-photos.
- OAuth refresh is reactive, not proactive. Token-expired flag flips only after a 401 actually comes back. A "5-minute warning before expiry" would require Google's OAuth2 client to expose the expiry timestamp, which it doesn't directly. Worst case under v20.2.5: one box's photos fail to upload before the badge turns red.

**Code hygiene**

- `getNextBoxNumber` + INSERT is not atomic. Race acceptable for single-user Phase 1
  (unique constraint catches it; user sees supa-fail on second submit).
- Sheet formatting may still race on first write — pre-existing from v19.
- Label PNG QR depends on api.qrserver.com availability — both in the preview canvas
  (#qr-cv) and in the rendered full label PNG. Same service as v19.

## Resolved (formerly known issues)

- ~~Orphan risk: if box insert succeeds but items insert fails, a box row exists with
  no items.~~ Resolved by adding `ON DELETE CASCADE` to the FK in May 2026 (also
  applied to `move_id` FKs on boxes / box_items / standalone_items). Cascade tested
  in production after a manual delete left orphan items behind.
- ~~`box_code` NOT NULL constraint from v19 silently survived the v20 migration~~,
  causing every save attempt to fail with a 400. Resolved by `alter column box_code
  drop not null`. Lesson: schema migrations can drop ALTER statements silently — verify
  with sanity-check SELECTs after every migration, not just by running the script.
- ~~Holiday Home button tap was a no-op (regex over-escape in template literal).~~
  Resolved in v20.2.2 (single-character fix on the render line).
- ~~Drive folder URL not persisted to Supabase~~ — QR scans showed box detail without
  a working photo link. Resolved in v20.2.3 by adding an UPDATE after Drive upload.
- ~~Two dead constants in script (`ITEM_COUNTER`, `counters`)~~ — removed in v20.2.4.
- ~~Label summary and item descriptions truncated with "..." on the printed label.~~
  Resolved in v20.2.4 with 2-line wrapping + dynamic content budget. Tested against
  all 23 real boxes — 0 truncations, 0 overflows, worst-case bottom margin 28px.
- ~~2-copy / 4-copy print buttons stack labels onto a single PNG that doesn't print
  correctly on Nelko thermal.~~ Resolved in v20.2.4 — collapsed to single "Print label"
  button. User sets copy count inside the Nelko app instead.
- ~~Drive upload failures logged only to console.warn.~~ Resolved in v20.2.4 — failures
  surface as a prominent red error bar on the success screen, same treatment as Sheet
  errors, with a link to Google Drive for manual verification.
- ~~"Connected" badge always green, even when token expired.~~ Resolved in v20.2.5 —
  badge turns red on 401, flag resets only on successful re-auth.
- ~~Per-photo Drive uploads can partially fail with no retry.~~ Resolved in v20.2.5 —
  each photo gets 3 attempts with exponential backoff; failed indices surface in a
  visible counter ("📷 Only 1 of 2 photos uploaded — missing: photo 2").

## Lessons logged for v21+

- **DOM IDs from user-facing strings need a generate-then-query test, not a code review.**
  The Holiday Home button bug (v20.2.1) had `id="oh-${h.replace(/\\s+/g,'-')}"` —
  double-escaped backslash inside a template literal. The renderer produced
  `oh-Holiday Home` (with a space); the bind handler queried `#oh-Holiday-Home` (with
  a hyphen). Both source lines looked correct in code review. Only a runtime test
  catches the mismatch. Add to the Phase-2 lint pass: any code building DOM IDs from
  template literals gets a unit test that round-trips a representative input.
- **Schema migrations need explicit verification, not trust.** Two production-blocking
  bugs in May 2026 (box_code NOT NULL, owners column missing in v20.1) both stemmed
  from migration ALTER statements that silently failed or were skipped. Future
  migrations include a "run this SELECT to confirm columns exist" verification step.
- **Layout math is not a substitute for testing against real data.** The original v20.2.4
  used pure-math budget reasoning (sample summary + 8 items, all sized to fit). When
  tested against the actual 23 boxes in the database with Liberation Sans (Arial-
  equivalent metrics), 6 boxes overflowed the bottom edge because long book titles
  wrap to 2 lines and stack up faster than expected. Replaced with a dynamic budget
  that pre-measures every item and stops adding when overflow is imminent. Pattern
  for future canvas/layout work: always run a measurement pass against real production
  rows before shipping.
- **Silent failures kill trust — surface them at the moment of failure, not later.** Drive errors going to `console.warn` only meant Marco packed for hours on May 9 unaware that photos hadn't uploaded. The same was true of OAuth token expiry — the "connected" badge stayed green long after the token died. v20.2.4 surfaced Drive errors with a red bar; v20.2.5 surfaced OAuth expiry with a red badge + label-screen banner + cleared token. Pattern for future work: any background operation whose failure would surprise the user later must surface visibly at the time of failure, AND provide a positive-feedback signal when it succeeds (e.g. v20.2.5's green "📷 N of N photos uploaded" bar — Marco wanted to know things were working, not just that they weren't broken).
- **Retry once, retry quickly, retry visibly.** v20.2.5's per-photo retry uses 3 attempts with 500ms / 1500ms backoff — total worst-case ~4 seconds per photo. Long enough to recover from a typical mobile WiFi blip, short enough that the user doesn't notice. The visible counter ("📷 3 of 4 uploaded") tells the user the outcome without exposing the retry mechanics. Apply this pattern to AI photo analysis and any other single-fetch network call in v21.

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
  (May 29)? If yes, it is critical path. If no, it can wait.

---

## The move timeline (for prioritisation)

- **Now → May 29 2026:** Intensive packing. Everything needed to document the move
  must exist. Critical: PDF export, offline queue, all packing features stable.
- **May 29 → June 29 2026:** Container shipped. Build arrival-day features:
  delivery check-in (QR scan), unpacking mode, damage documentation.
- **June 29 2026:** Marco and Geetha fly to Aachen.
- **~July/August 2026:** Container arrives in Aachen. Delivery check-in and unpacking
  mode used for real.
- **Phase 2 (July–October 2026):** Backend, user accounts, multi-device. Box Packer
  gets ~30% of available time; LiveYourMoney (financial planning app) takes priority.

---

*Last updated: May 12, 2026 (v20.2.5 — OAuth expiry visibility + per-photo upload
retry with visible counter, targeting the silent failure modes Marco hit on May 9.
Builds on v20.2.4 (canvas label wrapping with dynamic content budget, single print
button, visible Drive errors, dead code removed). Earlier May work: drive_folder_url
persistence (v20.2.3); Holiday Home button regex hotfix (v20.2.2); box sizes / image
compression / QR scan-to-lookup / skip-photos (v20.2.1)). Update this file whenever
a significant architectural decision is made, a known issue is resolved, or a
deferred item moves to active development.*
