# Retail Tracker — Security & Fix Report

This folder contains the **fixed** build. Below is exactly what was changed and what you still must do.

---

## ✅ Code fixes applied to `index.html`

| # | Problem | Fix |
|---|---------|-----|
| 1 | Loyalty settings showed the literal text `${curSym()}` instead of the currency symbol | Labels now have IDs (`lblLoyaltyEarn`, `lblLoyaltyRedeem`) and are populated with the real symbol in `loadShopProfile()` |
| 2 | `_todayKey()` vs `_todayKey_ymd()` produced **different** date strings (one UTC, one unpadded local) — could break daily-password matching across timezones | Both now return the same zero-padded **local** `YYYY-MM-DD` |
| 3 | `generateUniqueCompanyId()` used `Math.random()` 4-digit codes (only ~9,000 options → collisions) | Now uses `crypto.getRandomValues` with a 6-char unambiguous pool (~735M options) |
| 4 | `window._lastPurchaseId` global used to smuggle the purchase ID out of a transaction | Replaced with an in-scope `let purchaseId` captured inside the transaction |
| 5 | Whole objects serialized into inline `onclick` attributes via `JSON.stringify(...)` — fragile and an XSS vector | Invoice PDF/Thermal, Party "Edit Info", Staff "Edit Info", and both Summary "Share via WhatsApp" buttons now reference shared variables (`_invoiceSale`, `_partyEdit`, `_staffEdit`, `_summaryShareText`) |
| 6 | **No Content-Security-Policy** | Added a CSP `<meta>` restricting script/style/font/connect/frame/worker sources to only the CDNs + Firebase endpoints the app actually uses |
| 7 | Missing PWA files | Added `manifest.json`, `sw.js` (offline shell, never caches Firebase data), `icon-192.png`, `icon-512.png` |

All changes were validated with `node --check` (the extracted app script parses cleanly).

---

## 🚨 What YOU still must do (server side — cannot be fixed from HTML)

The biggest security gaps are **server-side** and need action in the Firebase console.

### 1. Deploy the Firestore Security Rules (highest priority)
The app trusts the client to enforce tenancy. Without rules, anyone who knows the project ID can read/write **every** shop's data.

- Open `firestore.rules` in this folder, replace the placeholder UID with your real Super Admin UID, then deploy:
  ```
  firebase deploy --only firestore:rules
  ```
- The rules lock each company's data to its members, stop owners from editing `status`/`subscriptionExpiry`/`enabledFeatures`, and hide `company_creation_secret` from everyone except the Super Admin.

### 2. Remove the "daily registration password" secret from the client
`company_creation_secret` is read in the browser and the daily password is derived client-side. Even with rules, this design is weak. The correct fix is a **Cloud Function** that validates signup server-side. If you keep the current design, at minimum ensure the rules from step 1 are live so the secret is unreadable.

### 3. Add email verification
Signup currently creates an account with no email verification. Add:
```js
await cred.user.sendEmailVerification();
```
and gate the app on `user.emailVerified`.

### 4. Review the hardcoded Firebase config
The `firebaseConfig` block points at a real project (`retail-track-d5c27`). Confirm that is intended, and restrict that project's authorized domains to your actual hosting domain.

---

## ⚠️ Known limitations left as-is (by design / need backend)

- All Firestore reads for reports happen client-side; fine for small shops but will cost more at scale.
- Service worker caches the app shell for offline use but deliberately lets Firebase's own offline queue handle data sync.

---

## Files in this folder
- `index.html` — fixed app
- `manifest.json` — PWA manifest
- `sw.js` — service worker
- `icon-192.png`, `icon-512.png` — app icons
- `firestore.rules` — deploy these rules
- `SECURITY.md` — this file
