# v20 Deployment Steps

## 1. Verify the SQL migration ran
You've already run the migration. Sanity check in Supabase SQL Editor:

```sql
-- boxes table should have these columns
select column_name from information_schema.columns
where table_name='boxes' and column_name in
  ('box_number','short_summary','origin_home','origin_room','box_size','weight_kg');
-- should return 6 rows

-- origin_rooms seeded
select origin_home, count(*) from origin_rooms
where move_id='f3ec17dd-437a-4e60-bbe5-58d1ce340ed9'
group by origin_home;
-- should show Chicago Home and Holiday Home with rooms

-- tier CHECK allows A-E
-- (test: should succeed)
-- insert into boxes (move_id, box_number, tier) values ('f3ec17dd-437a-4e60-bbe5-58d1ce340ed9', 999, 'E') returning id;
-- rollback after

-- confirm no stale test data
select count(*) from boxes;          -- should be 0
select count(*) from standalone_items; -- should be 0
```

## 2. Deploy to GitHub Pages
1. Copy `index.html` → your local `box-packer` repo root, overwriting the existing v19 file
2. `git add index.html`
3. `git commit -m "v20: Nelko label rewrite, 5 tiers, integer box numbers, new fields"`
4. `git push`
5. Wait ~1 minute for GitHub Pages to rebuild
6. On your Android phone, open https://marcoknauf82.github.io/box-packer/ — you may need a hard refresh (Chrome menu → history → clear browsing data → cached files, or reinstall the PWA)

## 3. Also copy these into your repo and project
- `CLAUDE.md` → repo root (overwrites v19 file)
- `CHANGELOG.md` → repo root (overwrites v19 file)

Re-upload both to your Claude Project as well, so future sessions have the updated context.

## 4. Smoke test before real packing
1. Open the app, tap "Pack a Box"
2. Verify the 5 tier buttons A–E all render and are selectable
3. Pick Chicago Home → verify rooms chip list populates
4. Pick a room, then tap "+ Add" a new made-up room → verify it persists after refresh
5. Pick Holiday Home → verify a separate room list appears
6. Take a photo, run analyze, confirm a test box
7. Check Supabase Dashboard → boxes table → newest row has all new fields populated
8. From the label screen, tap "1 copy" — verify the label PNG generates and the NELKO app opens
9. Print one test label and confirm readability

## 5. Known characteristics (see CHANGELOG for full list)
- If you pack from two devices simultaneously with the same tier number candidate, the second will fail with a unique-constraint error and show "supa-fail". Just retry — the next save will get the next number.
- Orphan-box risk: if the box row inserts but the items fail, you see an error. The box row will exist in Supabase without items. Manually delete via Supabase UI, or proceed and fix later.
- The first `loadOriginRooms()` call on app startup is async. If you tap "Pack a Box" within the first ~500ms on a slow connection, the room list will render empty. Tap a home button again to trigger a re-render.

