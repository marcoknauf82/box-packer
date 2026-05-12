# v20.2.5 — OAuth Expiry Visibility + Per-Photo Retry

No SQL changes. Two-feature hotfix targeting the silent failures from May 9.

## What's new

### 1. Google OAuth expiry surfaces immediately

When any Google API call returns 401 (Sheets, Drive, etc.), three things happen:
- The "connected" badge at the top turns **red** with "Re-authenticate" text
- The success screen shows a banner: "Google session expired. Tap the red Re-authenticate button..."
- The `googleToken` is cleared so subsequent calls don't waste time on a dead token

Tapping the red badge runs the same OAuth flow as the initial connect. After successful re-auth, the badge turns green again and `googleTokenExpired` resets to false.

### 2. Per-photo retry with visible counter

Photo uploads now retry **3 times each** with backoff (500ms, 1500ms). Each photo is tracked independently — if photo 1 of 4 fails permanently, photos 2-4 still get their full retry budget.

After upload, the success screen shows one of:
- Green bar: "📷 N of N photos uploaded to Drive" (all good)
- Red bar: "**📷 Only X of Y photos uploaded** — missing: photos 2, 4. Re-take or re-pack to retry."

You see the actual photo numbers that failed, not just "something went wrong."

### 3. 401 detection in Drive uploads exits retry early

If a photo upload gets a 401, the retry loop **stops immediately** for that photo (no point retrying with a dead token). The token-expiry flag flips so the badge turns red right away, even mid-batch. Subsequent photos in the same batch also fail fast.

## Deploy

1. Replace `index.html` in your repo
2. `git add index.html && git commit -m "v20.2.5: OAuth expiry banner + per-photo retry with counter" && git push`
3. Hard refresh PWA on phone

## Smoke test

Hard to test the failure paths without actually losing connection or letting OAuth expire — but here's what to look for during real packing:

1. **Top bar shows green badge** with your email after Google connect — normal state.
2. **Pack a box with photos.** On the success screen you should see a small green bar: "📷 N of N photos uploaded to Drive" (this appears even when all goes well — good positive feedback).
3. **If WiFi blips during photo upload:** retry will quietly do its job. You may see only the green "N of N uploaded" banner if retries succeeded, or red if any failed completely.
4. **If your Google session expires:** the badge turns red, success screen shows the expiry warning. Tap badge → re-auth → next box continues normally.

## What this doesn't fix (still v21 territory)

- Full offline queue — if Supabase itself is unreachable, the box save still fails with the red bar. Retry-on-reconnect not yet implemented.
- AI photo analysis fetch is still a single attempt. No retry there.
- Proactive token refresh — we still wait for a 401 to know. A "5-minute warning" before expiry needs token-expiry-time tracking that Google's OAuth2 client doesn't expose directly.

These remain in the proper v21 session scope.
