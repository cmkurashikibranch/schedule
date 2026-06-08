# schedule_v4F Firebase Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate `schedule_v4.html` to `schedule_v4F.html` by replacing localStorage with Firebase Authentication (Google Sign-In) + Cloud Firestore, while preserving all existing UI and features.

**Architecture:** Copy v4 to `schedule_v4F.html`. A dynamic `import()` IIFE inside a second `<script>` tag loads Firebase SDK and exposes all auth/save helpers as `window.*` globals. A `StorageAdapter` object wraps Firestore read/write. `onAuthStateChanged` drives app initialization. `saveData()` becomes a debounced wrapper; `loadData()` becomes async. All `onclick=` handlers and the existing JS scope remain untouched.

**Tech Stack:** Firebase JS SDK v10.12.2 (CDN, dynamic import), Firebase Auth (Google Sign-In, browserLocalPersistence), Cloud Firestore (persistent local cache), Vanilla JS, GitHub Pages.

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `schedule_v4.html` | **No change** | Original localStorage version — preserved |
| `schedule_v4F.html` | **Create** | Firebase version — all changes go here |
| `index.html` | **Update** (Task 12) | Copy of v4F after all tests pass |

---

## Critical Notes Before Starting

- **Do NOT use `<script type="module">`**. The existing app has `onclick="fn()"` attribute handlers throughout. Module scope breaks these. Use dynamic `import()` inside a plain `<script>` instead.
- **Use `var currentUser`** (not `let`). In browsers, top-level `let` does NOT create `window.*` properties. `var` does. The Firebase IIFE sets `window.currentUser = user`, so `currentUser` in the existing script must be `var` to refer to the same binding.
- **Three display states**: `#auth-loading-screen` (SDK loading), `#login-screen` (logged out), `#app-screen` (logged in). This prevents the flicker of seeing the login screen on every page load for already-logged-in users.

---

## Task 1: Firebase Console Setup

**Files:** None (manual setup in Firebase Console)

- [ ] **Step 1-1: Create Firebase project**

  Go to console.firebase.google.com → "プロジェクトを追加" → Name: `care-schedule` → Continue through wizard → Create project.

- [ ] **Step 1-2: Enable Google Sign-In**

  Build → Authentication → Get started → Sign-in method → Google → Enable → support email → Save.

- [ ] **Step 1-3: Authorise GitHub Pages domain**

  Authentication → Settings → Authorised domains → Add domain:
  ```
  cmkurashikibranch.github.io
  ```

- [ ] **Step 1-4: Create Firestore database**

  Build → Firestore Database → Create database → **Production mode** → Region: `asia-northeast1` (Tokyo) → Enable.

- [ ] **Step 1-5: Set Security Rules**

  Firestore Database → Rules tab → Replace entire content with:

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
                     && request.resource.data.nextId        is int
                     && request.resource.data.lastWriteTime is int
                     && request.resource.data.morning       is list
                     && request.resource.data.afternoon     is list
                     && request.resource.data.anytime       is list
                     && request.resource.data.morning.size()   <= 200
                     && request.resource.data.afternoon.size() <= 200
                     && request.resource.data.anytime.size()   <= 200
                     && request.resource.data.future.size()    <= 500;
      }

      match /{document=**} {
        allow read, write: if false;
      }
    }
  }
  ```

  **Replace the 3 email addresses** with the actual Gmail accounts of team members → Publish.

- [ ] **Step 1-6: Copy Firebase client config**

  Project settings (gear icon) → General → "マイアプリ" → `</>` Web → Register app → name `schedule` → Register → Copy the `firebaseConfig` object:

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

  Save this — you will paste it into `schedule_v4F.html` in Task 8.

- [ ] **Step 1-7: Verify**

  Firestore → Rules → confirm rules saved.
  Authentication → Sign-in method → Google: Enabled.
  Authentication → Settings → Authorised domains: `cmkurashikibranch.github.io` listed.

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

  Expected: same byte count as `schedule_v4.html`.

- [ ] **Step 2-3: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat: create schedule_v4F.html as base for Firebase migration"
  ```

---

## Task 3: Add Three-State Screen HTML + All CSS

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html`

The app uses three mutually exclusive screens:
- `#auth-loading-screen` — shown immediately, until Firebase resolves auth state (~500ms)
- `#login-screen` — shown when user is not logged in
- `#app-screen` — shown when user is logged in (wraps the existing header + main)

- [ ] **Step 3-1: Replace `<body>` opening and wrap existing content**

  Find this line (line 1076):
  ```html
  <body>
  ```

  Replace with:
  ```html
  <body>

    <!-- ── Auth Loading Screen (shown while Firebase SDK loads) ── -->
    <div id="auth-loading-screen">
      <div class="auth-spinner"></div>
    </div>

    <!-- ── Login Screen ──────────────────────────────────────── -->
    <div id="login-screen" style="display:none;">
      <div class="login-card">
        <div class="login-logo">
          <svg width="48" height="48" viewBox="0 0 48 48" aria-hidden="true">
            <rect width="48" height="48" rx="12" fill="rgba(129,140,248,0.15)"/>
            <text x="24" y="31" text-anchor="middle"
                  font-family="Inter,sans-serif" font-size="18" font-weight="700"
                  fill="#818CF8">SC</text>
          </svg>
        </div>
        <h1 class="login-title">スケジュール管理</h1>
        <p class="login-sub">職場のGoogleアカウントでログインしてください</p>
        <button class="login-btn" id="login-btn" type="button" onclick="login()">
          <svg class="google-icon" viewBox="0 0 24 24" aria-hidden="true">
            <path d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z" fill="#4285F4"/>
            <path d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z" fill="#34A853"/>
            <path d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l3.66-2.84z" fill="#FBBC05"/>
            <path d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z" fill="#EA4335"/>
          </svg>
          Googleでログイン
        </button>
        <p class="login-error" id="login-error" role="alert" aria-live="polite"></p>
      </div>
    </div>

    <!-- ── App Screen (existing UI wrapped) ──────────────────── -->
    <div id="app-screen" style="display:none;">
  ```

- [ ] **Step 3-2: Close `#app-screen` before `</body>`**

  Find (the last 2 lines of the file before the added IIFE script):
  ```html
  </body>
  </html>
  ```

  Replace with:
  ```html
    </div><!-- /#app-screen -->

  </body>
  </html>
  ```

- [ ] **Step 3-3: Add all new CSS**

  Find this exact line (just before `</style>`):
  ```css
    .add-memo-input::placeholder { color: var(--text-3); }
  </style>
  ```

  Replace with:
  ```css
    .add-memo-input::placeholder { color: var(--text-3); }

    /* ── Auth Loading Screen ───────────────────────────────── */
    #auth-loading-screen {
      position: fixed; inset: 0; z-index: 9999;
      display: flex; align-items: center; justify-content: center;
      background: var(--bg);
    }
    .auth-spinner {
      width: 32px; height: 32px;
      border: 3px solid rgba(129,140,248,0.25);
      border-top-color: var(--accent);
      border-radius: 50%;
      animation: spin 0.7s linear infinite;
    }
    @keyframes spin { to { transform: rotate(360deg); } }

    /* ── Login Screen ───────────────────────────────────────── */
    #login-screen {
      position: fixed; inset: 0; z-index: 9998;
      display: flex; align-items: center; justify-content: center;
      background: var(--bg);
    }
    .login-card {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 16px;
      padding: 40px 32px;
      text-align: center;
      width: min(340px, calc(100vw - 32px));
      box-shadow: 0 8px 32px rgba(0,0,0,0.4);
    }
    .login-logo { margin-bottom: 16px; }
    .login-title {
      font-size: 22px; font-weight: 700; color: var(--text);
      margin-bottom: 8px; letter-spacing: -0.02em;
    }
    .login-sub { font-size: 13px; color: var(--text-2); margin-bottom: 32px; line-height: 1.5; }
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
      min-height: 16px; line-height: 1.5;
    }

    /* ── User Menu (header right) ───────────────────────────── */
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
    .user-avatar-fallback {
      width: 22px; height: 22px; border-radius: 50%;
      background: var(--accent-bg); color: var(--accent);
      font-size: 10px; font-weight: 700;
      display: none; align-items: center; justify-content: center;
    }
    .user-dropdown {
      position: absolute; right: 0; top: calc(100% + 6px);
      background: var(--surface); border: 1px solid var(--border);
      border-radius: 8px; padding: 4px;
      box-shadow: 0 4px 16px rgba(0,0,0,0.3);
      min-width: 140px; max-width: calc(100vw - 32px); z-index: 200;
    }
    .user-dropdown-name {
      padding: 8px 12px 6px;
      font-size: 11px; color: var(--text-3);
      border-bottom: 1px solid var(--border);
      margin-bottom: 4px;
      white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
    }
    .user-dropdown button {
      display: block; width: 100%; text-align: left;
      padding: 8px 12px; background: none; border: none;
      color: var(--text); font-size: 13px; cursor: pointer;
      border-radius: 5px;
    }
    .user-dropdown button:hover { background: var(--surface-2); color: var(--pri-high); }

    /* ── Save Indicator ─────────────────────────────────────── */
    .save-indicator {
      font-family: 'JetBrains Mono', monospace;
      font-size: 11px; padding: 3px 8px; border-radius: 5px;
      transition: opacity 0.3s;
    }
    .save-indicator.pending { color: var(--text-2); }
    .save-indicator.ok      { color: #4ade80; }
    .save-indicator.error   {
      color: var(--pri-high);
      background: rgba(248,113,113,0.18);
      border: 1px solid rgba(248,113,113,0.35);
    }
    .save-indicator.hidden  { opacity: 0; pointer-events: none; }

    /* ── Offline Banner ─────────────────────────────────────── */
    .offline-banner {
      position: fixed; top: 0; left: 0; right: 0; z-index: 9997;
      background: rgba(129,140,248,0.20);
      border-bottom: 1px solid rgba(129,140,248,0.40);
      color: var(--text); font-size: 12px;
      padding: 8px 48px 8px 16px;
      display: flex; align-items: center; gap: 8px;
    }
    .offline-banner-close {
      position: absolute; right: 12px; top: 50%; transform: translateY(-50%);
      background: none; border: none; color: var(--text-2);
      cursor: pointer; font-size: 18px; line-height: 1;
    }

    /* ── Mobile adjustments ─────────────────────────────────── */
    @media (max-width: 600px) {
      #user-name { display: none; }
      .login-card { padding: 32px 24px; }
    }

    /* ── Print: hide login/loading overlays ─────────────────── */
    @media print {
      #auth-loading-screen, #login-screen { display: none !important; }
    }
  </style>
  ```

- [ ] **Step 3-4: Verify in browser**

  Open `schedule_v4F.html` in Chrome. Expected: spinning loading indicator on dark background. Login screen is hidden. App is hidden.

- [ ] **Step 3-5: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): add 3-state screens (loading/login/app) and all new CSS"
  ```

---

## Task 4: Add Save Indicator + User Menu to Header

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — `<header>` section (around line 1083)

- [ ] **Step 4-1: Add save indicator and user menu to header**

  Find this exact block:
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
        <button class="user-btn" id="user-btn" type="button" onclick="toggleUserMenu()">
          <img class="user-avatar" id="user-avatar" src="" alt=""
               onerror="this.style.display='none';document.getElementById('user-avatar-fallback').style.display='flex';">
          <span class="user-avatar-fallback" id="user-avatar-fallback"></span>
          <span id="user-name"></span>
          <span style="font-size:9px;margin-left:2px;">▾</span>
        </button>
        <div class="user-dropdown" id="user-dropdown" style="display:none;">
          <div class="user-dropdown-name" id="user-email"></div>
          <button type="button" onclick="logout()">ログアウト</button>
        </div>
      </div>
    </header>
  ```

- [ ] **Step 4-2: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): add save indicator and user menu to header"
  ```

---

## Task 5: Replace `saveData()` with Debounced Version + Add Stubs

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — lines 1298–1318 (localStorage section)

- [ ] **Step 5-1: Replace the `LocalStorage` section**

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
    // NOTE: var (not let) so that window.currentUser from the IIFE refers
    // to the same binding — top-level let does NOT create window.* properties
    var currentUser = null;
    var _saveTimer  = null;
    var _appInitialized = false;

    // No-op stubs — real implementations are set by the Firebase IIFE below.
    // These prevent crashes if saveData() fires before the IIFE has loaded.
    window.showSaveIndicator = function() {};
    window.showOfflineBanner = function() {};
    window.fbSignOutGlobal   = function() { return Promise.resolve(); };

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
      _saveTimer = setTimeout(() => _flushSave(), 300);
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

- [ ] **Step 5-2: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): replace saveData with debounced Firestore version, add stubs"
  ```

---

## Task 6: Replace `loadData()` with Async Firestore Version

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — lines 1320–1357

- [ ] **Step 6-1: Replace `loadData()` and add `defaultData()`**

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
        if (err.code === 'permission-denied') {
          // This account is not on the allowlist — sign out and show clear message
          await fbSignOutGlobal();
          const errEl = document.getElementById('login-error');
          if (errEl) errEl.textContent = 'このアカウントはアクセスできません。登録されたアカウントでログインしてください。';
          return;
        }
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

      // Migrate legacy sequential nextId (< 1,000,000) to timestamp-based
      nextId = (!saved.nextId || saved.nextId < 1_000_000) ? Date.now() : saved.nextId;

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

## Task 7: Make `init()` Async + Remove Direct Call

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — lines 2752–2760 (bottom of `<script>`)

- [ ] **Step 7-1: Replace `init()` and remove the bare call**

  Find this exact block at the bottom of the first `<script>` tag:
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

    // init() is called from onAuthStateChanged inside the Firebase IIFE (second <script> below).
    // The bare init() call has been removed to prevent a crash before the IIFE loads.
    </script>
  ```

  **Critical:** The bare `init()` call on the second-to-last line MUST be removed. If it stays, the page crashes on load because `StorageAdapter` does not exist yet.

- [ ] **Step 7-2: Verify**

  Open `schedule_v4F.html` in browser.
  Expected: spinner shows. No console errors. Page does not crash.

- [ ] **Step 7-3: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): make init() async, remove bare init() call"
  ```

---

## Task 8: Add `checkLocalStorageMigration()` Function

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — just before the closing `</script>` of the first script block

- [ ] **Step 8-1: Add migration function before `</script>`**

  Find this line (at the end of the first `<script>` block):
  ```javascript
    // init() is called from onAuthStateChanged inside the Firebase IIFE (second <script> below).
    // The bare init() call has been removed to prevent a crash before the IIFE loads.
    </script>
  ```

  Replace with:
  ```javascript
    // init() is called from onAuthStateChanged inside the Firebase IIFE (second <script> below).
    // The bare init() call has been removed to prevent a crash before the IIFE loads.

    // ── localStorage → Firestore migration ───────────────────
    async function checkLocalStorageMigration() {
      try {
        if (localStorage.getItem('care-schedule-v1-migrated') === '1') return;
        const raw = localStorage.getItem('care-schedule-v1');
        if (!raw) return;
        JSON.parse(raw); // validate JSON — throws if corrupt
      } catch {
        return;
      }

      const card = document.createElement('div');
      card.style.cssText = [
        'position:fixed;bottom:max(24px,env(safe-area-inset-bottom,0px) + 16px);',
        'left:50%;transform:translateX(-50%);',
        'background:var(--surface);border:1.5px solid var(--accent);',
        'border-radius:12px;padding:20px;z-index:8000;',
        'box-shadow:0 4px 20px rgba(0,0,0,0.4);',
        'width:min(360px,calc(100vw - 32px));',
      ].join('');
      card.innerHTML = `
        <p style="font-size:13px;color:var(--text);margin-bottom:4px;font-weight:600;">データの移行</p>
        <p style="font-size:12px;color:var(--text-2);margin-bottom:16px;line-height:1.5;">
          このデバイスに保存されているデータをクラウドに移行しますか？<br>
          ※移行後もローカルデータは保持されます
        </p>
        <div style="display:flex;gap:8px;justify-content:flex-end;">
          <button id="_mig_skip" type="button"
            style="padding:7px 14px;border-radius:6px;border:1px solid var(--border);
                   background:none;color:var(--text-2);cursor:pointer;font-size:12px;">
            今はしない
          </button>
          <button id="_mig_ok" type="button"
            style="padding:7px 14px;border-radius:6px;border:none;
                   background:var(--accent);color:#fff;cursor:pointer;
                   font-size:12px;font-weight:600;">
            クラウドに移行する
          </button>
        </div>
        <p id="_mig_status" style="font-size:11px;color:var(--text-2);margin-top:10px;min-height:14px;"></p>
      `;
      document.body.appendChild(card);

      document.getElementById('_mig_skip').onclick = () => card.remove();

      document.getElementById('_mig_ok').onclick = async () => {
        const okBtn    = document.getElementById('_mig_ok');
        const skipBtn  = document.getElementById('_mig_skip');
        const status   = document.getElementById('_mig_status');
        okBtn.disabled = true; skipBtn.disabled = true;
        status.textContent = '移行中…';
        try {
          const raw = localStorage.getItem('care-schedule-v1');
          const localData = JSON.parse(raw);
          // Migrate nextId to timestamp-based if old sequential
          if (!localData.nextId || localData.nextId < 1_000_000) {
            localData.nextId = Date.now();
          }
          localData.lastWriteTime = Date.now();
          // Save to Firestore FIRST — only remove localStorage after success
          await StorageAdapter.save(currentUser.uid, localData);
          localStorage.setItem('care-schedule-v1-migrated', '1');
          status.textContent = '移行が完了しました';
          setTimeout(async () => {
            card.remove();
            await init();
            renderAll();
          }, 1200);
        } catch (err) {
          status.textContent = '移行に失敗しました。再度お試しください。';
          status.style.color = 'var(--pri-high)';
          okBtn.disabled = false; skipBtn.disabled = false;
        }
      };
    }
    </script>
  ```

- [ ] **Step 8-2: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): add localStorage to Firestore migration UX"
  ```

---

## Task 9: Add Firebase IIFE (Second `<script>` Block)

**Files:**
- Modify: `スケジュール管理/schedule_v4F.html` — between `</div><!-- /#app-screen -->` and `</body>`

This is the core Firebase integration. It lives in a separate `<script>` tag after the existing one, ensuring no conflicts with the existing code.

- [ ] **Step 9-1: Add the Firebase IIFE script**

  Find this exact block at the very end of the file:
  ```html
    </div><!-- /#app-screen -->

  </body>
  </html>
  ```

  Replace with:
  ```html
    </div><!-- /#app-screen -->

  <!-- ── Firebase SDK + Auth + Firestore ────────────────────── -->
  <script>
  // Catch failed dynamic imports (network error / CDN blocked)
  window.addEventListener('unhandledrejection', function(event) {
    var msg = (event.reason && event.reason.message) ? event.reason.message : String(event.reason);
    if (msg.indexOf('gstatic.com') !== -1 || msg.indexOf('Failed to fetch') !== -1) {
      var spinner = document.getElementById('auth-loading-screen');
      if (spinner) spinner.innerHTML =
        '<div style="color:var(--text-2);text-align:center;padding:20px;font-size:13px;">' +
        'ネットワークに接続できません。<br>接続を確認して再読み込みしてください。<br><br>' +
        '<button onclick="location.reload()" style="padding:8px 16px;border-radius:8px;' +
        'background:var(--accent);border:none;color:#fff;cursor:pointer;">再読み込み</button></div>';
      event.preventDefault();
    }
  });

  (async function firebaseInit() {
    var V    = '10.12.2';
    var BASE = 'https://www.gstatic.com/firebasejs/' + V;

    var appMod   = await import(BASE + '/firebase-app.js');
    var authMod  = await import(BASE + '/firebase-auth.js');
    var storeMod = await import(BASE + '/firebase-firestore.js');

    var initializeApp           = appMod.initializeApp;
    var getAuth                 = authMod.getAuth;
    var setPersistence          = authMod.setPersistence;
    var browserLocalPersistence = authMod.browserLocalPersistence;
    var signInWithPopup         = authMod.signInWithPopup;
    var signInWithRedirect      = authMod.signInWithRedirect;
    var getRedirectResult        = authMod.getRedirectResult;
    var GoogleAuthProvider      = authMod.GoogleAuthProvider;
    var onAuthStateChanged      = authMod.onAuthStateChanged;
    var signOut                 = authMod.signOut;
    var initializeFirestore     = storeMod.initializeFirestore;
    var persistentLocalCache    = storeMod.persistentLocalCache;
    var persistentMultipleTabManager = storeMod.persistentMultipleTabManager;
    var doc                     = storeMod.doc;
    var getDoc                  = storeMod.getDoc;
    var setDoc                  = storeMod.setDoc;

    // ── Firebase config (replace with real values from Task 1-6) ──
    var firebaseConfig = {
      apiKey:            "PASTE_YOUR_API_KEY_HERE",
      authDomain:        "PASTE_YOUR_AUTH_DOMAIN_HERE",
      projectId:         "PASTE_YOUR_PROJECT_ID_HERE",
      storageBucket:     "PASTE_YOUR_STORAGE_BUCKET_HERE",
      messagingSenderId: "PASTE_YOUR_SENDER_ID_HERE",
      appId:             "PASTE_YOUR_APP_ID_HERE"
    };

    var app  = initializeApp(firebaseConfig);
    var auth = getAuth(app);
    var db   = initializeFirestore(app, {
      localCache: persistentLocalCache({
        tabManager: persistentMultipleTabManager()
      })
    });
    var googleProvider = new GoogleAuthProvider();

    // Persist login across browser sessions (survive tab close / restart)
    await setPersistence(auth, browserLocalPersistence);

    // ── Storage Adapter ──────────────────────────────────────
    window.StorageAdapter = {
      save: async function(uid, payload) {
        await setDoc(doc(db, 'users', uid, 'schedule', 'data'), payload);
      },
      load: async function(uid) {
        var snap = await getDoc(doc(db, 'users', uid, 'schedule', 'data'));
        return snap.exists() ? snap.data() : null;
      }
    };

    // ── Expose sign-out for loadData() error handling ────────
    window.fbSignOutGlobal = async function() { await signOut(auth); };

    // ── UI helpers ────────────────────────────────────────────
    function getAuthErrorMessage(err) {
      var map = {
        'auth/network-request-failed':
          'ネットワークに接続できませんでした。Wi-Fiまたはモバイルデータをご確認ください。',
        'auth/popup-blocked':
          'ポップアップがブロックされました。ブラウザの設定でポップアップを許可してください。',
        'auth/popup-closed-by-user':
          'ログインがキャンセルされました。もう一度お試しください。',
        'auth/cancelled-popup-request':
          'ログインがキャンセルされました。もう一度お試しください。',
        'auth/unauthorized-domain':
          'このURLからのログインは許可されていません。管理者にお問い合わせください。',
        'auth/user-disabled':
          'このアカウントは無効になっています。管理者にお問い合わせください。',
      };
      return map[err.code] || 'ログインに失敗しました。しばらく待ってからもう一度お試しください。';
    }

    window.showSaveIndicator = function(state) {
      var el = document.getElementById('save-indicator');
      if (!el) return;
      el.className = 'save-indicator ' + state;
      if (state === 'pending') { el.textContent = '保存中…'; }
      if (state === 'ok')      { el.textContent = '保存済み'; setTimeout(function(){ el.classList.add('hidden'); }, 2000); }
      if (state === 'error')   { el.textContent = '保存失敗'; }
    };

    window.showOfflineBanner = function() {
      if (document.getElementById('offline-banner')) return;
      var banner = document.createElement('div');
      banner.id = 'offline-banner';
      banner.className = 'offline-banner';
      banner.innerHTML =
        'オフライン中 — 接続が回復すると自動的に更新されます' +
        '<button class="offline-banner-close" type="button" ' +
        'onclick="document.getElementById(\'offline-banner\').remove()" ' +
        'aria-label="閉じる">×</button>';
      document.body.appendChild(banner);
      window.addEventListener('online', function() {
        var b = document.getElementById('offline-banner');
        if (b) b.remove();
      }, { once: true });
    };

    window.showLoginScreen = function() {
      document.getElementById('auth-loading-screen').style.display = 'none';
      document.getElementById('login-screen').style.display        = 'flex';
      document.getElementById('app-screen').style.display          = 'none';
      document.getElementById('user-menu').style.display           = 'none';
    };

    window.showAppScreen = function(user) {
      document.getElementById('auth-loading-screen').style.display = 'none';
      document.getElementById('login-screen').style.display        = 'none';
      document.getElementById('app-screen').style.display          = '';
      var menu = document.getElementById('user-menu');
      menu.style.display = '';
      var avatar    = document.getElementById('user-avatar');
      var fallback  = document.getElementById('user-avatar-fallback');
      var nameEl    = document.getElementById('user-name');
      var emailEl   = document.getElementById('user-email');
      if (user.photoURL) {
        avatar.src = user.photoURL;
      } else {
        avatar.style.display = 'none';
        fallback.style.display = 'flex';
        fallback.textContent = (user.displayName || user.email || 'U').charAt(0).toUpperCase();
      }
      nameEl.textContent  = user.displayName ? user.displayName.split(' ')[0] : '';
      emailEl.textContent = user.email || '';
    };

    window.toggleUserMenu = function() {
      var dd = document.getElementById('user-dropdown');
      dd.style.display = dd.style.display === 'none' ? '' : 'none';
    };

    document.addEventListener('click', function(e) {
      var menu = document.getElementById('user-menu');
      var dd   = document.getElementById('user-dropdown');
      if (dd && menu && !menu.contains(e.target)) dd.style.display = 'none';
    });

    // ── Login ────────────────────────────────────────────────
    window.login = async function() {
      var btn    = document.getElementById('login-btn');
      var errEl  = document.getElementById('login-error');
      btn.disabled = true;
      errEl.textContent = '';
      try {
        await signInWithPopup(auth, googleProvider);
      } catch (e) {
        if (['auth/popup-blocked','auth/popup-closed-by-user',
             'auth/cancelled-popup-request'].includes(e.code)) {
          try {
            await signInWithRedirect(auth, googleProvider);
            return; // redirect navigates away; no need to re-enable button
          } catch (e2) {
            errEl.textContent = getAuthErrorMessage(e2);
          }
        } else if (e.code !== 'auth/popup-closed-by-user') {
          errEl.textContent = getAuthErrorMessage(e);
        }
        btn.disabled = false;
      }
    };

    // ── Logout ───────────────────────────────────────────────
    window.logout = async function() {
      if (!confirm('ログアウトしますか？\n保存されていない変更は失われます。')) return;
      if (_saveTimer) { clearTimeout(_saveTimer); await _flushSave(); }
      await signOut(auth);
      // localStorage cleanup for shared devices
      try { localStorage.removeItem('care-schedule-v1'); } catch(_) {}
    };

    // ── Flush on tab/window close ────────────────────────────
    window.addEventListener('beforeunload', function() {
      if (_saveTimer) { clearTimeout(_saveTimer); _flushSave(); }
    });

    // ── Capture redirect sign-in result (for iOS Safari) ─────
    await getRedirectResult(auth).catch(function(err) {
      if (err.code && err.code !== 'auth/popup-closed-by-user') {
        console.error('getRedirectResult:', err.code);
      }
    });

    // ── Auth state driver ─────────────────────────────────────
    onAuthStateChanged(auth, async function(user) {
      if (user) {
        currentUser = user; // var at top-level = window.currentUser
        showAppScreen(user);
        if (!_appInitialized) {
          _appInitialized = true;
          await init();
        }
      } else {
        _appInitialized = false;
        if (_saveTimer) { clearTimeout(_saveTimer); _saveTimer = null; }
        currentUser = null;
        showLoginScreen();
        data = defaultData();
      }
    });

  })().catch(function(err) {
    console.error('Firebase initialization failed:', err);
    var el = document.getElementById('auth-loading-screen');
    if (el) el.innerHTML =
      '<div style="color:var(--text-2);text-align:center;padding:20px;font-size:13px;">' +
      '初期化に失敗しました。<br>ページを再読み込みしてください。<br><br>' +
      '<button onclick="location.reload()" style="padding:8px 16px;border-radius:8px;' +
      'background:var(--accent);border:none;color:#fff;cursor:pointer;">再読み込み</button></div>';
  });
  </script>

  </body>
  </html>
  ```

  **Important:** Replace all 6 `"PASTE_YOUR_..."` values with the actual config from Task 1-6.

- [ ] **Step 9-2: Paste real Firebase config**

  Open `schedule_v4F.html`, find the `firebaseConfig` object, and replace the 6 placeholder strings with real values.

- [ ] **Step 9-3: Verify login flow**

  Open `schedule_v4F.html` in Chrome.
  1. Expected: spinner shown for ~0.5s, then login screen appears.
  2. Click "Googleでログイン" → Google popup → sign in with allowlisted email.
  3. Expected: spinner briefly, then app screen. Name shown in header.
  4. Add a task. Wait 1s. Expected: "保存済み" in header.
  5. Open Firebase Console → Firestore → Expected: `users/{uid}/schedule/data` document exists.

- [ ] **Step 9-4: Commit**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add schedule_v4F.html
  git commit -m "feat(v4F): add Firebase IIFE (Auth, Firestore, auth state management)"
  ```

---

## Task 10: End-to-End Testing

**Files:** None (manual testing)

- [ ] **Step 10-1: Google login — desktop Chrome**

  Open file in Chrome. Expected: spinner → login screen → Google popup → app. No console errors.

- [ ] **Step 10-2: Already logged in (second visit)**

  Close and reopen `schedule_v4F.html`. Expected: spinner briefly (~500ms), then app screen directly — login screen should NOT flash. This verifies the 3-state design and `browserLocalPersistence` work correctly.

- [ ] **Step 10-3: Unauthorised account**

  Log out. Sign in with a Google account NOT in the Security Rules email list.
  Expected: the app loads, then immediately shows the login screen with the message "このアカウントはアクセスできません。登録されたアカウントでログインしてください。". The user is automatically signed out. NOT "オフラインモード".

- [ ] **Step 10-4: Data persistence across devices**

  On Device A: add 3 tasks. Wait for "保存済み". On Device B: open app, log in with same account. Expected: 3 tasks visible.

- [ ] **Step 10-5: Data isolation between users**

  Log in as User A. Add task "ユーザーAの専用タスク". Log out. Log in as User B. Expected: "ユーザーAの専用タスク" is NOT visible.

- [ ] **Step 10-6: All existing features**

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
  - [ ] Print (Ctrl+P — check print preview shows no login overlay)

- [ ] **Step 10-7: Offline behavior**

  Load app (logged in). DevTools → Network → Offline. Add a task. Expected: "保存中…" stays in header.
  Reconnect. Expected: pending write syncs. "保存済み" appears. Refresh page. Expected: task is there.

- [ ] **Step 10-8: Logout confirmation**

  Click user menu → "ログアウト". Expected: browser `confirm()` dialog appears. Click "キャンセル". Expected: NOT logged out. Click again → "OK". Expected: logged out, login screen shown.

- [ ] **Step 10-9: Mobile (iOS Safari if available)**

  Open GitHub Pages URL on iPhone. Expected: tap "Googleでログイン" → either popup or redirect to Google auth → returns to app logged in.

---

## Task 11: localStorage Migration Test

**Files:** None (manual testing)

- [ ] **Step 11-1: Simulate existing localStorage data**

  In DevTools → Application → Local Storage → add:
  - Key: `care-schedule-v1`
  - Value: `{"saveDate":"2026-06-01","morning":[{"id":300,"title":"テスト移行タスク","category":"admin","priority":"mid","time":"09:00","slot":"morning","done":false}],"afternoon":[],"anytime":[],"ongoing":[],"overdue":[],"future":[],"holidays":{},"recurring":{"yamanakajuku":{"type":"nth-weekday","n":3,"dow":5}},"nextId":301}`

- [ ] **Step 11-2: Log out and log back in**

  Expected: migration card appears at bottom of screen with "データの移行" title.

- [ ] **Step 11-3: Click "クラウドに移行する"**

  Expected: "移行中…" → "移行が完了しました" → card disappears → "テスト移行タスク" appears in morning section.
  Open Firebase Console → Firestore → Expected: task is in the Firestore document.

- [ ] **Step 11-4: Log out and in again**

  Expected: migration card does NOT appear again (migration-done flag set).

---

## Task 12: Deploy

**Files:**
- Modify: `スケジュール管理/index.html`

- [ ] **Step 12-1: Copy v4F to index.html**

  ```powershell
  Copy-Item "C:\Users\kkmh2\claude\スケジュール管理\schedule_v4F.html" `
            "C:\Users\kkmh2\claude\スケジュール管理\index.html" -Force
  ```

- [ ] **Step 12-2: Commit and push**

  ```powershell
  cd "C:\Users\kkmh2\claude\スケジュール管理"
  git add index.html schedule_v4F.html
  git commit -m "feat: deploy Firebase-backed schedule (v4F) as index.html"
  git push
  ```

- [ ] **Step 12-3: Verify GitHub Pages**

  Wait 2–3 minutes. Open `https://cmkurashikibranch.github.io/schedule/` in browser.
  Expected: spinner → login screen → sign in → app works as in Task 10 tests.

- [ ] **Step 12-4: Final cross-device verification**

  Open on phone. Log in. Add a task. Open on desktop. Confirm task appears.

---

## Self-Review

### Spec Coverage Check

| Requirement | Task |
|---|---|
| Google Sign-In login | Task 9 |
| 3-state display (no flicker on return visit) | Task 3, Task 9 |
| browserLocalPersistence (survive tab close) | Task 9 |
| Firestore data per user | Task 9 (StorageAdapter) |
| Security Rules with email allowlist + validation | Task 1 |
| `var currentUser` (window binding fix) | Task 5 |
| No-op stubs (prevent crash before IIFE loads) | Task 5 |
| `#app-screen` closing div in raw HTML (not JS string) | Task 3 |
| Debounced saveData (300ms) | Task 5 |
| async loadData from Firestore | Task 6 |
| `defaultData()` as hoisted function declaration | Task 6 |
| `permission-denied` → sign out + Japanese message | Task 6 |
| `_appInitialized` flag (prevent double init()) | Task 5, Task 9 |
| bare `init()` call removed | Task 7 |
| Redirect fallback for iOS Safari | Task 9 |
| Japanese error messages | Task 9 |
| Logout with confirm() dialog + pending flush | Task 9 |
| Avatar fallback on image load error | Task 4 |
| Email shown in dropdown | Task 4 |
| beforeunload flush | Task 9 |
| CDN error → unhandledrejection handler | Task 9 |
| Offline banner with × button + auto-hide on reconnect | Task 9 |
| Print CSS: hide login/loading overlays | Task 3 |
| Mobile: login card width min() | Task 3 |
| Mobile: hide user name on narrow screens | Task 3 |
| localStorage migration UX (save-then-flag-then-remove) | Task 8 |
| nextId migration guard | Task 6 |
| Array size bounds in Security Rules | Task 1 |

### Placeholder Scan

The only intentional placeholders are the 6 `"PASTE_YOUR_..."` strings in Task 9. These must be replaced with real Firebase config values before the app can authenticate. All other steps contain actual code.

### Type/Name Consistency

- `StorageAdapter` — defined in Task 9 IIFE as `window.StorageAdapter`, referenced in Task 6 and Task 8 as `StorageAdapter` — consistent (window global).
- `currentUser` — declared as `var currentUser = null` in Task 5, set as `currentUser = user` in Task 9 IIFE — consistent (`var` creates `window.*` property, so assignment in IIFE works).
- `_saveTimer` / `_flushSave` — declared `var` in Task 5, referenced in Task 9 — consistent.
- `_appInitialized` — declared `var` in Task 5, set in Task 9 — consistent.
- `defaultData()` — defined as `function defaultData()` (hoisted) in Task 6, called in Task 9 IIFE — consistent.
- `showSaveIndicator` / `showOfflineBanner` — defined as stubs in Task 5, overwritten in Task 9 IIFE — consistent.
- `fbSignOutGlobal` — stub in Task 5, real impl in Task 9 as `window.fbSignOutGlobal`, called in Task 6 — consistent.
- `checkLocalStorageMigration()` — defined in Task 8, called in Task 6 `loadData()` — consistent.
- `login()` / `logout()` / `toggleUserMenu()` — defined in Task 9 as `window.*`, referenced in Task 3 HTML `onclick=` — consistent.
