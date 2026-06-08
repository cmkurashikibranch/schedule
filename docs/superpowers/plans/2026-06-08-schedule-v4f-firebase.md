# schedule_v4F Firebase Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate `schedule_v4.html` to `schedule_v4F.html` by replacing localStorage with Firebase Authentication (Google Sign-In) + Cloud Firestore, while preserving all existing UI and features.

**Architecture:** Copy v4 to a new file `schedule_v4F.html`. Add a thin Firebase layer: a dynamic `import()` IIFE inside the existing `<script>` loads the SDK, a `StorageAdapter` object wraps Firestore read/write, and `onAuthStateChanged` drives app initialization. `saveData()` becomes a debounced wrapper; `loadData()` becomes async. All `onclick=` handlers remain untouched.

**Tech Stack:** Firebase JS SDK v10.12.2 (CDN, dynamic import), Firebase Auth (Google Sign-In), Cloud Firestore (persistent local cache), Vanilla JS, GitHub Pages.

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `schedule_v4.html` | **No change** | Original localStorage version — preserved |
| `schedule_v4F.html` | **Create** | Firebase version — all changes go here |
| `index.html` | **Update** (Task 14) | Copy of v4F after all tests pass |
| `docs/superpowers/specs/2026-06-08-schedule-v4f-firebase-design.md` | Already exists | Design spec with security rules |

---

## Pre-requisites (do before Task 1)

You need a Firebase project with:
- Google Sign-In enabled
- Firestore database created
- Security Rules configured
- GitHub Pages domain authorised

See Task 1 for step-by-step setup.

---

## Task 1: Firebase Console Setup

**Files:** None (browser-based setup in Firebase Console)

> This task has no code. Follow each step in the Firebase Console at console.firebase.google.com.

- [ ] **Step 1-1: Create Firebase project**

  Go to console.firebase.google.com → "プロジェクトを追加" → Name: `care-schedule` → Continue through wizard → Create project.

- [ ] **Step 1-2: Enable Google Sign-In**

  Project → Build → Authentication → Get started → Sign-in method tab → Google → Enable → enter support email → Save.

- [ ] **Step 1-3: Authorise GitHub Pages domain**

  Authentication → Settings → Authorised domains → Add domain:
  ```
  cmkurashikibranch.github.io
  ```

- [ ] **Step 1-4: Create Firestore database**

  Build → Firestore Database → Create database → **Production mode** → Select region: `asia-northeast1` (Tokyo) → Enable.

- [ ] **Step 1-5: Set Security Rules**

  Firestore Database → Rules tab → Replace the entire content with:

  ```javascript
  rules_version = '2';
  service cloud.firestore {
    match /databases/{database}/documents {

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
                     && request.resource.data.keys().hasOnly([
                          'lastWriteTime','saveDate','morning','afternoon','anytime',
                          'ongoing','overdue','future','holidays','recurring','nextId'
                        ])
                     && request.resource.data.nextId is int
                     && request.resource.data.morning   is list
                     && request.resource.data.afternoon is list
                     && request.resource.data.anytime   is list;
      }

      match /{document=**} {
        allow read, write: if false;
      }
    }
  }
  ```

  **Replace the email addresses** with the actual Gmail addresses of team members. → Publish.

- [ ] **Step 1-6: Copy Firebase client config**

  Project settings (gear icon) → General tab → "マイアプリ" section → `</>` Web → Register app → name: `schedule` → Register → Copy the `firebaseConfig` object. It looks like:

  ```javascript
  const firebaseConfig = {
    apiKey:            "AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    authDomain:        "care-schedule-XXXXX.firebaseapp.com",
    projectId:         "care-schedule-XXXXX",
    storageBucket:     "care-schedule-XXXXX.appspot.com",
    messagingSenderId: "123456789012",
    appId:             "1:123456789012:web:XXXXXXXXXXXXXXXX"
  };
  ```

  Save this somewhere — you'll paste it into `schedule_v4F.html` in Task 3.

- [ ] **Step 1-7: Verify**

  Firestore → Rules → check the rules are saved.
  Authentication → Sign-in method → Google: Enabled.
  Authentication → Settings → Authorised domains: `cmkurashikibranch.github.io` is listed.

---

## Task 2: Create `schedule_v4F.html`

**Files:**
- Create: `スケジュール管理/schedule_v4F.html`

- [ ] **Step 2-1: Copy v4 to v4F**

  ```powershell
  Copy-Item "C:\Users\kkmh2\claude\スケジュール管理\schedule_v4.html" `
            "C:\Users\kkmh2\claude\スケジュール管理\schedule_v4F.html"
  ```

- [ ] **Step 2-2: Verify the copy**

  ```powershell
  (Get-Item "C:\Users\kkmh2\claude\スケジュール管理\schedule_v4F.html").Length
  ```

  Expected: same byte count as `schedule_v4.html` (approx 180,000+ bytes).

- [ ] **Step 2-3: Commit the base copy**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat: create schedule_v4F.html as base for Firebase migration"
  ```

---

## Task 3: Add Login Screen HTML + CSS

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — `<body>` tag and `</style>` section

The login screen is a full-page overlay. The existing `<header>` + `<main>` are wrapped in `#app-screen` which starts hidden.

- [ ] **Step 3-1: Wrap existing body content in `#app-screen`**

  In `schedule_v4F.html`, find this exact line (line 1076):
  ```html
  <body>
  ```

  Replace it with:
  ```html
  <body>

    <!-- ── Login Screen ───────────────────────────────── -->
    <div id="login-screen">
      <div class="login-card">
        <div class="login-logo">📋</div>
        <h1 class="login-title">スケジュール管理</h1>
        <p class="login-sub">ケアマネジャー向けスケジュール管理アプリ</p>
        <button class="login-btn" id="login-btn" onclick="login()">
          <svg class="google-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
            <path d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z" fill="#4285F4"/>
            <path d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z" fill="#34A853"/>
            <path d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l3.66-2.84z" fill="#FBBC05"/>
            <path d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z" fill="#EA4335"/>
          </svg>
          Googleでログイン
        </button>
        <p class="login-error" id="login-error"></p>
      </div>
    </div>

    <!-- ── App Screen (existing UI) ───────────────────── -->
    <div id="app-screen" style="display:none;">
  ```

- [ ] **Step 3-2: Close `#app-screen` div before `</body>`**

  Find the closing `</body>` tag (line 2763):
  ```html
  </body>
  ```

  Replace it with:
  ```html
    </div><!-- /#app-screen -->
  </body>
  ```

- [ ] **Step 3-3: Add login screen CSS**

  Find this exact line (just before `</style>`):
  ```css
    .add-memo-input::placeholder { color: var(--text-3); }
  </style>
  ```

  Replace with:
  ```css
    .add-memo-input::placeholder { color: var(--text-3); }

    /* ── Login Screen ──────────────────────────────────── */
    #login-screen {
      position: fixed; inset: 0; z-index: 9999;
      display: flex; align-items: center; justify-content: center;
      background: var(--bg);
    }
    .login-card {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 16px;
      padding: 48px 40px;
      text-align: center;
      width: 340px;
      box-shadow: 0 8px 32px rgba(0,0,0,0.4);
    }
    .login-logo { font-size: 40px; margin-bottom: 16px; }
    .login-title {
      font-size: 22px; font-weight: 700; color: var(--text);
      margin-bottom: 8px; letter-spacing: -0.02em;
    }
    .login-sub { font-size: 13px; color: var(--text-2); margin-bottom: 32px; }
    .login-btn {
      display: flex; align-items: center; justify-content: center; gap: 10px;
      width: 100%; padding: 12px 20px; border-radius: 8px;
      border: 1px solid var(--border);
      background: var(--surface-2); color: var(--text);
      font-family: 'Inter', 'Noto Sans JP', sans-serif;
      font-size: 14px; font-weight: 500; cursor: pointer;
      transition: background 0.15s, border-color 0.15s;
    }
    .login-btn:hover { background: var(--accent-bg); border-color: var(--accent); }
    .login-btn:disabled { opacity: 0.5; cursor: not-allowed; }
    .google-icon { width: 18px; height: 18px; flex-shrink: 0; }
    .login-error {
      margin-top: 16px; font-size: 12px; color: var(--pri-high);
      min-height: 16px; line-height: 1.4;
    }

    /* ── User Menu (header right) ───────────────────────── */
    .user-menu { position: relative; }
    .user-btn {
      display: flex; align-items: center; gap: 8px;
      background: none; border: 1px solid var(--border);
      border-radius: 8px; padding: 5px 10px 5px 6px;
      color: var(--text-2); cursor: pointer;
      font-family: 'Inter', 'Noto Sans JP', sans-serif;
      font-size: 12px; transition: background 0.12s;
    }
    .user-btn:hover { background: var(--surface-2); }
    .user-avatar {
      width: 22px; height: 22px; border-radius: 50%;
      object-fit: cover; background: var(--surface-2);
    }
    .user-dropdown {
      position: absolute; right: 0; top: calc(100% + 6px);
      background: var(--surface); border: 1px solid var(--border);
      border-radius: 8px; padding: 4px;
      box-shadow: 0 4px 16px rgba(0,0,0,0.3);
      min-width: 120px; z-index: 200;
    }
    .user-dropdown button {
      display: block; width: 100%; text-align: left;
      padding: 8px 12px; background: none; border: none;
      color: var(--text); font-size: 13px; cursor: pointer;
      border-radius: 5px;
    }
    .user-dropdown button:hover { background: var(--surface-2); color: var(--pri-high); }

    /* ── Save Indicator ─────────────────────────────────── */
    .save-indicator {
      font-family: 'JetBrains Mono', monospace;
      font-size: 10px; padding: 3px 8px;
      border-radius: 5px; transition: opacity 0.3s;
    }
    .save-indicator.pending { color: var(--text-2); }
    .save-indicator.ok      { color: #4ade80; }
    .save-indicator.error   { color: var(--pri-high); background: rgba(248,113,113,0.12); }
    .save-indicator.hidden  { opacity: 0; pointer-events: none; }
  </style>
  ```

- [ ] **Step 3-4: Verify HTML structure in browser**

  Open `schedule_v4F.html` directly in Chrome (drag file into browser).
  Expected: A dark login screen with "スケジュール管理" title and Google login button is shown. The app behind it should be invisible.

- [ ] **Step 3-5: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): add login screen HTML and CSS"
  ```

---

## Task 4: Add Header UI (save indicator + user menu)

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — `<header>` section (around line 1078)

- [ ] **Step 4-1: Add save indicator and user menu to header**

  Find this exact block (around line 1083–1101):
  ```html
      <div class="header-stats">
        <div class="stat-item s-visits">
          <div class="stat-num" id="stat-visits">—</div>
          <div class="stat-label">訪問</div>
        </div>
        <div class="stat-item s-meeting">
          <div class="stat-num" id="stat-meetings">—</div>
          <div class="stat-label">会議</div>
        </div>
        <div class="stat-item">
          <div class="stat-num" id="stat-total">—</div>
          <div class="stat-label">合計</div>
        </div>
        <div class="stat-item s-done">
          <div class="stat-num" id="stat-done">0</div>
          <div class="stat-label">完了</div>
        </div>
      </div>
    </header>
  ```

  Replace with:
  ```html
      <div class="header-stats">
        <div class="stat-item s-visits">
          <div class="stat-num" id="stat-visits">—</div>
          <div class="stat-label">訪問</div>
        </div>
        <div class="stat-item s-meeting">
          <div class="stat-num" id="stat-meetings">—</div>
          <div class="stat-label">会議</div>
        </div>
        <div class="stat-item">
          <div class="stat-num" id="stat-total">—</div>
          <div class="stat-label">合計</div>
        </div>
        <div class="stat-item s-done">
          <div class="stat-num" id="stat-done">0</div>
          <div class="stat-label">完了</div>
        </div>
      </div>
      <span class="save-indicator hidden" id="save-indicator"></span>
      <div class="user-menu" id="user-menu" style="display:none;">
        <button class="user-btn" id="user-btn" onclick="toggleUserMenu()">
          <img class="user-avatar" id="user-avatar" src="" alt="">
          <span id="user-name"></span>
          <span style="font-size:9px;margin-left:2px;">▾</span>
        </button>
        <div class="user-dropdown" id="user-dropdown" style="display:none;">
          <button onclick="logout()">ログアウト</button>
        </div>
      </div>
    </header>
  ```

- [ ] **Step 4-2: Verify header renders**

  Open `schedule_v4F.html` in browser. The login screen shows. The header (behind it) has the new elements — you can verify by temporarily setting `#login-screen { display:none }` in DevTools.

- [ ] **Step 4-3: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): add save indicator and user menu to header"
  ```

---

## Task 5: Replace `saveData()` with Debounced Firestore Version

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — lines 1298–1318 (localStorage section)

- [ ] **Step 5-1: Replace `saveData()` and add helpers**

  Find this exact block (lines 1298–1318):
  ```javascript
    // ── LocalStorage ──────────────────────────────────────
    function todayStr() {
      const d = new Date();
      return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
    }

    function saveData() {
      const saved = {
        saveDate:  todayStr(),
        morning:   data.today.filter(t => t.slot === 'morning'),
        afternoon: data.today.filter(t => t.slot === 'afternoon'),
        anytime:   data.today.filter(t => t.slot === 'anytime'),
        ongoing:   data.ongoing,
        overdue:   data.overdue,
        future:    data.future,
        holidays:  data.holidays || {},
        recurring: data.recurring || {},
        nextId,
      };
      localStorage.setItem(STORAGE_KEY, JSON.stringify(saved));
    }
  ```

  Replace with:
  ```javascript
    // ── Storage (Firestore) ───────────────────────────────
    let currentUser = null;
    let _saveTimer  = null;

    function todayStr() {
      const d = new Date();
      return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
    }

    function buildSavePayload() {
      return {
        lastWriteTime: Date.now(),
        saveDate:  todayStr(),
        morning:   data.today.filter(t => t.slot === 'morning'),
        afternoon: data.today.filter(t => t.slot === 'afternoon'),
        anytime:   data.today.filter(t => t.slot === 'anytime'),
        ongoing:   data.ongoing,
        overdue:   data.overdue,
        future:    data.future,
        holidays:  data.holidays || {},
        recurring: data.recurring || {},
        nextId,
      };
    }

    function saveData() {
      showSaveIndicator('pending');
      clearTimeout(_saveTimer);
      _saveTimer = setTimeout(() => _flushSave(), 800);
    }

    async function _flushSave() {
      if (!currentUser) return;
      try {
        await StorageAdapter.save(currentUser.uid, buildSavePayload());
        showSaveIndicator('ok');
      } catch (err) {
        console.error('Firestore save failed:', err);
        showSaveIndicator('error');
      }
    }
  ```

- [ ] **Step 5-2: Open `schedule_v4F.html` in browser and verify no console errors**

  At this point the Firebase SDK is not yet loaded, so `StorageAdapter` is undefined. But `saveData()` only delegates to `_flushSave()` after 800ms, and `_flushSave()` checks `if (!currentUser) return`. So no crash occurs. Console should be clean.

  Open browser DevTools → Console → drag file into browser → expected: no errors.

- [ ] **Step 5-3: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): replace saveData with debounced Firestore version"
  ```

---

## Task 6: Replace `loadData()` with Async Firestore Version

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — lines 1320–1357 (`loadData` function)

- [ ] **Step 6-1: Replace `loadData()`**

  Find this exact block (lines 1320–1357):
  ```javascript
    function loadData() {
      const raw   = localStorage.getItem(STORAGE_KEY);
      const today = todayStr();
      if (!raw) { data = { today:[], ongoing:[], overdue:[], future:[], holidays:{}, recurring:{ yamanakajuku:{type:'nth-weekday',n:3,dow:5}, bushokai_shacho:{}, bushokai_dentatsu:{}, bushokai_kyotaku:{}, tomita:{}, keiei_1:{}, keiei_2:{}, keiei:{} } }; return; }
      const saved = JSON.parse(raw);
      nextId = saved.nextId || 300;
      data.future = saved.future || [];
      data.holidays = saved.holidays || {};
      data.recurring = saved.recurring || {};
      if (!data.recurring.yamanakajuku) data.recurring.yamanakajuku = { type: 'nth-weekday', n: 3, dow: 5 };
      if (!data.recurring.bushokai_shacho)   data.recurring.bushokai_shacho   = {};
      if (!data.recurring.bushokai_dentatsu) data.recurring.bushokai_dentatsu = {};
      if (!data.recurring.bushokai_kyotaku)  data.recurring.bushokai_kyotaku  = {};
      if (!data.recurring.tomita)   data.recurring.tomita   = {};
      if (!data.recurring.keiei_1) {
        data.recurring.keiei_1 = data.recurring.keiei || {};
      }
      if (!data.recurring.keiei_2) data.recurring.keiei_2 = {};
      if (!data.recurring.keiei)   data.recurring.keiei   = {};
      if (saved.saveDate === today) {
        data.today   = [...(saved.morning||[]), ...(saved.afternoon||[]), ...(saved.anytime||[])];
        data.ongoing = saved.ongoing || [];
        data.overdue = saved.overdue || [];
      } else {
        // 日付が変わった：morning/afternoon/anytime のうち未完了タスクをすべて翌日に繰り越す
        const prev = [
          ...(saved.morning   || []),
          ...(saved.afternoon || []),
          ...(saved.anytime   || []),
        ];
        data.today = prev
          .filter(t => t && t.done !== true)
          .map(t => ({ ...t, done: false }));
        data.ongoing = saved.ongoing || [];
        data.overdue = saved.overdue || [];
      }
      promoteFuture();
    }
  ```

  Replace with:
  ```javascript
    function defaultData() {
      return {
        today:[], ongoing:[], overdue:[], future:[], holidays:{},
        recurring:{
          yamanakajuku:{type:'nth-weekday',n:3,dow:5},
          bushokai_shacho:{}, bushokai_dentatsu:{}, bushokai_kyotaku:{},
          tomita:{}, keiei_1:{}, keiei_2:{}, keiei:{},
        },
      };
    }

    async function loadData() {
      const today = todayStr();
      let saved = null;
      try {
        saved = await StorageAdapter.load(currentUser.uid);
      } catch (err) {
        console.error('Firestore load failed:', err);
        showOfflineBanner();
        data = defaultData();
        return;
      }

      if (!saved) {
        data = defaultData();
        await checkLocalStorageMigration();
        return;
      }

      // Migrate old sequential nextId (< 1,000,000) to timestamp-based
      nextId = (saved.nextId && saved.nextId < 1_000_000) ? Date.now() : (saved.nextId || Date.now());

      data.future   = saved.future   || [];
      data.holidays = saved.holidays || {};
      data.recurring = saved.recurring || {};
      if (!data.recurring.yamanakajuku) data.recurring.yamanakajuku = { type: 'nth-weekday', n: 3, dow: 5 };
      if (!data.recurring.bushokai_shacho)   data.recurring.bushokai_shacho   = {};
      if (!data.recurring.bushokai_dentatsu) data.recurring.bushokai_dentatsu = {};
      if (!data.recurring.bushokai_kyotaku)  data.recurring.bushokai_kyotaku  = {};
      if (!data.recurring.tomita)   data.recurring.tomita   = {};
      if (!data.recurring.keiei_1) {
        data.recurring.keiei_1 = data.recurring.keiei || {};
      }
      if (!data.recurring.keiei_2) data.recurring.keiei_2 = {};
      if (!data.recurring.keiei)   data.recurring.keiei   = {};

      if (saved.saveDate === today) {
        data.today   = [...(saved.morning||[]), ...(saved.afternoon||[]), ...(saved.anytime||[])];
        data.ongoing = saved.ongoing || [];
        data.overdue = saved.overdue || [];
      } else {
        const prev = [
          ...(saved.morning   || []),
          ...(saved.afternoon || []),
          ...(saved.anytime   || []),
        ];
        data.today = prev
          .filter(t => t && t.done !== true)
          .map(t => ({ ...t, done: false }));
        data.ongoing = saved.ongoing || [];
        data.overdue = saved.overdue || [];
      }
      promoteFuture();
    }
  ```

- [ ] **Step 6-2: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): replace loadData with async Firestore version"
  ```

---

## Task 7: Modify `init()` and Remove Direct Call

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — lines 2752–2760 (bottom of `<script>`)

- [ ] **Step 7-1: Make `init()` async and remove direct call**

  Find this exact block at the very bottom of the script (lines 2752–2760):
  ```javascript
    function init() {
      const now = new Date();
      document.getElementById('today-date').textContent =
        `${now.getFullYear()}年${now.getMonth()+1}月${now.getDate()}日（${DAYS[now.getDay()]}）`;
      loadData();
      renderAll();
    }

    init();
    </script>
  ```

  Replace with:
  ```javascript
    async function init() {
      const now = new Date();
      document.getElementById('today-date').textContent =
        `${now.getFullYear()}年${now.getMonth()+1}月${now.getDate()}日（${DAYS[now.getDay()]}）`;
      await loadData();
      renderAll();
    }

    // init() is now called from onAuthStateChanged in the Firebase IIFE below
    </script>
  ```

- [ ] **Step 7-2: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): make init() async, remove direct call"
  ```

---

## Task 8: Add Firebase IIFE (SDK + Auth + Firestore)

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — after closing `</script>`, before `</body>`

This is the core Firebase integration. The entire Firebase setup lives in a separate async IIFE appended after the existing `</script>` tag.

- [ ] **Step 8-1: Add Firebase IIFE after `</script>`**

  Find this exact block (the last 3 lines of the file):
  ```html
    // init() is now called from onAuthStateChanged in the Firebase IIFE below
    </script>
  </body>
  </html>
  ```

  Replace with:
  ```html
    // init() is now called from onAuthStateChanged in the Firebase IIFE below
    </script>

  <!-- ── Firebase SDK + Auth + Firestore ───────────────── -->
  <script>
  // CDN load error handler
  window.addEventListener('error', (e) => {
    if (e.filename && e.filename.includes('gstatic.com')) {
      const el = document.getElementById('login-screen');
      if (el) el.querySelector('.login-sub').textContent =
        'ネットワークエラーです。接続を確認してください。';
    }
  }, true);

  (async function firebaseInit() {
    const V = '10.12.2';
    const BASE = `https://www.gstatic.com/firebasejs/${V}`;

    const { initializeApp } =
      await import(`${BASE}/firebase-app.js`);
    const { getAuth, signInWithPopup, signInWithRedirect,
            getRedirectResult, GoogleAuthProvider,
            onAuthStateChanged, signOut: fbSignOut } =
      await import(`${BASE}/firebase-auth.js`);
    const { initializeFirestore, persistentLocalCache,
            persistentMultipleTabManager, doc, getDoc, setDoc } =
      await import(`${BASE}/firebase-firestore.js`);

    // ── Firebase config (paste your config from Task 1-6) ──
    const firebaseConfig = {
      apiKey:            "PASTE_YOUR_API_KEY_HERE",
      authDomain:        "PASTE_YOUR_AUTH_DOMAIN_HERE",
      projectId:         "PASTE_YOUR_PROJECT_ID_HERE",
      storageBucket:     "PASTE_YOUR_STORAGE_BUCKET_HERE",
      messagingSenderId: "PASTE_YOUR_SENDER_ID_HERE",
      appId:             "PASTE_YOUR_APP_ID_HERE"
    };

    const app  = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    const db   = initializeFirestore(app, {
      localCache: persistentLocalCache({
        tabManager: persistentMultipleTabManager()
      })
    });
    const googleProvider = new GoogleAuthProvider();

    // ── Storage Adapter ─────────────────────────────────
    window.StorageAdapter = {
      async save(uid, payload) {
        await setDoc(doc(db, 'users', uid, 'schedule', 'data'), payload);
      },
      async load(uid) {
        const snap = await getDoc(doc(db, 'users', uid, 'schedule', 'data'));
        return snap.exists() ? snap.data() : null;
      }
    };

    // ── UI helpers ──────────────────────────────────────
    window.showLoginScreen = function() {
      document.getElementById('login-screen').style.display = 'flex';
      document.getElementById('app-screen').style.display   = 'none';
      document.getElementById('user-menu').style.display    = 'none';
    };

    window.showAppScreen = function(user) {
      document.getElementById('login-screen').style.display = 'none';
      document.getElementById('app-screen').style.display   = '';
      const menu = document.getElementById('user-menu');
      menu.style.display = '';
      const avatar = document.getElementById('user-avatar');
      const name   = document.getElementById('user-name');
      if (user.photoURL) avatar.src = user.photoURL;
      name.textContent = user.displayName ? user.displayName.split(' ')[0] : user.email;
    };

    window.showSaveIndicator = function(state) {
      const el = document.getElementById('save-indicator');
      if (!el) return;
      el.className = 'save-indicator ' + state;
      if (state === 'pending') { el.textContent = '保存中…'; }
      if (state === 'ok')      { el.textContent = '保存済み ✓'; setTimeout(() => el.classList.add('hidden'), 2000); }
      if (state === 'error')   { el.textContent = '保存失敗 ⚠'; }
    };

    window.showOfflineBanner = function() {
      const el = document.createElement('div');
      el.id = 'offline-banner';
      el.style.cssText = 'position:fixed;top:0;left:0;right:0;background:#7C3AED;color:#fff;text-align:center;padding:6px;font-size:12px;z-index:500;';
      el.textContent = 'オフラインモード — データを読み込めませんでした';
      document.body.appendChild(el);
    };

    window.toggleUserMenu = function() {
      const dd = document.getElementById('user-dropdown');
      dd.style.display = dd.style.display === 'none' ? '' : 'none';
    };

    document.addEventListener('click', (e) => {
      const menu = document.getElementById('user-menu');
      const dd   = document.getElementById('user-dropdown');
      if (dd && !menu.contains(e.target)) dd.style.display = 'none';
    });

    // ── Login / Logout ──────────────────────────────────
    window.login = async function() {
      const btn = document.getElementById('login-btn');
      const err = document.getElementById('login-error');
      btn.disabled = true;
      err.textContent = '';
      try {
        await signInWithPopup(auth, googleProvider);
      } catch (e) {
        if (['auth/popup-blocked', 'auth/popup-closed-by-user',
             'auth/cancelled-popup-request'].includes(e.code)) {
          await signInWithRedirect(auth, googleProvider);
        } else if (e.code === 'auth/unauthorized-domain') {
          err.textContent = 'このドメインはFirebaseで許可されていません。管理者に連絡してください。';
        } else if (e.code !== 'auth/popup-closed-by-user') {
          err.textContent = 'ログインに失敗しました: ' + e.message;
        }
        btn.disabled = false;
      }
    };

    window.logout = async function() {
      if (_saveTimer) { clearTimeout(_saveTimer); await _flushSave(); }
      await fbSignOut(auth);
    };

    // ── beforeunload: flush pending saves ───────────────
    window.addEventListener('beforeunload', () => {
      if (_saveTimer) { clearTimeout(_saveTimer); _flushSave(); }
    });

    // ── Capture redirect result (for iOS Safari) ────────
    await getRedirectResult(auth).catch(() => {});

    // ── Auth state driver ───────────────────────────────
    onAuthStateChanged(auth, async (user) => {
      if (user) {
        window.currentUser = user;
        showAppScreen(user);
        await init();
      } else {
        window.currentUser = null;
        showLoginScreen();
        window.data = defaultData ? defaultData() : { today:[], ongoing:[], overdue:[], future:[], holidays:{}, recurring:{} };
      }
    });

  })().catch(err => {
    console.error('Firebase init failed:', err);
    const sub = document.querySelector('.login-sub');
    if (sub) sub.textContent = '初期化に失敗しました。ページを再読み込みしてください。';
  });
  </script>

  </div><!-- /#app-screen -->
  </body>
  </html>
  ```

  **Important:** Replace all 6 `"PASTE_YOUR_..."` values with the actual config from Task 1-6.

- [ ] **Step 8-2: Remove the duplicate `</div><!-- /#app-screen -->` added in Task 3**

  In Task 3 you replaced `</body>` with:
  ```html
    </div><!-- /#app-screen -->
  </body>
  ```

  Now that the IIFE contains `</div><!-- /#app-screen -->`, find and remove the one added in Task 3 (just before `</body>` in the old position — now replaced by the script above). Verify there is only ONE `</div><!-- /#app-screen -->` in the file (it should be inside the new `<script>` block at the very bottom).

  Run in PowerShell to verify:
  ```powershell
  Select-String -Path "C:\Users\kkmh2\claude\スケジュール管理\schedule_v4F.html" -Pattern "app-screen"
  ```

  Expected output: exactly 3 matches:
  1. `<div id="app-screen" style="display:none;">`  (opening, in Task 3)
  2. `document.getElementById('app-screen')` (in showLoginScreen)
  3. `document.getElementById('app-screen')` (in showAppScreen)
  4. `</div><!-- /#app-screen -->` (closing, in IIFE at bottom)

  If you see an extra `</div><!-- /#app-screen -->` from Task 3, remove it.

- [ ] **Step 8-3: Paste real Firebase config**

  Open `schedule_v4F.html` and replace the 6 placeholder strings with your actual values from Task 1-6.

- [ ] **Step 8-4: Verify login flow in browser**

  Open `schedule_v4F.html` in Chrome (drag file into browser).
  - Expected: Login screen shows.
  - Click "Googleでログイン" → Google popup opens → sign in with an email on the allowlist.
  - Expected: Login screen disappears, app screen appears with your name in the header.
  - Open Firebase Console → Firestore Database → Expected: `users/{your-uid}/schedule/data` document does NOT exist yet (data is only saved when you make a change).
  - Add a task → wait 1 second → Expected: "保存済み ✓" indicator appears in header.
  - Open Firebase Console → Firestore → Expected: the document now exists with your data.

- [ ] **Step 8-5: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): add Firebase IIFE (SDK, Auth, Firestore, StorageAdapter)"
  ```

---

## Task 9: Add `checkLocalStorageMigration()` Function

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — just before the closing `</script>` of the original script block

- [ ] **Step 9-1: Add migration function**

  Find this exact line (just before `</script>` in the original script):
  ```javascript
    // init() is now called from onAuthStateChanged in the Firebase IIFE below
    </script>
  ```

  Replace with:
  ```javascript
    // init() is now called from onAuthStateChanged in the Firebase IIFE below

    // ── localStorage → Firestore migration ───────────────
    async function checkLocalStorageMigration() {
      try {
        if (localStorage.getItem('care-schedule-v1-migrated') === '1') return;
        const raw = localStorage.getItem('care-schedule-v1');
        if (!raw) return;
        const localData = JSON.parse(raw);
        if (!localData) return;
      } catch {
        return;
      }

      const banner = document.createElement('div');
      banner.id = 'migrate-banner';
      banner.style.cssText = [
        'position:fixed;bottom:24px;left:50%;transform:translateX(-50%);',
        'background:var(--surface);border:1.5px solid var(--accent);',
        'border-radius:12px;padding:16px 20px;z-index:300;',
        'box-shadow:0 4px 20px rgba(0,0,0,0.4);max-width:360px;width:90%;',
      ].join('');
      banner.innerHTML = `
        <p style="font-size:13px;color:var(--text);margin-bottom:12px;">
          このデバイスに保存されているデータをクラウドに移行しますか？
        </p>
        <div style="display:flex;gap:8px;justify-content:flex-end;">
          <button id="migrate-skip-btn" style="padding:6px 14px;border-radius:6px;border:1px solid var(--border);background:none;color:var(--text-2);cursor:pointer;font-size:12px;">スキップ</button>
          <button id="migrate-ok-btn" style="padding:6px 14px;border-radius:6px;border:none;background:var(--accent);color:#fff;cursor:pointer;font-size:12px;font-weight:600;">移行する</button>
        </div>
        <p id="migrate-status" style="font-size:11px;color:var(--text-2);margin-top:8px;min-height:14px;"></p>
      `;
      document.body.appendChild(banner);

      document.getElementById('migrate-skip-btn').onclick = () => banner.remove();

      document.getElementById('migrate-ok-btn').onclick = async () => {
        const status = document.getElementById('migrate-status');
        document.getElementById('migrate-ok-btn').disabled = true;
        status.textContent = '移行中…';
        try {
          const raw = localStorage.getItem('care-schedule-v1');
          const localData = JSON.parse(raw);
          // Migrate nextId to timestamp-based if old sequential
          if (localData.nextId && localData.nextId < 1_000_000) localData.nextId = Date.now();
          localData.lastWriteTime = Date.now();
          await StorageAdapter.save(currentUser.uid, localData);
          localStorage.setItem('care-schedule-v1-migrated', '1');
          localStorage.removeItem('care-schedule-v1');
          status.textContent = '移行完了！';
          setTimeout(() => {
            banner.remove();
            init().then(() => renderAll());
          }, 1200);
        } catch (err) {
          status.textContent = '移行に失敗しました。再度お試しください。';
          document.getElementById('migrate-ok-btn').disabled = false;
        }
      };
    }
    </script>
  ```

- [ ] **Step 9-2: Verify migration banner**

  To test: open DevTools → Application → Local Storage → add key `care-schedule-v1` with any JSON value (e.g. `{"saveDate":"2026-06-01","morning":[],"afternoon":[],"anytime":[],"ongoing":[],"overdue":[],"future":[],"holidays":{},"recurring":{},"nextId":300}`).

  Log out → log back in → Expected: migration banner appears at bottom of screen.
  Click "移行する" → Expected: "移行完了！" shown → banner disappears → app reloads data.
  Check Firebase Console → Firestore → Expected: data from localStorage is now in Firestore.

- [ ] **Step 9-3: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): add localStorage to Firestore migration UX"
  ```

---

## Task 10: End-to-End Testing

**Files:** None (manual testing)

- [ ] **Step 10-1: Google login — desktop Chrome**

  Open `schedule_v4F.html` in Chrome. Click "Googleでログイン". Sign in with a whitelisted account.
  Expected: App screen shown. Header shows your name. No console errors.

- [ ] **Step 10-2: Unauthorised account**

  Log out. Try signing in with a Google account NOT in the Security Rules allowlist.
  Expected: Firestore read is denied. The app shows an empty schedule (Firestore returns permission-denied, caught by the try/catch in `loadData`). No crash.

  Note: Firebase Auth itself does not block the login — the user gets a uid. But Firestore rules deny the read/write. `loadData()` catches the error and calls `showOfflineBanner()` and `defaultData()`.

- [ ] **Step 10-3: Data persistence across devices**

  On Device A (e.g., home PC): add 3 tasks. Wait for "保存済み ✓".
  On Device B (e.g., phone): open `schedule_v4F.html`, log in with the same account.
  Expected: the 3 tasks appear on Device B.

- [ ] **Step 10-4: Data isolation between users**

  Log in as User A. Add a task titled "User A's private task". Log out.
  Log in as User B. Expected: User B does NOT see "User A's private task".

- [ ] **Step 10-5: All existing features**

  Run through each feature and confirm it still works:
  - [ ] Add task (morning / afternoon / anytime)
  - [ ] Edit task title, time, note inline
  - [ ] Toggle done checkbox
  - [ ] Cycle category tag (click to rotate)
  - [ ] Cycle priority dot
  - [ ] Delete task
  - [ ] Drag task between slots (daily D&D)
  - [ ] Add ongoing task with progress bar
  - [ ] Add long-term project
  - [ ] Add future task with date
  - [ ] Switch to monthly calendar view
  - [ ] Navigate months (‹ ›)
  - [ ] Set holiday (click date cell)
  - [ ] Set recurring event dates in panel
  - [ ] Drag future task chip to another date in calendar
  - [ ] "来月にコピー" button
  - [ ] Print (Ctrl+P — check print preview looks correct)

- [ ] **Step 10-6: Offline behavior**

  Load the app (logged in). Disconnect network (DevTools → Network → Offline).
  Add a task. Expected: save indicator shows "保存中…" indefinitely (network offline).
  Reconnect network. Expected: Firestore syncs the queued write. "保存済み ✓" appears.

- [ ] **Step 10-7: Tab close within debounce window**

  Add a task. Immediately close the tab (within 800ms).
  Reopen the app. Log in. Expected: the task is present (beforeunload handler flushed the write).

  Note: this may not be 100% reliable on all browsers (beforeunload can be interrupted), but Firestore's offline queue provides a second safety net.

- [ ] **Step 10-8: iOS Safari (if available)**

  Open GitHub Pages URL on iPhone/iPad.
  Expected: tapping "Googleでログイン" either opens a popup OR redirects to Google auth page and returns to the app logged in.

---

## Task 11: Deploy

**Files:**
- Modify: `スケジュール管理/index.html`

- [ ] **Step 11-1: Copy v4F to index.html**

  ```powershell
  Copy-Item "C:\Users\kkmh2\claude\スケジュール管理\schedule_v4F.html" `
            "C:\Users\kkmh2\claude\スケジュール管理\index.html" -Force
  ```

- [ ] **Step 11-2: Commit and push**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add index.html schedule_v4F.html
  git commit -m "feat: deploy Firebase-backed schedule (v4F) as index.html"
  git push
  ```

- [ ] **Step 11-3: Verify GitHub Pages**

  Wait 2–3 minutes. Open `https://cmkurashikibranch.github.io/schedule/` in browser.
  Expected: Login screen appears. Sign in. App works as in Task 10 tests.

- [ ] **Step 11-4: Final verification on mobile**

  Open URL on phone. Log in. Add a task. Open on desktop. Confirm task appears.

---

## Self-Review

### Spec Coverage Check

| Spec Section | Task |
|---|---|
| Architecture (login screen + app screen) | Task 3 |
| Firebase SDK via dynamic import (preserves onclick=) | Task 8 |
| Firebase config (pinned v10.12.2) | Task 8 |
| StorageAdapter | Task 8 |
| Firestore Security Rules | Task 1 |
| Email allowlist | Task 1 |
| Google Sign-In with redirect fallback | Task 8 |
| `saveData()` debounced | Task 5 |
| `loadData()` async Firestore | Task 6 |
| `init()` async, called from onAuthStateChanged | Task 7 |
| Save indicator (pending/ok/error) | Task 3 (CSS) + Task 8 (showSaveIndicator) |
| User menu + logout | Task 4 (HTML) + Task 8 (logout function) |
| beforeunload flush | Task 8 |
| CDN error handler | Task 8 |
| Offline persistence (persistentLocalCache) | Task 8 |
| nextId → Date.now() migration | Task 6 |
| localStorage migration UX | Task 9 |
| promoteFuture() calling debounced saveData() | Works automatically — no change needed |
| `defaultData()` helper | Task 6 |

### Placeholder Scan

No TBD/TODO items. The only intentional placeholders are the 6 `"PASTE_YOUR_..."` strings in Task 8 — these must be replaced with real values from Task 1-6 before the app can function.

### Type / Name Consistency

- `StorageAdapter` — defined in Task 8 IIFE as `window.StorageAdapter`, referenced in Task 6 `loadData()` as `StorageAdapter` — consistent.
- `currentUser` — declared in Task 5 as `let currentUser = null`, set in Task 8 as `window.currentUser = user` — consistent (same scope, `window.x === x` in global scope).
- `defaultData()` — defined in Task 6, called in Task 8 IIFE — consistent.
- `_flushSave()` — defined in Task 5, called in Task 8 (`beforeunload`, `logout`) — consistent.
- `_saveTimer` — declared in Task 5, referenced in Task 8 — consistent.
- `checkLocalStorageMigration()` — defined in Task 9, called in Task 6 `loadData()` — consistent.
- `showSaveIndicator()` — defined in Task 8 as `window.showSaveIndicator`, called in Task 5 `saveData()` and `_flushSave()` — consistent.
- `showOfflineBanner()` — defined in Task 8 as `window.showOfflineBanner`, called in Task 6 `loadData()` — consistent.
- `toggleUserMenu()` — defined in Task 8 as `window.toggleUserMenu`, referenced in Task 4 HTML `onclick="toggleUserMenu()"` — consistent.
- `login()` / `logout()` — defined in Task 8 as `window.login` / `window.logout`, referenced in Task 3 HTML `onclick="login()"` and Task 4 HTML `onclick="logout()"` — consistent.
