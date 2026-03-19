# Box Packer — Changelog

> Moving inventory app · AI-powered · PWA on GitHub Pages
> Repo: https://github.com/marcoknauf82/box-packer

---

## How to update this file

After each development session, add a new entry at the top of the version list below.
Format: `## v[N] — [Month YYYY]` followed by bullet points.
Keep entries concise — one line per meaningful change.
Mark open bugs under **Known issues** at the bottom.

---

## v19 — March 2026

**Visual restyle — new light theme with Inter font and yellow/black accent**

- CSS-only restyle — zero JavaScript changes, all logic identical to v18b
- Background changed from purple-tinted `#f5f4fc` to warm off-white `#F7F7F5`; cards are white with `1px` neutral border
- Font switched from system `-apple-system` stack to Inter (loaded from Google Fonts)
- Primary colour changed from purple `#534AB7` to near-black `#0D0E12`; yellow `#F5D800` is the single accent
- Tier A badge and selected states use yellow fill; Tier B mid-gray; Tier C light gray
- All interactive selected states (Box/Item toggle, Packed By, category grid, tier buttons) use yellow highlight instead of purple
- Step number circles changed to rounded squares (`border-radius: 8px`)
- Registry dots changed to rounded squares matching the boxier aesthetic
- Spinner colours updated from purple to dark/neutral
- Back links changed from purple to muted gray with dark hover
- Label preview and canvas label renderer unchanged — physical label output identical
- Inspired by logistics app reference image (dark charcoal + electric yellow); adapted to light background variant

---

## v18b — March 2026

**Bug fix: stale state carried between boxes when starting from menu**

- Fixed: tapping "Pack a Box" from the main menu did not reset packing state from the previous box
- Root cause: the menu handler only set `S.screen` and `S.step` but left `S.desc`, `S.dest`, `S.tier`, `S.photos`, `S.items`, and all other packing fields untouched
- Effect: the AI description guard (`if(parsed.desc && !S.desc)`) saw the old description as still present and silently discarded the new one, causing the previous box's summary to appear on the next box's record
- Fix: menu "Pack a Box" handler now resets all packing state fields, matching the `nextBox()` reset logic
- Note: any boxes already saved with wrong descriptions need manual correction in Supabase
- Also produced: `box_packer_restyled.html` — CSS-only restyle with Workiva-inspired enterprise aesthetic (dark navy `#1B2A4A`, blue accent `#2E6BE6`, DM Sans font). Exploratory only, not deployed.

---

## v18 — March 2026

**Main menu navigation + Box Registry browser with detail view**

- App now opens to a main menu with three options: Pack a Box, Box Registry, Documents
- Menu shows 4 most recent boxes at the bottom with tier dots and counts
- Pack a Box navigates to the existing packing flow with "← Menu" back link on step 0
- Box Registry screen lists all boxes and standalone items loaded from Supabase with tier counts
- Tapping a box row opens a detail view — queries Supabase for full box record and all items
- Detail view shows: box code, tier badge, fragile flag, description, destination, packed by, date, total value
- Detail view items list: item number, description, category, individual value, and total at bottom
- Detail view photos section: links to Google Drive folder (opens in new tab) if drive URL exists
- Reprint label button on detail view — loads box data into packing state and jumps to label step (step 4)
- Label step "Menu" button is context-aware — returns to detail view if coming from reprint, otherwise returns to menu
- Documents screen is a placeholder (PDF inventory and customs forms coming later)
- Registry row items show chevron icon hinting they are tappable
- `S.screen` state property added to route between menu, pack, registry, registry-detail, and documents screens
- `loadBoxDetail()` async function fetches box + items from Supabase, handles both boxes and standalone items
- `reprintFromDetail()` maps detail data back into packing state for label generation
- CSS added for menu cards, back links, and menu icon containers

---

## v17 — March 2026

**Supabase database integration — persistent box numbering**

- Supabase project created (Postgres database, free tier, US region)
- Database schema: `moves`, `boxes`, `box_items`, `standalone_items` tables with foreign keys and unique constraints
- `unique (move_id, box_code)` constraint prevents duplicate box codes — fixes session reset numbering bug
- Supabase JS library loaded from CDN (`@supabase/supabase-js@2`)
- Supabase client initialized with publishable key and project URL
- `getNextBoxCode(tier)` queries database for highest existing code per tier, returns next available (e.g. A-01 exists → A-02)
- `getNextItemCode()` same logic for standalone items (I-01, I-02 etc.)
- `saveBoxToSupabase()` writes box + all items to database on confirm — box first (to get UUID), then items linked via foreign key
- `confirmBox()` now async — awaits database for next code before proceeding
- Dual-write: data saves to both Supabase (source of truth) and Google Sheets (backup/export) on every confirm
- Row Level Security (RLS) enabled on all tables with permissive policy for pre-authentication phase
- Move record created: "Chicago → Aachen 2026" with departure date 2026-06-29
- Values stored as integer cents (e.g. 4500 = $45.00) to avoid floating point precision issues

---

## v16 — March 2026

**UX improvements — packed by, photo lightbox, edit modal**

- Added Packed By selector (Marco / Geetha) — persists across boxes in session, written to sheet and shown on label
- Added photo thumbnail strip on review screen — scrollable row of thumbnails with magnify icon overlay
- Added fullscreen photo lightbox — tap thumbnail to open, prev/next navigation, metadata banner (date/time, box ID, packed by)
- Replaced direct contenteditable item editing with pencil button — items no longer editable by tapping text directly
- Improved edit modal — item name, category grid (12 options), custom category input, est. value field, value source field
- Manual value edits now auto-set value source to "Manual input"
- Item labels use same 4×2.4 inch format as box labels with larger ID, category, dimensions, and value displayed
- Added PWA manifest and service worker — app installable on Android home screen from Chrome

---

## v15 — March 2026

**Standalone items, insurance values, value source attribution**

- Added Box / Item mode toggle on step 1 — switch between packing boxes and logging standalone bulky items
- Item IDs: I-01, I-02 etc. — parallel ID sequence to boxes
- New Item Registry tab auto-created in Google Sheet on first standalone item write
- Item review screen: description, category grid (10 options), destination, est. value, dimensions, fragile checkbox
- AI now returns estimated USD replacement value per item in box analysis
- AI now returns value source explanation per item (e.g. "Current Amazon price for comparable item")
- Value and value source displayed on review screen next to each item
- Est. Value (USD) column added to Box Registry (box total) and Item Detail (per item)
- Value Source column added to Item Detail and Item Registry tabs

---

## v14 — March 2026

**Google Drive photo upload + sheet formatting fixes**

- Photos now upload to Google Drive: creates `Aachen Move / [Box ID] /` folder structure automatically
- Photos (Drive) column in Box Registry with clickable `=HYPERLINK(...)` formula
- Drive folder URL shown in success status bar after box save, with direct link
- Drive scope changed from `drive.file` to `drive` for full folder creation access
- Sheet formatting error fixed: Sheets API requires camelCase field names (`foregroundColor` not `foreground_color`)
- Header row index corrected to account for title row above column headers
- `ensureHeaders()` now checks rows 1 and 2 for existing header to prevent duplicate header creation
- Drive upload errors shown in UI (non-fatal — data still saves to sheet if Drive fails)

---

## v13 — March 2026

**Print to Brother QL + canvas label rendering**

- Print to Brother QL via Android Web Share API — generates PNG and shares to Brother iPrint&Label app
- 1 copy, 2 copies, 4 copies options — 2-copy stacks vertically, 4-copy renders 2×2 grid on single PNG
- Canvas-based label renderer: tier stripe, box ID, items, destination, QR code at 174 dpi (4×2.4 inch)
- roundRect polyfill added for older Android Chrome versions
- Download label HTML button retained alongside print buttons

---

## v12 — March 2026

**Native Google Sheet migration + sheet write reliability**

- Migrated to native Google Sheet (created at sheets.new) — resolves FAILED_PRECONDITION 400 error from xlsx-converted sheet
- Sheet metadata pre-check before append — shows actual tab names found if mismatch, not generic error
- sheetsAppend now uses quoted tab name format (`'Box Registry'!A1`) for robustness
- Drive and Sheets write errors now non-fatal — data saves even if formatting step fails
- formatHeaderRow row index corrected — targets row 1 (header) not row 0 (title)
- formatNewRows uses correct camelCase color fields throughout

---

## v11 — March 2026

**Google Sheets formatting + column widths**

- Sheets API batchUpdate formatting fixed — all field names corrected to camelCase
- Header rows formatted: dark background (#1E1E2E), white bold text, 28px height
- Data rows formatted: tier-colored background (green/amber/purple), box ID in bold monospace
- Column widths set for Box Registry (11 columns) and Item Detail (8 columns) on first write
- ensureHeaders() creates properly formatted header rows with correct column counts
- Box Registry gained: Packed By column, Est. Value (USD) column, Photos (Drive) column

---

## v10 — March 2026

**Apps Script sheet setup + GitHub Pages deployment**

- Google Apps Script (`sheet_setup.gs`) written for one-time sheet formatting setup
- Script sets: dark title row, dark header row, column widths, freeze panes on both tabs
- `formatExistingRows()` function added to script to retroactively format existing data rows
- App deployed to GitHub Pages: https://marcoknauf82.github.io/box-packer/
- Google OAuth Client ID registered for `https://marcoknauf82.github.io` origin
- Google Drive API enabled in Google Cloud project alongside Sheets API

---

## v9 — March 2026

**Google OAuth re-consent + Drive scope**

- Google OAuth now forces consent screen on connect (`prompt: "consent"`) to ensure all scopes granted
- Drive scope upgraded to `https://www.googleapis.com/auth/drive` (from `drive.file`)
- Connected Google account shown as badge in top-right with email prefix
- Badge is tappable to force re-authentication (shows ↺ indicator)
- API key test button added to step 1 — sends minimal test request, shows exact error on failure
- Error handler in analyze() now shows specific error message and hint for common failures

---

## v8 — March 2026

**Anthropic API key security + browser compatibility**

- `anthropic-dangerous-direct-browser-access: true` header added — required for direct browser API calls
- `x-api-key` header format used (replaces Bearer token format for Anthropic)
- API key saved to localStorage on first entry — never needs re-entering in same browser
- API key field shows saved state with Change button
- Image compression added before API call: resize to 1200px max, JPEG 82% quality
- Resolves: Android camera photos exceeding Anthropic 5MB limit (compressed to ~300–500KB)
- `compressImage()` function uses HTML Canvas — works without server, purely client-side

---

## v7 — March 2026

**Google Sheets auto-write + Drive integration**

- Google OAuth 2.0 sign-in added — Connect Google Sheets button on step 1
- On box confirm: automatically appends row to Box Registry and Item Detail tabs
- On box confirm: automatically uploads photos to Google Drive in `Aachen Move / [Box ID] /` folder
- Tier-colored rows in sheet (green=A, amber=B, purple=C)
- Sheet ID hardcoded to user's live Google Sheet
- `sheetsAppend()` and `uploadPhotosToDrive()` functions written
- QR code on label encodes URL to Google Sheet filtered to that box's row

---

## v6 — March 2026

**Label designer + QR codes**

- Label preview panel added — live preview updates as form is filled
- QR code generated on label using qrserver.com API
- Label encodes Google Sheet URL + `?box=[BoxID]` query parameter
- Download printable label button — generates self-contained HTML file
- Print instructions included in downloaded label HTML
- Label sized for 4×2.4 inch output (Brother DK-11204 / DK-11209)

---

## v5 — March 2026

**Packing tier planner + box registry UI**

- Interactive tier planner built: Tier A (daily life), B (karting gear), C (storage)
- Box registry panel added — tracks all boxes packed in current session
- Running stats: total boxes, count by tier
- Add box form: description input, tier selector, key items field, destination room
- Box list shows box ID, description, tier badge, delete button
- Session-only state (no persistence between browser refreshes — by design at this stage)

---

## v4 — March 2026

**Photo analysis workflow — initial implementation**

- Photo upload: tap to take photo or choose from gallery, up to 4 images
- Image sent to Anthropic API (claude-sonnet-4-20250514) for analysis
- API returns: item list, box description, fragile flag, clarification questions
- Clarification questions rendered as tap-to-answer multiple choice cards
- Item review screen: tap × to remove items, pencil to edit, Add field for missing items
- Meta fields on review screen: description, destination room

---

## v3 — March 2026

**Initial prototype — box setup form**

- Step 1: tier selection (A/B/C), destination room, description hint, fragile checkbox
- Step 2: photo upload zone (drag/drop on desktop, tap on mobile)
- Basic 5-step progress indicator
- Tier color system established: green (A), amber (B), purple (C)
- Anthropic API key input with localStorage persistence
- App title and Aachen move branding

---

## v2 — March 2026

**PWA conversion**

- `manifest.json` added: app name, icons, theme color (#534AB7), display standalone
- `service-worker.js` added: caches app shell, bypasses cache for API calls
- App installable on Android home screen via Chrome "Add to Home screen"
- App shortcuts added: "Pack a box" and "Log an item"
- Icon generator HTML file created for generating 192px and 512px icons

---

## v1 — March 2026

**Initial deployment**

- Single HTML file (`box_packer_v1.html` → renamed `index.html`)
- Deployed to GitHub Pages: https://marcoknauf82.github.io/box-packer/
- Google Cloud project created, Sheets API and Drive API enabled
- OAuth 2.0 client registered for GitHub Pages origin
- Google OAuth consent screen configured (External, test user added)

---

## Known issues (open as of v18)

- **Sheet formatting inconsistent on first box write** — race condition in batchUpdate calls; workaround: run `formatExistingRows()` in Apps Script after first write
- **QR code shows fallback text in Claude artifact sandbox** — works correctly on GitHub Pages; not a bug in production
- **Session registry lost on browser refresh** — in-memory registry resets, but box codes now persist in Supabase so numbering is correct
- **Label PNG QR depends on api.qrserver.com availability** — if service is unreachable, QR shows URL text fallback
- **Item Registry tab must exist before first Item write** — app creates it automatically but only on the first confirmed item; if tab was manually deleted, re-run sheet setup
- **Google OAuth re-consent occasionally required** — particularly after Drive scope was upgraded; tap the ↺ badge on step 1 to re-authenticate
- **Supabase publishable key in client code** — acceptable for dogfooding phase with RLS enabled; move to backend proxy in Phase 2
- **Move ID hardcoded** — single move ("Chicago → Aachen 2026") hardcoded as constant; multi-move support deferred to Phase 2

---

## Planned (not yet built)

- Load existing boxes from Supabase on app start (populate registry from database)
- Offline queue for failed writes (service worker background sync)
- PDF export of full inventory with insurance totals
- Backend: Node.js/Express on Railway + Supabase (Phase 2)
- User accounts and multi-device sync (Phase 2)
- Stripe subscriptions and feature gating (Phase 3)
- Play Store submission via TWA (Phase 3)
- React Native app (Phase 4)
