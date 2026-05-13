# v20.2.6 — Two Lowe's box sizes

## Required SQL — run before deploying

```sql
alter table boxes drop constraint if exists boxes_box_size_check;
alter table boxes add constraint boxes_box_size_check
  check (box_size = any (array[
    'Extra Small','Small','Medium','Large','Extra Large',
    'Imperfect Box',
    'Lowe''s S','Lowe''s M',
    'Oversized'
  ]));
```

Note the doubled apostrophe in `'Lowe''s S'` — that's the SQL way to escape a single quote inside a string literal.

## What's new

Two new box-size tiles in the picker:
- **Lowe's S** (16×12×12 in)
- **Lowe's M** (18×18×16 in)

Both have known dimensions, so no manual prompt — they auto-fill the dimensions field, like Small / Medium / Large / Imperfect.

The new tiles render between "Imperfect Box" and "Oversized" in the grid, so the unknown-dimensions options (XS, XL, Oversized) stay grouped at the corners.

## Deploy

1. Run the SQL above
2. Replace `index.html`, commit, push
3. Hard refresh PWA on phone
4. Tap "Pack a Box" → reach box-size picker → verify two new Lowe's tiles appear and pre-fill dimensions when selected

## What's deferred to a later version

Coarse-category column (Books / Clothing / Documents / etc) for the EOS PDF list. Marco will do this as a batched pass at the end of packing, not per-box. Tracked in memory.
