# v20.2.3 — Drive link persistence

## Defensive SQL — run before deploying

Make sure the `drive_folder_url` column exists on both tables. Idempotent:

```sql
alter table boxes add column if not exists drive_folder_url text;
alter table standalone_items add column if not exists drive_folder_url text;
```

## What's new

After photos upload to Drive, the resulting folder URL is now persisted back to
the `boxes.drive_folder_url` (or `standalone_items.drive_folder_url`) column.
Effect: scanning the QR code on a label opens the box detail screen with a working
"Photos" card that links to the Drive folder.

## Backfill for box 1 (the test box you just packed)

Drive folder URL was never written. To fix it manually:

```sql
-- Find the Drive folder URL by browsing to Aachen Move/1/photos in Google Drive
-- Right-click the parent folder (named "1") → "Share" → "Copy link"
-- That link looks like: https://drive.google.com/drive/folders/<long-id>?...

update boxes
set drive_folder_url = 'PASTE_DRIVE_FOLDER_LINK_HERE'
where move_id = 'f3ec17dd-437a-4e60-bbe5-58d1ce340ed9'
and box_number = 1;
```

Strip the `?usp=...` query suffix from the URL if it has one — keep just the
`/folders/<id>` portion. Or leave it; the link works either way.

If you don't care about box 1's Drive link, skip this — every box from 2 onward
will have it automatically.

## Deploy

1. Run the defensive SQL above
2. Replace `index.html` in your repo
3. Commit + push
4. Hard refresh PWA on phone

## Smoke test

1. Pack a fresh box with photos
2. Confirm save succeeds
3. Open Box Registry → tap the new box → verify "Photos" card with Drive link appears
4. Scan the QR code on that box's label → app should navigate to the same detail screen with photos card present
