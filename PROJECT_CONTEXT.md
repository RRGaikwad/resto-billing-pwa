# 🍽️ Restaurant Billing PWA — Project Context & Agent Handoff Document

> **Purpose:** This file is the single source of truth for any AI agent working on this project.
> Read this ENTIRE file before making any changes. It will save you time, prevent repeated mistakes,
> and give you a full picture of the project's current state.

---

## 📋 Project Overview

| Property | Value |
|---|---|
| **App Name** | Restaurant Billing PWA |
| **Type** | Single-file Progressive Web App (PWA) |
| **Current Version** | v1.6.0 |
| **GitHub Repo** | https://github.com/RRGaikwad/resto-billing-pwa |
| **Deployed URL** | Deployed on Vercel (auto-deploys on every `git push` to `main`) |
| **Owner** | RRGaikwad |
| **Local Path** | `c:\Users\HP\OneDrive\Documents\Desktop\Restaurant PWA\` |

---

## 🏗️ Architecture

### Tech Stack
- **Frontend:** Pure HTML + Vanilla CSS + Vanilla JavaScript (no framework, no build step)
- **Backend/Database:** Firebase (Firestore + Firebase Auth)
- **Hosting:** Vercel (static, auto-deploy from GitHub)
- **PWA:** Service Worker (`sw.js`) + Web App Manifest (`manifest.json`)
- **PDF/Image Export:** `html2canvas` + `jsPDF` (loaded from CDN)
- **Fonts:** Google Fonts (Inter, Playfair Display, Lato)

### File Structure
```
Restaurant PWA/
├── restaurant-invoice-pwa.html   ← ENTIRE app lives here (HTML + CSS + JS all in one file)
├── sw.js                         ← Service Worker for PWA offline caching
├── manifest.json                 ← PWA install manifest
├── icon.svg                      ← App icon
├── vercel.json                   ← Vercel deployment config
└── PROJECT_CONTEXT.md            ← This file
```

### Key Design Principle
**Everything is in one file:** `restaurant-invoice-pwa.html` contains all HTML structure, all CSS styles (`<style>` tag), and all JavaScript logic (`<script>` tag). There is no build pipeline, no npm, no node_modules.

---

## 🔥 Firebase Configuration

### Services Used
1. **Firebase Authentication** — Email/Password + Google Sign-In
2. **Cloud Firestore** — Real-time database for user data sync

### Firebase SDK Version
```
firebase-app-compat.js    v10.8.1
firebase-auth-compat.js   v10.8.1
firebase-firestore-compat.js v10.8.1
```
> ⚠️ Uses **compat** (v8 API style) — do NOT mix with modular v9+ import syntax.

### Firestore Data Structure
```
Firestore Root
└── users/
    └── {userId}/                  ← One document per authenticated user
        ├── currentData: {}        ← Full form state snapshot
        ├── resPin: "1234"         ← 4-digit PIN for Restaurant Info lock
        ├── menuCategories: []     ← Array of category names
        ├── menuCatalogue: []      ← Array of saved menu items for auto-fill
        ├── lastUpdatedBy: string  ← clientId of device that last saved (for sync logic)
        └── updatedAt: timestamp
        └── invoices/              ← Sub-collection for saved invoice history
            └── {docId}/
                ├── label: string
                ├── customer: string
                ├── date: string
                ├── formData: {}   ← Full form snapshot at time of save
                └── createdAt: timestamp
```

### Firestore Security Rules
**CRITICAL — The database will not work without these rules.**
Go to Firebase Console → Firestore → Rules tab → publish:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

### Authorized Domains for Firebase Auth
Go to Firebase Console → Authentication → Settings → Authorized Domains.
The deployed Vercel domain **must** be listed here or Google Sign-In will throw `auth/unauthorized-domain`.

---

## 🔐 Security — Restaurant Info PIN

- Restaurant Info section is locked behind a **4-digit PIN**
- PIN is stored in Firestore under `users/{uid}/resPin`
- PIN is cached in memory as `userPin` variable on load — no extra Firestore read needed on unlock
- **Reset flow:** User can reset PIN via email link using Firebase Auth `sendPasswordResetEmail`
- Access is **per-account** — data is isolated by Firebase UID, not by email string

---

## 📡 Real-Time Sync Architecture

### How it works
The app uses **Firestore `onSnapshot` listeners** (not one-time `.get()` calls) for live sync.

**Main form data listener** (in `loadData()` function):
```
db.collection('users').doc(uid).onSnapshot(...)
```

**Invoice history listener** (in `openHistory()` function):
```
db.collection('users').doc(uid).collection('invoices').onSnapshot(...)
```

### clientId System (prevents cursor-jumping)
- On app load, a random `clientId` is generated: `Math.random().toString(36).substring(2, 15)`
- Every save writes `lastUpdatedBy: clientId` to Firestore
- On receiving an `onSnapshot` update, the app checks: if `lastUpdatedBy === clientId`, it **skips** restoring the form (prevents overwriting what the user is currently typing on their own device)
- If `lastUpdatedBy` belongs to a **different device**, the form is **restored from cloud data** instantly

### unsubscribe pattern
- `unsubscribeData` — stores the main form listener unsubscribe function
- `unsubscribeHistory` — stores the history panel listener unsubscribe function
- Both are cleaned up when the listener is replaced or the modal is closed

---

## 🚀 Deployment Workflow

```bash
# After making changes to any file:
git add .
git commit -m "Description of what changed"
git push
# Vercel auto-deploys within ~30 seconds
```

> ⚠️ **PowerShell note:** Do NOT use `&&` between commands in PowerShell. Use `;` instead.
> Correct: `git add .; git commit -m "msg"; git push`
> Wrong:   `git add . && git commit -m "msg" && git push`

### Service Worker Cache Busting
The PWA aggressively caches files. After deploying, users may still see the old version.
**Every time you change the HTML, also bump the version in `sw.js`:**
```js
// sw.js line 1:
const CACHE_NAME = 'restaurant-billing-v7'; // ← increment this number each deploy
```
This forces all installed PWAs to download the new version.

### Current SW Cache Version
`restaurant-billing-v14`

---

## 📝 Complete Change History & Bug Log

### v1.6.0 — Menu Categories & Edit Capabilities
**What was added:**
- Added a "Manage Categories" modal to the Settings tab to create and delete menu categories.
- Made the Menu Catalogue form highly responsive (stacking nicely on mobile displays using flex-wrap).
- Added an "Edit" button to Menu Catalogue items to update their name, prices, and category.
- Menu Catalogue items are now grouped by Category visually in the list.
- App version updated to v1.6.0
- SW cache version bumped to v14

### v1.5.0 — Menu Catalogue Feature
**What was added:**
- Added a "Menu Catalogue" modal to the Settings tab to manage preset menu items.
- Items are synced in real-time to Firestore under the `menuCatalogue` array.
- Enhanced the main invoice form with a `<datalist>` for auto-completing item names.
- When an item from the catalogue is typed or selected, its Rate, GST, and Discount auto-fill automatically.
- App version updated to v1.5.0
- SW cache version bumped to v13

### v1.4.4 — Mobile View Improvements & Preview Button Relocation
**What was added:**
- Added "👁️ Preview" button to top-right of dashboard sidebar
- Added scrollToPreview() JS function for smooth scroll to preview
- Removed "Restaurant Info" section label from sidebar
- Improved mobile view ratio with better scaling
- App version v1.4.4
- SW cache version v12

---

### v1.4.3 — Relocate Restaurant Info Edit Button to Settings Modal
**What was added:**
- Moved "Edit Restaurant Info" button from sidebar to Settings modal
- Added new "Restaurant Settings" section in Settings modal as the new home for this button
- Cleaner sidebar layout
- App version v1.4.3
- SW cache version v11

---

### v1.4.2 — More Attractive WhatsApp Share Message
**What was added:**
- Updated WhatsApp share message to be more friendly and inviting:
  - Emojis: 👋, 🍽️✨
  - More personal tone
  - Warm sign-off from "Team [Restaurant Name]"
- Clearer attachment instructions
- App version v1.4.2
- SW cache version v10

---

### v1.4.1 — Auto-Generated Invoice Number & Hidden Date/Time Fields
**What was added:**
- Auto-incrementing invoice number (stored in localStorage)
- Invoice Number, Date, and Time fields are now hidden from the sidebar form
- Invoice number automatically increments every time "New Invoice" is clicked
- Date and Time automatically set to current date/time when creating a new invoice
- App version v1.4.1
- SW cache version v9

---

### v1.4.0 — Settings Modal & Sidebar Reorganization
**What was added:**
- Dedicated settings modal accessed via ⚙️ Settings button in sidebar
- Moved History, Reset, and Logout buttons inside the settings modal for cleaner UX
- Improved sidebar layout (now only shows most essential buttons
- Beautiful, responsive styling for settings modal with icons and hover effects
- Click outside to close the settings modal
- App version updated to v1.4.0
- SW cache version bumped to v8 for proper cache busting

---

### v1.0.0 — Initial Build
- Built full single-file Restaurant Billing PWA
- Features: Invoice form, item line-items, live preview, PDF export, WhatsApp share, thermal receipt mode
- Local storage only — no cloud sync

---

### v1.1.0 — Firebase Auth & Cloud Sync
**What was added:**
- Firebase Authentication (Email/Password + Google Sign-In)
- Full-screen login overlay before accessing the app
- Firestore persistence — form state saved to `users/{uid}/currentData`
- Invoice history saved to `users/{uid}/invoices/` sub-collection

**Bug fixed:**
- `auth/unauthorized-domain` error on Google Sign-In
  - **Cause:** The Vercel deployment domain was not listed in Firebase Auth → Authorized Domains
  - **Fix:** Added the deployed domain to Firebase Console → Authentication → Settings → Authorized Domains

---

### v1.2.0 — Restaurant Info PIN Security
**What was added:**
- 4-digit PIN lock for the Restaurant Info section
- PIN stored in Firestore under `resPin` field
- "Reset via email" link for forgotten PINs
- Per-user data isolation (data stored under `users/{uid}`, not shared)

**Bugs fixed:**
1. **Edit button always showed "Create PIN" tab instead of "Enter PIN"**
   - **Cause:** PIN was not being read from Firestore on app load
   - **Fix:** `userPin` is loaded and cached from Firestore in `loadData()` on auth

2. **"💾 Save & Lock Settings" button didn't save data**
   - **Cause:** `forceSaveData()` was not being called on button click
   - **Fix:** Wired the button click to call `forceSaveData()` which immediately writes to Firestore

3. **"Error loading security settings" when clicking Edit**
   - **Cause:** Firestore read for PIN was failing silently
   - **Fix:** `userPin` is now pre-cached from the initial `loadData()` call; no separate read needed

---

### v1.3.0 — Real-Time Multi-Device Sync
**What was added:**
- Replaced all one-time `.get()` calls with real-time `onSnapshot` listeners
- `clientId` system to prevent self-overwriting when user is actively typing
- `lastUpdatedBy` field written to every Firestore save payload

**Bugs fixed:**
1. **Data not syncing across devices**
   - **Root Cause 1:** App used `.get()` (one-time fetch) — it only loaded data once on startup. Changes on another device were never received.
   - **Fix:** Switched to `onSnapshot()` which creates a persistent live connection to Firestore. Any change from any device triggers an immediate UI update.
   - **Root Cause 2:** `hasInitiallyLoaded` flag was blocking server updates after the first load. Even when another device saved new data, the flag prevented it from being applied.
   - **Fix:** Removed `hasInitiallyLoaded` entirely. The `clientId` check is sufficient — any update from a different device (`lastUpdatedBy !== clientId`) is applied immediately.

2. **Invoice History not showing saved invoices**
   - **Root Cause:** History used `.orderBy('createdAt', 'desc')` query. Firebase Firestore excludes documents from ordered queries if the sort field has a **pending server timestamp** (the timestamp is `null` for a split second after saving on the client).
   - **Fix:** Removed the `.orderBy()` from the query. Now all documents are fetched immediately and **sorted locally** in JavaScript, so new invoices appear instantly.

3. **"The Grand Kitchen" default name persisting instead of showing cloud data**
   - **Cause:** `hasInitiallyLoaded` flag was set to `true` after loading from localStorage, so when Firestore returned the real user data a moment later, the `if (!hasInitiallyLoaded)` condition blocked it.
   - **Fix:** Same as Root Cause 2 above — removed the flag entirely.

---

### v1.3.1 — Diagnostic: UID Display
**What was added:**
- Small `#app-meta` div in sidebar header showing version and UID
- Used to diagnose whether both devices are logged into the same Firebase account
- Updates to `v1.3.x | UID: xxxxxx ✓ Live` once Firestore connects successfully

---

### v1.3.2 — Firestore Error Visibility + DB Creation
**What was added:**
- Visible toast errors for Firestore read/write failures (previously silent `console.error`)
- `#app-meta` now shows `❌ DB Error: permission-denied` if rules block access
- `forceSaveData()` now shows `✓ Settings saved & synced!` toast on success

**Root Cause of ALL sync issues discovered and fixed:**
- **The Firestore database had never been created.** The Firebase project existed, Auth was working, but the Firestore database itself was never initialized via the Firebase Console.
- **Fix:** User created database via Firebase Console → Firestore → Create Database → `asia-south1` region → Production mode → Published correct security rules.
- After database creation, all sync features worked immediately and perfectly.

---

## ⚠️ Critical Gotchas for Any AI Agent

1. **One file app** — All code is in `restaurant-invoice-pwa.html`. Do not create separate `.js` or `.css` files unless the user requests a full refactor.

2. **Always bump `sw.js` cache version** when changing the HTML. Otherwise users get the old cached version and changes don't appear.

3. **Use semicolons `;` not `&&`** in PowerShell terminal commands.

4. **Firebase is compat SDK (v8 style)** — use `firebase.firestore()`, `firebase.auth()` etc. Do NOT use `import { initializeApp } from 'firebase/app'` modular syntax.

5. **`onSnapshot` not `.get()`** — All Firestore reads must use `onSnapshot` for real-time sync. Never revert to `.get()`.

6. **`orderBy` on server timestamps breaks new documents** — Do NOT use `.orderBy('createdAt')` on the invoices collection. Sort locally instead.

7. **`clientId` is session-scoped** — It is generated fresh on every page load with `Math.random()`. It is used purely for same-device loop prevention.

8. **Firestore Security Rules must be set** — Default rules deny everything. The rules in the Firebase section above must be published for the app to work.

9. **PIN is stored as plain text string** in Firestore `resPin` field. If improving security, consider hashing.

10. **Restaurant info is only editable when unlocked** — The `#restaurant-info-content` div is `display:none` by default. It is shown only after PIN verification.

---

## 🎯 Current App Status

| Feature | Status |
|---|---|
| Login (Email + Google) | ✅ Working |
| Restaurant Info PIN Lock | ✅ Working |
| Invoice Form (all fields) | ✅ Working |
| Live Preview | ✅ Working |
| PDF / JPG / WhatsApp Export | ✅ Working |
| Thermal Receipt Mode | ✅ Working |
| Save to History | ✅ Working |
| Load / Delete History | ✅ Working |
| Real-time Sync (main form) | ✅ Working |
| Real-time Sync (history panel) | ✅ Working |
| Multi-device sync | ✅ Working |
| PWA Install (Add to Home Screen) | ✅ Working |
| Offline Mode | ✅ Working (cached assets) |
| Menu Catalogue (Auto-fill) | ✅ Working |

---

## 🔮 Possible Future Improvements (Not Yet Built)

- Print button for thermal receipt
- Multiple restaurant profiles per account
- Role-based access (owner vs. cashier)
- Monthly/daily sales analytics
- PIN hashing for better security
- PIN hashing for better security
- Push notifications for new orders

---

*Last updated: 2026-07-01 | App Version: v1.6.0 | SW Cache: v14*
