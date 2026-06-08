# Design Spec: schedule_v4F — Firebase Migration

**Date:** 2026-06-08  
**Author:** Claude Code  
**Status:** Revised after senior engineer review (v2)

---

## Revision Log

| Rev | Date | Change |
|-----|------|--------|
| v1 | 2026-06-08 | Initial draft |
| v2 | 2026-06-08 | Applied senior engineer review — 3 blockers + 9 warnings resolved |

---

## 1. Overview

### Goal

Migrate `schedule_v4.html` (localStorage-based, single-device) to `schedule_v4F.html`
(Firebase-backed, multi-device, multi-user) while preserving all existing UI and features.

### Non-Goals

- No backend server (GitHub Pages stays as-is, static hosting only)
- No real-time multi-device sync within the same session
- No admin dashboard or user management UI
- No changes to existing CSS or visual design

### Success Criteria

- [ ] User can log in with Google account
- [ ] Schedule data persists across devices after login
- [ ] Each user sees only their own data
- [ ] All existing features work: daily tasks, monthly calendar, D&D, copy-to-next-month, print
- [ ] No secrets committed to the GitHub repository

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│  Browser (any device)                               │
│                                                     │
│  index.html / schedule_v4F.html (GitHub Pages)      │
│  ┌───────────────────────────────────────────────┐  │
│  │  #login-screen  (shown when unauthenticated)  │  │
│  │  #app-screen    (existing UI, hidden at first)│  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  Firebase JS SDK v10.12.2 (CDN, dynamic import)     │
│  ← loaded with dynamic import() inside plain        │
│    <script> to preserve global onclick= handlers    │
└──────────────────────────┬──────────────────────────┘
                           │ HTTPS
           ┌───────────────┴───────────────┐
           │                               │
  ┌────────────────┐            ┌─────────────────────┐
  │ Firebase Auth  │            │  Cloud Firestore     │
  │                │            │                      │
  │ Google Sign-In │            │  users/{uid}/        │
  │ → issues uid   │            │  schedule/data       │
  └────────────────┘            └─────────────────────┘
```

**Key principle:** The HTML file is fully static. All logic runs in the browser.
Firebase Auth and Firestore handle identity and persistence.

---

## 3. Technology Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Hosting | GitHub Pages | No change |
| Auth | Firebase Authentication v10.12.2 | Google Sign-In only |
| Database | Cloud Firestore | Document model |
| SDK | Firebase JS SDK **v10.12.2** (CDN, pinned version) | dynamic `import()` in plain `<script>` |
| Offline cache | Firestore `persistentLocalCache` (SDK v10 API) | **Not** `enableIndexedDbPersistence` (removed in v10) |
| Language | Vanilla JavaScript (ES2020+) | No build step, no TypeScript |

### Why dynamic `import()` instead of `<script type="module">`

The existing app uses inline `onclick="functionName()"` attribute handlers throughout.
These resolve against the global `window` scope. `<script type="module">` creates
module scope — functions inside it are NOT visible to `window`, which would silently
break every button and interactive element.

**Solution:** Keep the existing `<script>` (non-module). Load Firebase SDK using
dynamic `import()` inside an async IIFE:

```javascript
// Inside existing plain <script> — no type="module" needed
(async function () {
  const { initializeApp } = await import('https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js');
  const { getAuth, ... }  = await import('https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js');
  const { initializeFirestore, ... } = await import('https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js');
  // … rest of Firebase init and app bootstrap
})();
```

This preserves all `onclick=` handlers and requires zero changes to the HTML template.

---

## 4. Firestore Data Model

### Document Path

```
users/{uid}/schedule/data
```

One document per user. The document mirrors the current localStorage JSON exactly,
with one addition: `lastWriteTime` for optimistic write-conflict detection.

### Document Shape

```json
{
  "lastWriteTime": 1717840000000,
  "saveDate":  "2026-06-08",
  "morning":   [ { "id": 1717840001234, "title": "...", "category": "visit",
                   "priority": "mid", "time": "09:00", "slot": "morning",
                   "done": false, "note": null, "order": 0 } ],
  "afternoon": [ ... ],
  "anytime":   [ ... ],
  "ongoing":   [ { "id": 1717840002000, "title": "...", "deadline": "5月末",
                   "progress": 40, "category": "admin" } ],
  "overdue":   [ { "id": 1717840003000, "title": "...", "phase": "企画中",
                   "daysOver": 0 } ],
  "future":    [ { "id": 1717840004000, "title": "...", "date": "2026-06-15",
                   "slot": "afternoon", "category": "visit",
                   "priority": "high", "time": "14:00" } ],
  "holidays":  { "2026-05-04": "full" },
  "recurring": {
    "yamanakajuku":       { "type": "nth-weekday", "n": 3, "dow": 5 },
    "bushokai_shacho":    { "2026-06": 12 },
    "bushokai_dentatsu":  {},
    "bushokai_kyotaku":   {},
    "tomita":             {},
    "keiei_1":            {},
    "keiei_2":            {},
    "keiei":              {}
  },
  "nextId": 1717840010000
}
```

**Size estimate:** Typical daily usage ≈ 5–20 tasks ≈ 2–15 KB per document.
Firestore document limit is 1 MB. No risk of hitting the limit.

### ID Strategy (revised from v1)

**Use `Date.now()` as the initial `nextId`, incremented per task.**

The original sequential `nextId` (300, 301, …) risks collision when the same user
opens the app on two devices simultaneously: both load `nextId: 310`, both create
`id: 310` for different tasks, and one overwrites the other.

Using `Date.now()` (millisecond timestamp) as the starting value — e.g., `1717840010000` —
makes collisions between two devices in the same millisecond practically impossible.

Migration: on first Firestore write, if `nextId < 1_000_000` (old sequential ID),
reset to `Date.now()`.

---

## 5. Security Model

### Firestore Security Rules (revised from v1)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Each user can only read/write their own schedule document
    match /users/{userId}/schedule/data {
      allow read: if request.auth != null
                  && request.auth.uid == userId
                  && request.auth.token.email in [
                       'user1@gmail.com',
                       'user2@gmail.com',
                       'user3@gmail.com'
                     ];

      allow write: if request.auth != null
                   && request.auth.uid == userId
                   && request.auth.token.email in [
                        'user1@gmail.com',
                        'user2@gmail.com',
                        'user3@gmail.com'
                      ]
                   // Validate document structure to prevent garbage writes
                   && request.resource.data.keys().hasOnly([
                        'lastWriteTime','saveDate','morning','afternoon','anytime',
                        'ongoing','overdue','future','holidays','recurring','nextId'
                      ])
                   && request.resource.data.nextId is int
                   && request.resource.data.morning   is list
                   && request.resource.data.afternoon is list
                   && request.resource.data.anytime   is list;
    }

    // Deny everything else
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

**Email allowlist:** Restrict access to the specific care manager team. Any Google account
not on this list is denied even if they have the correct uid. Add/remove email addresses
in the Firebase Console (Security Rules) as team members change.

### Firebase Client Config

The Firebase client config (`apiKey`, `authDomain`, etc.) is **intentionally public**.
It identifies the project but grants no privileges. Security is enforced by:
1. Firebase Auth (users must be authenticated)
2. Security Rules (uid must match + email must be in allowlist + document structure validated)

This config is safe to commit to GitHub. **Never commit service account JSON keys.**

---

## 6. Authentication Flow

### Initialization (CDN load failure handling)

```javascript
window.addEventListener('error', (e) => {
  if (e.filename && e.filename.includes('gstatic.com')) {
    document.getElementById('login-screen').innerHTML =
      '<p>ネットワークエラーです。接続を確認してください。</p>';
  }
});
```

### Login with popup + automatic redirect fallback

```javascript
async function login() {
  try {
    await signInWithPopup(auth, googleProvider);
  } catch (err) {
    if (['auth/popup-blocked',
         'auth/popup-closed-by-user',
         'auth/cancelled-popup-request'].includes(err.code)) {
      // iOS Safari / Firefox strict mode: fall back to redirect
      await signInWithRedirect(auth, googleProvider);
    } else if (err.code === 'auth/unauthorized-domain') {
      showLoginError('このドメインは許可されていません。管理者に連絡してください。');
    } else {
      showLoginError('ログインに失敗しました: ' + err.message);
    }
  }
}
```

### Redirect result capture (must run before onAuthStateChanged)

```javascript
// At app bootstrap — capture redirect login result
const redirectResult = await getRedirectResult(auth).catch(() => null);
// onAuthStateChanged then fires with the user from the redirect
```

### Auth state management

```javascript
onAuthStateChanged(auth, async (user) => {
  if (user) {
    currentUser = user;
    showAppScreen();
    await loadData();
    renderAll();
  } else {
    currentUser = null;
    showLoginScreen();
    data = defaultData();
  }
});
```

### Session Persistence

Firebase SDK persists auth state in IndexedDB by default (`browserLocalPersistence`).
Users remain logged in across browser restarts without re-authenticating.

### Logout (flush pending saves first)

```javascript
async function logout() {
  // Flush any pending debounced write before signing out
  if (_saveTimer) {
    clearTimeout(_saveTimer);
    await _flushSave();
  }
  await signOut(auth);
  // onAuthStateChanged fires → showLoginScreen(), data cleared
}
```

---

## 7. Storage Abstraction Layer

```javascript
const StorageAdapter = {
  async save(uid, payload) {
    const ref = doc(db, 'users', uid, 'schedule', 'data');
    await setDoc(ref, payload);
  },
  async load(uid) {
    const ref = doc(db, 'users', uid, 'schedule', 'data');
    const snap = await getDoc(ref);
    return snap.exists() ? snap.data() : null;
  }
};
```

`saveData()` and `loadData()` call `StorageAdapter` — they don't know whether the
backend is Firestore, localStorage, or a mock. Enables future backend swaps and testing.

---

## 8. Debounced Writes

```javascript
let _saveTimer = null;

function saveData() {
  // Show "保存中..." indicator
  showSaveIndicator('pending');
  clearTimeout(_saveTimer);
  _saveTimer = setTimeout(() => _flushSave(), 800);
}

async function _flushSave() {
  if (!currentUser) return;
  const payload = buildSavePayload();
  payload.lastWriteTime = Date.now();
  try {
    await StorageAdapter.save(currentUser.uid, payload);
    showSaveIndicator('ok');       // "保存済み ✓"
  } catch (err) {
    console.error('Save failed:', err);
    showSaveIndicator('error');    // "保存失敗 — 再試行中..."
  }
}

// Flush on tab/window close to avoid losing the final edit
window.addEventListener('beforeunload', () => {
  if (_saveTimer) {
    clearTimeout(_saveTimer);
    _flushSave();  // best-effort; Firestore offline queue handles the rest
  }
});
```

**Save indicator:** A small status pill in the header (e.g., top-right corner):
- `pending` → "保存中..." (spinning icon)
- `ok`      → "保存済み ✓" (fades out after 2s)
- `error`   → "保存失敗" (red, persistent until next successful save)

---

## 9. Offline Persistence (SDK v10 API)

```javascript
// SDK v10: use initializeFirestore with localCache option
// NOT enableIndexedDbPersistence (removed in v10)
import { initializeFirestore, persistentLocalCache, persistentMultipleTabManager }
  from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js';

const db = initializeFirestore(app, {
  localCache: persistentLocalCache({
    tabManager: persistentMultipleTabManager()
  })
});
```

`persistentMultipleTabManager()` allows the app to work correctly when open in
multiple browser tabs — important since care managers may keep both the daily view
and monthly calendar tabs open simultaneously.

**Effect:** If the user opens the app without internet, they see their last-loaded data
and can continue editing. Changes are queued and synced when connectivity resumes.

---

## 10. Error Handling

| Scenario | Behavior |
|---------|---------|
| Firestore write fails (network drop) | `showSaveIndicator('error')` — persistent red banner until next successful save |
| Firestore read fails (offline, first load) | Show "オフラインモード — データを読み込めませんでした" banner; data = empty default |
| Firestore read fails (offline, cached) | IndexedDB cache serves stale data; "オフラインモード" banner shown |
| Auth popup blocked (desktop) | `showLoginError` message; auto-fallback to `signInWithRedirect` |
| Auth popup closed by user | Catch `auth/popup-closed-by-user`; show dismissable "ログインをキャンセルしました" |
| User signs out | `onAuthStateChanged` fires; login screen shown; in-memory data cleared |
| Logout with pending debounce | `_flushSave()` called synchronously before `signOut()` |
| Firebase CDN unreachable | `window.onerror` handler shows network error message |
| Unauthorized email tries to log in | Security Rules deny; show "このアカウントはご利用いただけません" |
| Tab close within 800ms debounce window | `beforeunload` calls `_flushSave()`; Firestore offline queue persists the write |
| promoteFuture() fires on load | Save called immediately (not debounced) if tasks were promoted |

### `promoteFuture` save handling

`promoteFuture()` (line 1456) calls `saveData()` when future tasks are promoted to
today. In the Firestore version this fires during `loadData()`, before `renderAll()`.
The debounce guard (`if (!currentUser) return`) will NOT suppress this because
`currentUser` is set before `loadData()` is called. However, to avoid the debounce
delay masking the save:

```javascript
// In loadData(), after calling promoteFuture():
if (promoteFuture() > 0) {
  clearTimeout(_saveTimer);
  await _flushSave();  // immediate write, not debounced
}
```

---

## 11. Code Changes Summary

### Files

| File | Action |
|------|--------|
| `schedule_v4.html` | **No change** — preserved as-is |
| `schedule_v4F.html` | **New file** — copy of v4 + Firebase changes |
| `index.html` | **Update after v4F stabilises** — copy of v4F |
| `docs/superpowers/specs/...` | **New** — this design doc |

### Additions to `schedule_v4F.html`

1. **`<script>` (async IIFE at bottom):**
   - Dynamic `import()` of Firebase SDK v10.12.2 from CDN
   - `firebaseConfig` constant
   - `initializeApp` + `initializeFirestore` with `persistentLocalCache`
   - `getAuth`, `GoogleAuthProvider` init
   - `StorageAdapter` object
   - `currentUser` variable
   - `_saveTimer`, `_flushSave()` debounce wrapper
   - `onAuthStateChanged` handler
   - `getRedirectResult` call (redirect auth capture)
   - `login()` with popup + redirect fallback
   - `logout()` with pre-flush
   - `showSaveIndicator()` helper
   - `showLoginScreen()` / `showAppScreen()` CSS toggle helpers
   - `window.beforeunload` handler
   - CDN error handler
   - `nextId` migration guard (`if nextId < 1_000_000, reset to Date.now()`)
2. **`#login-screen` HTML block** (before `#app-screen`)
3. **Save indicator pill** in header HTML
4. **Login screen CSS** (overlay, Google button)

### Modified in `schedule_v4F.html`

| Function | Change |
|---------|--------|
| `saveData()` | Becomes debounce trigger; delegates to `_flushSave` → `StorageAdapter.save` |
| `loadData()` | Becomes `async`; reads from `StorageAdapter.load`; promotes future tasks then immediate-saves if needed |
| `init()` | Removed from direct call; logic absorbed into `onAuthStateChanged` handler |
| `nextId` init | Changed from `let nextId = 300` to timestamp-based on first write |

**Total lines changed/added:** approximately 160–180 lines out of 2763.
**Lines of existing logic modified:** fewer than 15.

---

## 12. localStorage Migration

### Detection

On first login, check if:
1. The Firestore document does NOT exist (new user to Firestore)
2. AND localStorage has data under key `'care-schedule-v1'`
3. AND migration flag `'care-schedule-v1-migrated'` is NOT set

```javascript
function getLocalStorageData() {
  try {
    const raw = localStorage.getItem('care-schedule-v1');
    return raw ? JSON.parse(raw) : null;
  } catch {
    return null;  // private mode or parse error
  }
}
```

### Migration Flow

```
First login → Firestore doc missing → localStorage data found
  → show banner "以前のデータをクラウドに移行しますか？"
    → [移行する] → write to Firestore
                 → localStorage.setItem('care-schedule-v1-migrated', '1')
                 → localStorage.removeItem('care-schedule-v1')
    → [スキップ]  → start with empty Firestore document
```

**Atomicity:** The migration-done flag (`-migrated`) is set AFTER a successful
Firestore write. If the write fails, the flag is not set and migration retries on
next login. The original localStorage data is preserved until migration succeeds.

---

## 13. Implementation Phases

### Phase 1 — Firebase Console Setup (no code changes)
1. Create Firebase project
2. Enable Google Sign-In in Authentication settings
3. Add GitHub Pages domain to "Authorised domains" in Auth settings
4. Create Firestore database in production mode
5. Enter Firestore Security Rules from Section 5 (including email allowlist)
6. Validate Security Rules with Firebase emulator
7. Copy Firebase client config (apiKey, authDomain, etc.)

### Phase 2 — Create `schedule_v4F.html`
1. Copy `schedule_v4.html` → `schedule_v4F.html`
2. Add async IIFE with Firebase SDK dynamic imports
3. Add `firebaseConfig`, `initializeApp`, `initializeFirestore` with `persistentLocalCache`
4. Add `StorageAdapter`
5. Add `currentUser`, debounce wrapper, `_flushSave`
6. Modify `loadData()` → async, reads from `StorageAdapter`
7. Add `promoteFuture()` immediate-save handling
8. Add `onAuthStateChanged` handler (replaces direct `init()` call)
9. Add `getRedirectResult` call at bootstrap
10. Add `login()` with popup + redirect fallback
11. Add `logout()` with pre-flush
12. Add `#login-screen` HTML
13. Add save indicator pill in header
14. Add `beforeunload` and CDN error handlers
15. Add `nextId` migration guard

### Phase 3 — Testing
1. Google login popup works (Chrome desktop)
2. Google login redirect works (iOS Safari, mobile)
3. Unauthorized Google account is denied with clear error message
4. Data saves to Firestore (check Firebase Console)
5. Data loads correctly on second device after login
6. Other user's data is inaccessible
7. All existing features: daily tasks, D&D, monthly calendar, print, recurring, copy-to-next-month
8. Offline: disconnect → edit → reconnect → verify data synced
9. Tab close within 800ms → data not lost
10. Logout flushes pending saves

### Phase 4 — localStorage Migration
1. Implement migration banner
2. Test on device with existing localStorage data
3. Verify migration flag prevents double-migration

### Phase 5 — Deploy
1. Copy `schedule_v4F.html` → `index.html`
2. `git add index.html schedule_v4F.html`
3. `git commit -m "feat: Firebase Auth + Firestore migration (v4F)"`
4. `git push` → GitHub Pages auto-deploys

---

## 14. Risk Matrix (updated)

| Risk | P | Impact | Mitigation |
|------|---|--------|-----------|
| Concurrent write from two devices (same user) | Low | Low | `lastWriteTime` field + `Date.now()` IDs reduce collision; single-user pattern means "last write wins" is acceptable |
| iOS/Safari popup blocked | Medium | Low | Automatic `signInWithRedirect` fallback |
| Tab close within debounce window | Low | Low | `beforeunload` flushes; Firestore offline queue catches remainder |
| Firebase free tier exceeded | Very Low | Medium | ~3–5 users × ~30 writes/day = ~150/day vs 20K limit |
| Firestore document too large | Very Low | Low | Max ~50KB vs 1MB limit |
| Email not in allowlist tries to sign in | Low | None | Security Rules deny immediately |
| CDN unreachable (corporate firewall) | Low | High | Error banner shown; cannot use app offline-first in this scenario |
| Firebase misconfiguration | Medium | High | Validate with emulator before Phase 5 |

---

## 15. Firebase Free Tier

Firebase Spark plan (free):
- **Reads:** 50,000/day
- **Writes:** 20,000/day
- **Storage:** 1 GB

Estimated usage (5 users):
- 5 reads/day (one load per user per day)
- ~150 writes/day (30 debounced writes per user)

Comfortably within free tier. No billing concerns.

---

## 16. Open Questions — Resolved

| Q | Decision |
|---|---------|
| Email allowlist? | **Yes.** Added to Security Rules. Required for healthcare app used by named team. |
| `signInWithRedirect` fallback? | **Yes.** Automatic fallback on popup failure. Required for iOS/Safari support. |
| Firestore environments? | **Single project.** Use `schedule` vs `schedule-dev` collection name convention for dev testing. |
| Logout button placement? | **Header, right side.** Show `user.displayName` + dropdown containing "ログアウト". Requires confirmation click to prevent accidental logout. |
