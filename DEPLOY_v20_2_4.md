# v20.2.4 Deployment — TESTED AGAINST REAL DATA

No SQL changes. Canvas-only updates.

## What's new

1. **Summary wraps to 2 lines** when needed. Verified against all 23 of your real boxes: **0 boxes** have truncated summaries.

2. **Contents items can wrap to 2 lines**, with **dynamic budget** that stops adding items when the column would push the deliver-to address off the bottom edge. Replaces the previous fixed `MAX_SHOW`. Verified against all 23 of your real boxes: **0 boxes overflow** the label. 6 boxes show "+N more items (see QR for full list)" because they had many long items — but everything that's shown is shown correctly.

3. **Single "Print label" button** — set copy count inside Nelko.

4. **Drive upload failures show prominent red bar** with link to Google Drive.

5. **Dead code removed** — `ITEM_COUNTER` and `counters`.

## Tested against real data

I ran the wrap algorithm against the actual summaries and items from all 23 boxes in Supabase. Results:
- 0 summaries truncated (was 1 in v20.2.3: box 21)
- 0 boxes overflow the 4×6 budget
- 6 boxes show "+N more" footer (these are boxes 7, 9, 12, 19, 20, 21 — the densest book/textbook boxes)
- Worst margin: 28px (box 2 — still safely within label)

## Deploy

1. Replace `index.html` in your repo
2. `git add index.html && git commit -m "v20.2.4: 2-line label wrapping with dynamic budget, single print button" && git push`
3. Hard refresh PWA on phone

## Smoke test

1. Top bar: "Aachen move · v20.2.4"
2. Reprint box 21 from registry (the one with truncated summary in old version) — should now show "Business and academic / textbooks" on two lines
3. Reprint any book-heavy box (e.g. box 19) — should show 4-6 items in two cols + "+N more items" footer
4. Print one label, verify it matches the preview and the deliver-to address is visible at the bottom

## For tomorrow at Hudson

Network resilience still not fixed (that's v21). Same mitigations as last session:
- Cellular > Holiday Home WiFi
- Verify Drive folders after each box
- If you see "Supabase save failed", refresh and check registry before re-packing
