# v20.1.1 Deployment Steps

Canvas-rendering hotfix only. **No SQL migration.** Supersedes v20.1's deploy guide.

## If you haven't run v20.1's SQL migration yet

Run this first (from `DEPLOY_v20_1.md`):

```sql
alter table boxes add column owners text;
alter table standalone_items add column owners text;
```

## If you still see "Supabase save failed"

Most likely `box_code` is still NOT NULL from the v19 schema. Run:

```sql
alter table boxes alter column box_code drop not null;
```

Also confirm the tier CHECK allows A–E:

```sql
alter table boxes drop constraint if exists boxes_tier_check;
alter table boxes add constraint boxes_tier_check check (tier in ('A','B','C','D','E'));
alter table standalone_items drop constraint if exists standalone_items_tier_check;
alter table standalone_items add constraint standalone_items_tier_check check (tier in ('A','B','C','D','E'));
```

(All idempotent.)

## Deploy

1. Copy `index.html` → your local `box-packer` repo root (overwrites v20.1)
2. `git add index.html CLAUDE.md CHANGELOG.md`
3. `git commit -m "v20.1.1: sharper labels, centered box number, fill vertical space"`
4. `git push`
5. Phone: hard refresh the PWA (Chrome menu → History → Clear browsing data → Cached files, or reinstall the home-screen PWA)

## Smoke test

Print one label and check:
1. **Edges look sharper** — QR modules should be crisp black squares, not fuzzy. Text edges should be clean.
2. **Box number is centered** in the left half of the header row (not flush to left padding).
3. **Tier badge is bigger and more legible** from across the room.
4. **Bottom margin is small** — the "Deliver to" address should sit close to the bottom edge, not floating in whitespace.
5. **QR code is noticeably bigger** — should scan faster from the phone camera.

## API key reminder

If you need to re-enter your Anthropic API key:
- Go to **https://console.anthropic.com/settings/keys**
- Log in, click "Create Key" or copy an existing key (starts with `sk-ant-...`)
- In the app, on Step 0 of Pack a Box, paste it into the "Anthropic API key" field. It's saved to localStorage and never transmitted except as the `x-api-key` header to Anthropic's API on photo analysis calls.

