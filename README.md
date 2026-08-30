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

## 2. Turn on sign-in methods

Firebase Console → **Build → Authentication → Get started** → **Sign-in method** tab, then
enable **both**:

- **Google** → toggle **Enable**, pick a support email → **Save**.
- **Email/Password** → toggle **Enable** → **Save**. (This powers the username login — each
  username is stored as a synthetic `username@users.ligat-al.app` address, so no real email
  is ever needed. Firebase's own uniqueness check is what rejects an already-taken username.)

### Username login

Players can register/sign in with just a **username + password** (no email):

- Username: 3–20 chars, English letters / digits / `. _ -` (case-insensitive, must be free).
- Password: **5+ chars** (Firebase's built-in minimum is 6, so the app appends a fixed
  internal salt before sending it — users only ever type 5+).
- Registration auto-accepts and signs the user in immediately; a taken username is rejected
  with "שם המשתמש כבר תפוס".

## 3. Create the Firestore database

Firebase Console → **Build → Firestore Database → Create database** → start in
**production mode** (or test mode) → pick a location → **Enable**.

Set the rules below (also saved in [`firestore.rules`](firestore.rules)). A player can
always read & write **their own** guess, but reading **anyone else's** — a single doc or
the whole collection — is blocked at the database level **until the deadline passes**
(30 Aug 2026 23:59:59 Israel time). So no one can peek at others' tables early, even via
the browser console or the raw API. The real league table lives in `meta/standings`,
readable by all signed-in players and writable only by the admin account.
**Rules** tab → paste this → **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Submission deadline: 30 Aug 2026, 23:59:59 Israel time (UTC+3) = 20:59:59 UTC.
    // Keep in sync with DEADLINE in index.html.
    function afterDeadline() {
      return request.time >
             timestamp.date(2026, 8, 30) + duration.time(20, 59, 59, 0);
    }

    match /predictions/{uid} {
      allow write: if request.auth != null && request.auth.uid == uid;
      allow read:  if request.auth != null
                   && (request.auth.uid == uid || afterDeadline());
    }

    match /meta/{docId} {
      allow read:  if request.auth != null;
      allow write: if request.auth != null
                   && request.auth.token.email == 'eldar71191@gmail.com';
    }
  }
}
```

> If you change `ADMIN_EMAIL` or `DEADLINE` in `index.html`, update the email / deadline
> in these rules to match. Note: because reading others' guesses is now blocked before the
> deadline, the leaderboard also only works after it — which is fine, since the real
> standings (and thus any score) don't exist until the season is underway.

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

## Tabs

The app has three tabs (the third only appears for the admin):

- **הניחוש שלי** — the drag-and-drop guess editor. Loads your last saved submission and
  lets you re-order and re-save until the deadline, after which it locks.
- **טבלת המובילים** (leaderboard) — ranks all signed-in players against the real league
  table. Visible to everyone signed in.
- **עדכון טבלה** (admin only) — where the admin arranges the **real** current league
  standings and saves them; everyone's leaderboard score recomputes from that.

## Admin

Only the account whose email equals `ADMIN_EMAIL` in `index.html`
(`eldar71191@gmail.com`) sees the **עדכון טבלה** tab and can write `meta/standings`.
After each round the admin drags the real table into its current order, types the round
number, and presses **שמירת טבלת הליגה**.

## Scoring

For each player, every team costs `|guessedPosition − realPosition|` penalty points; the
player's score is the **negative** sum of those penalties. So a perfect table = `0`, and
the closer to `0`, the higher the rank. Example: you put Hapoel Haifa 10th but they're
really 2nd → that team costs 8, i.e. −8 points.

**Bonus points:** each of the five bonus guesses (court champion / first to fire a
coach / most penalties / most red cards / top scorer under-or-over 19.5 goals) is worth
**+5 points** if it matches the real answer. The admin fills in the real bonus answers on the **עדכון טבלה** tab at the end of
the season and saves; they're stored on `meta/standings.bonusAnswers` and added to each
player's score. A bonus the admin hasn't set yet counts for no one.

The real table is stored in Firestore as `meta/standings`:

```json
{ "order": ["mta", "... all 14 ids in real order ..."], "round": 3,
  "season": "2026-27", "updatedAt": 1735500000000 }
```

## Notes
- Works on mobile (touch drag) and desktop (mouse drag), plus ▲▼ buttons as a fallback.
- Login persists across sessions (Firebase `browserLocalPersistence`); the saved guess
  reloads automatically after login.
