# Trade Show Tracker — Firebase / Firestore setup

The app now supports shared, multi-user storage via **Firestore**. Until you paste
real Firebase config into `index.html`, it stays in **local-only mode** (data lives
in one browser only). The badge in the header shows which mode you're in:

- `○ Local only` — not shared yet
- `● Shared (cloud)` — synced to Firestore across all users

---

## 1. Create the Firebase project (~5 min)

1. Go to <https://console.firebase.google.com> → **Add project** (e.g. `acteon-tradeshow`).
   You can disable Google Analytics.
2. In the left nav: **Build → Firestore Database → Create database**.
   - Start in **test mode** for now (we lock it down in step 4).
   - Pick a location near you (e.g. `eur3` / `us-central`). This can't be changed later.
3. Project Overview → click the **Web** icon (`</>`) to "Add an app".
   - Give it a nickname, **don't** check Firebase Hosting yet.
   - Copy the `firebaseConfig = { ... }` object it shows you.

## 2. Paste the config

In `index.html`, find the `window.FIREBASE_CONFIG = { ... }` block near the top of
the `<body>` and replace the `PASTE_*` placeholder values with the real ones.
Reload — the header badge should flip to **● Shared (cloud)**.

> These keys are **safe to commit/share**. They identify the project, not authorize
> access. Access is controlled entirely by the security rules below.

## 3. Host the app at a URL (required for cloud mode)

Firebase needs `http(s)://`, so opening the file directly from OneDrive
(`file:///...`) will not work in cloud mode. Easiest option — **Firebase Hosting** (free):

```bash
npm install -g firebase-tools
firebase login
cd "path/to/Trade Show Tracker"
firebase init hosting     # choose existing project; public dir = ".", single-page app = No
firebase deploy
```

You'll get a URL like `https://acteon-tradeshow.web.app` to share with the team.
(Alternatives: Azure Static Web Apps, or upload to an internal SharePoint site.)

## 4. Security rules

In Firestore → **Rules** tab, replace the contents and **Publish**.

### Phase A — quick start (any signed-in OR open, internal use)
Test mode is open to the world and **expires after 30 days**. For a short-lived
internal pilot you can lock it to a shared secret-free, link-only model, but the
right move is Phase B. Minimum viable while you wire auth:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // TEMPORARY: anyone with the app URL can read/write. Replace with Phase B asap.
    match /{document=**} {
      allow read, write: if true;
    }
  }
}
```

### Phase B — lock to Acteon sign-in (recommended)
Once authentication is enabled (see step 5), restrict to signed-in Acteon users:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function acteon() {
      return request.auth != null
        && request.auth.token.email.matches('.*@acteongroup[.]com$');
    }
    match /shows/{showId}   { allow read, write: if acteon(); }
    match /images/{imageId} { allow read, write: if acteon(); }
    match /meta/{docId}     { allow read, write: if acteon(); }
  }
}
```

## 5. Authentication — Microsoft SSO (the sign-in gate is already built in)

The "Sign in with Microsoft" screen is **already coded** in `index.html` but stays
**dormant** until you flip `window.REQUIRE_SIGN_IN` to `true`. Steps to activate:

1. Firebase → **Build → Authentication → Get started**.
2. **Sign-in method → Add new provider → Microsoft.** Enable it.
3. It asks for an Azure **Application (client) ID** and **secret** — register an app in
   the Azure portal (Azure AD → App registrations → New). Copy the **redirect URI**
   Firebase shows you into the Azure app's "Web" redirect URIs.
4. Firebase → Authentication → **Settings → Authorized domains** — make sure your
   hosting domain is listed (`*.web.app` / `*.firebaseapp.com` are added automatically;
   add a custom/SharePoint domain manually if you host elsewhere).
5. In `index.html`, set:
   ```js
   window.REQUIRE_SIGN_IN = true;
   window.ACTEON_TENANT_ID = "<your Azure AD tenant GUID>"; // or leave "common"
   ```
   Setting the tenant GUID restricts the popup to the Acteon directory only.
6. Publish the **Phase-B** Firestore rules (step 4) so only signed-in Acteon users can
   read/write. Redeploy (`firebase deploy`).

After this, anyone hitting the app sees a sign-in screen first; once signed in their
name + a **Sign out** button appear in the header.

> Simpler-but-less-native alternative: enable **Google** sign-in instead (no Azure
> work) — but then change the provider id in `index.html` from `'microsoft.com'` to a
> `GoogleAuthProvider`, and your team would need Google accounts. Microsoft is the
> better fit for Acteon.

## 6. App Check (recommended for the open-rules pilot) — block bots, no user login

App Check verifies that requests come from *your* app, which closes the
"a bot could wipe or hammer the open database" risk **without making anyone sign in**.
It's invisible to users (reCAPTCHA v3 runs silently; only a small badge appears).
The code is already wired in `index.html` — it stays off until you add a site key.

1. Firebase → **Build → App Check**. Click your web app → **Register**.
2. Choose **reCAPTCHA v3** as the provider. It generates (or asks for) a **site key**.
   - If it sends you to the reCAPTCHA admin to create a key, pick **reCAPTCHA v3**,
     add your hosting domain(s) (e.g. `trade-show-tracker-1b35c.web.app`), and copy the
     **site key** (the public one, not the secret).
3. In `index.html` set `window.APPCHECK_SITE_KEY = "<your reCAPTCHA v3 site key>";`
4. **Local testing:** to test on `localhost`/the file preview, temporarily set
   `window.APPCHECK_DEBUG = true`, open the app, copy the debug token printed in the
   browser console, and register it under **App Check → Apps → Manage debug tokens**.
   Set it back to `false` before deploying.
5. In Firebase → App Check, App Check starts in **monitoring** mode. Watch the metrics
   for a day or two to confirm real traffic shows valid tokens, **then** switch
   Firestore to **Enforced**. (Enforcing before your domain/token is registered will
   block your own app, so do this step last.)

> App Check stops *outside* abuse but does not identify users — anyone with the app URL
> still has access under the open rules. For per-user control, use Microsoft SSO (step 5).

---

## Data model (for reference)

- `shows/{showId}` — one document per trade show (the show object and its
  `attendances` array; photos are **not** stored here).
- `images/{imageId}` — one document per photo (`{ id, showId, attId, dataUrl, caption,
  ts }`), kept separate so show docs stay under Firestore's 1 MB/doc limit.
- `meta/roster` — `{ reps: [ {id, name}, ... ] }`.

## Notes & limits

- **Images** are downscaled to ~1100 px JPEG on upload and stored one-per-document in
  the `images` collection (not inside the show doc), so a show with many photos can't
  hit the 1 MB/doc limit. The whole `images` collection is loaded on startup; for very
  photo-heavy use the scalable upgrade is **Firebase Storage** (URLs instead of inline
  data, requires the Blaze plan) — ask when needed.
- **Real-time:** edits made by one user appear for everyone within ~1 second.
- **Offline:** the app still loads from a local cache and works offline; changes sync
  when the connection returns.
