# ניחוש טבלת ליגת העל · Ligat Ha'Al Table Predictor

A single-file static web app. A user signs in with Google, drags the 14 Ligat Ha'Al
teams into the order they think the table will finish, and submits. Auth **and** storage
are handled by **Firebase** (Authentication + Cloud Firestore) — same pattern as the
RSR-Fitness project. No build step, no server code.

- `index.html` — the whole app (HTML + CSS + JS, using the Firebase modular SDK from CDN)
- `README.md` — this file

## 1. Create a Firebase project

1. Go to <https://console.firebase.google.com> → **Add project** → name it `ligat-al` →
   create (you can disable Google Analytics).
2. **Add a Web app:** on the project overview click the **`</>`** (web) icon, give it a
   nickname, **Register app**. Firebase shows a `firebaseConfig` object — keep that page open.

## 2. Turn on Google Sign-In

Firebase Console → **Build → Authentication → Get started** → **Sign-in method** tab →
click **Google** → toggle **Enable**, pick a support email → **Save**.

## 3. Create the Firestore database

Firebase Console → **Build → Firestore Database → Create database** → start in
**production mode** (or test mode) → pick a location → **Enable**.

Then set rules so each signed-in user can read/write only their own prediction.
**Rules** tab → paste this → **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /predictions/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

## 4. Paste your config into `index.html`

Open `index.html`, find the `FIREBASE_CONFIG` block near the top of the `<script>`, and
replace the placeholders with the values from step 1:

```js
const FIREBASE_CONFIG = {
    apiKey:            '...',
    authDomain:        'ligat-al.firebaseapp.com',
    projectId:         'ligat-al',
    storageBucket:     'ligat-al.firebasestorage.app',
    messagingSenderId: '...',
    appId:             '...'
};
```

> These values are **not secrets** — Firebase web config is meant to be public; your
> Firestore security rules (step 3) are what protect the data.

## 5. Authorize your domain (for Google popup)

Firebase Console → **Authentication → Settings → Authorized domains**. `localhost` is
already there by default. Add your production domain when you deploy.

## 6. Run it

Google Sign-In needs an `http(s)` origin (not `file://`). From this folder:

```bash
python -m http.server 8080     # then open http://localhost:8080
# or:  npx serve .
```

Open **http://localhost:8080**, click **התחברות עם Google**, drag the teams, and press
**שמירת הניחוש**.

## The teams

The 14 teams live in the `TEAMS` array in `index.html`. **Please verify them against the
current season** — a best-effort list is filled in. Each team:

```js
{ id: 'mta', he: 'מכבי תל אביב', en: 'Maccabi Tel Aviv', ab: 'מ"ת',
  color: '#f6d500', fg: '#0a3d91', logo: '' }
```

- `logo` — leave `''` for a colored badge with the abbreviation (`ab`), or paste an image
  URL for a real crest.
- `color` / `fg` — badge background and text colors.

## Zones (the colored stripes)

`const ZONES = { champ: 1, euro: 3, relegate: 2 }` → position 1 = champion, next up to 3 =
"Europe", bottom 2 = relegation. Edit to match the real league format.

## Data shape in Firestore

Collection **`predictions`**, one document per user (doc id = Firebase `uid`):

```json
{
  "uid": "abc123",
  "name": "דנה",
  "email": "dana@gmail.com",
  "photo": "https://...",
  "season": "2026-27",
  "order": ["mta", "mhaifa", "...  all 14 team ids ..."],
  "updatedAt": 1735500000000
}
```

- `order` is the complete 14-team ranking (index 0 = champion → last = relegation).
- Each user only ever writes their own document; re-submitting overwrites it.

## Notes
- Works on mobile (touch drag) and desktop (mouse drag), plus ▲▼ buttons as a fallback.
- Login persists across sessions (Firebase `browserLocalPersistence`); the saved guess
  reloads automatically after login.
