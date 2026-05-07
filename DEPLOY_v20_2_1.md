# v20.2.1 Deployment

## No SQL changes this round

(But if you haven't run v20.2's SQL yet, do that first — see DEPLOY_v20_2.md or run from memory.)

## What's in v20.2.1

- **QR codes now point to the app** instead of Google Sheets
- **Scan-to-lookup works** — scanning a printed label opens the app to that box's detail screen (with photos, items, all metadata)
- **Skip photos button** added to Step 1 — lets you manually enter contents without uploading photos

## 1. First, clear test data (run in Supabase SQL editor)

```sql
delete from boxes where move_id = 'f3ec17dd-437a-4e60-bbe5-58d1ce340ed9';
delete from standalone_items where move_id = 'f3ec17dd-437a-4e60-bbe5-58d1ce340ed9';

select count(*) from boxes where move_id = 'f3ec17dd-437a-4e60-bbe5-58d1ce340ed9';   -- 0
select count(*) from box_items where move_id = 'f3ec17dd-437a-4e60-bbe5-58d1ce340ed9'; -- 0
select count(*) from standalone_items where move_id = 'f3ec17dd-437a-4e60-bbe5-58d1ce340ed9'; -- 0
```

Optionally clear the Google Sheet rows manually (Box Registry tab) and any test box folders in Google Drive `Aachen Move/`.

## 2. Deploy

1. Replace `index.html` in your repo
2. `git add index.html && git commit -m "v20.2.1: QR scan-to-lookup, skip photos button" && git push`
3. Hard refresh PWA on phone

## 3. Smoke test

1. Top bar shows "Aachen move · v20.2.1"
2. Pack a Box → take photo → analyze → confirm. Save should succeed.
3. **Test scan-to-lookup**: print the label (or use phone QR scanner on the in-app preview QR), scan it. Phone should open the app and navigate to that box's detail screen automatically.
4. **Test skip photos**: Pack a Box → on Step 1 (photos), tap "Skip — enter contents manually". Should jump to review. Add 2-3 items via the "+" button. Type a short summary. Confirm box. Should save without ever uploading a photo.

## 4. Known behavior with skip photos

- AI doesn't suggest a short summary or item descriptions — you type everything
- Value source defaults to empty (`value_source_type` = `not_set`)
- You can still add manual values per item via the pencil edit modal
- Drive folder for the box won't be created (no photos to upload)

