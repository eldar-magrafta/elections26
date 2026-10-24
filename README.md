# ניחושי בחירות 2026 · Elections 2026 Mandate Predictor

A single-file static web app (same pattern as the `ligatAl` project). Each user registers
with just a **username + password** (no real email, no Google), then divides a shared
budget of **120 mandates** across the 14 running מפלגות however they like, and saves. No
party's slider can push the total past 120 — if you want to give one party more, you
first have to take mandates away from another. Auth **and** storage are handled by
**Firebase** (Authentication + Cloud Firestore). No build step, no server code — just
`index.html`.

- `index.html` — the whole app (HTML + CSS + JS, Firebase modular SDK from CDN)
- `firestore.rules` — the database security rules (paste into the Firebase console)
- `README.md` — this file

## What "username + password" actually means here

Firebase's built-in auth needs a real-looking email address, so under the hood each
username is turned into a fake address like `dana123@users.elections26.app` before being
sent to Firebase. You never see this — the UI only ever asks for a username and a
password. Firebase's own "this address is already registered" check is what makes
usernames unique; there's no separate username database to manage.

## 1. Create a Firebase project

1. Go to <https://console.firebase.google.com> → **Add project** → name it e.g.
   `elections26` → create (Google Analytics isn't needed, you can skip it).
2. **Add a Web app:** on the project overview page click the **`</>`** (web) icon, give it
   a nickname, **Register app**. Firebase will show you a `firebaseConfig` object — keep
   that page open, you'll need it in step 4.

## 2. Turn on Email/Password sign-in

Firebase Console → **Build → Authentication → Get started** → **Sign-in method** tab →
enable **Email/Password** → **Save**. (This is what powers the username login — see
above for why it uses email under the hood.) You do **not** need to enable Google or any
other provider; this app only uses username/password.

### Username/password rules in this app

- Username: 3–20 characters, English letters / digits / `. _ -` (case-insensitive, must
  be free).
- Password: **5+ characters** as typed by the user (Firebase's own minimum is 6, so the
  app silently appends a fixed internal suffix before sending it to Firebase — you only
  ever need to type 5+).
- Registering signs you in immediately. A taken username is rejected with "שם המשתמש כבר
  תפוס".

## 3. Create the Firestore database

Firebase Console → **Build → Firestore Database → Create database** → start in
**production mode** → pick a location → **Enable**.

Then set the security rules. They're saved in [`firestore.rules`](firestore.rules) —
open the **Rules** tab in the console, paste the contents of that file in, and
**Publish**. In short, those rules say:

- You can only write **your own** prediction, and only before the deadline.
- You can't read anyone **else's** prediction (single doc or the whole list) until the
  deadline passes — so nobody can copy someone else's guess early.
- The real election results (`meta/results`) can be read by anyone signed in, but only
  written by the one admin account.

> If you change the admin email or the deadline in `index.html`, update them to match in
> `firestore.rules` too — the two files must agree, since the rules are what actually
> enforce this (the JavaScript UI lock is just a convenience, not security).

## 4. Paste your config into `index.html`

Open `index.html`, find the `FIREBASE_CONFIG` block near the top of the second
`<script type="module">` block, and replace the placeholders with the values from step 1:

```js
const FIREBASE_CONFIG = {
    apiKey:            '...',
    authDomain:        'elections26.firebaseapp.com',
    projectId:         'elections26',
    storageBucket:     'elections26.firebasestorage.app',
    messagingSenderId: '...',
    appId:             '...'
};
```

> These values are **not secrets** — Firebase web config is meant to be public; your
> Firestore security rules (step 3) are what actually protect the data.

## 5. Set the deadline and the admin account

- `DEADLINE` in `index.html` — set this to the real election date/time (poll-closing
  time is the natural choice). It currently has a placeholder value; **it will not match
  the real 2026 election date until you edit it.**
- `ADMIN_UID` in `index.html` — the Firebase UID (not email/username) of whichever
  account you want to be the admin (the one who, after the election, enters the real
  mandate count per party). Register that account first, then copy its UID from Firebase
  Console → Authentication → Users, and set it here. Update the matching UID in
  `firestore.rules` too — using the UID instead of the email avoids leaking the admin's
  username in the public repo.

## 6. Run it

```bash
python -m http.server 8080     # then open http://localhost:8080
# or: npx serve .
```

Open **http://localhost:8080**, register a username + password, divide the 120 mandates
across the parties, and press **שמירת הניחוש**.

## The parties

The 14 running מפלגות live in the `RAW_PARTIES` array in `index.html`, each as
`[ot, he]` — `ot` is the ballot letter(s) (e.g. `מחל`, `עם`), `he` is the full party
name. Colors are auto-assigned from a fixed palette; edit `RAW_PARTIES` directly to
add/remove parties or if the official ballot list changes before election day.

## Data shape in Firestore

Collection **`predictions`**, one document per user (doc id = Firebase `uid`):

```json
{
  "uid": "abc123",
  "name": "dana123",
  "email": "dana123@users.elections26.app",
  "season": "2026",
  "mandates": { "מחל": 24, "כן": 14, "...": 0 },
  "updatedAt": 1735500000000
}
```

- `mandates` maps every party's ballot letter to the predicted seat count. The 14 values
  always sum to at most 120 — the UI enforces that live, since 120 is a shared budget.
- Each user only ever writes their own document; re-saving overwrites it.

The real results live in `meta/results`, same shape under `mandates`, written only by the
admin account after the election.

## Tabs

- **הניחוש שלי** — divide 120 mandates across the parties with a slider + number field
  per party. Each slider's reachable max is capped live to whatever's left of the
  120-mandate budget, so the total can never exceed 120 (it turns green once it hits
  exactly 120).
- **כל הניחושים** — everyone's saved predictions, revealed only after the deadline.
- **לוח התוצאות** — leaderboard, visible once the admin has entered the real results.
  For each party, let `diff = |predicted − real|`: a bullseye (`diff == 0`) scores
  **+3**, anything else scores **−diff** (e.g. off by 2 seats = −2). Score = the sum of
  that across all 14 parties; highest total wins and ranks first.
- **עדכון תוצאות** (admin only) — where the admin enters the real mandate count per party
  after the election and saves; everyone's leaderboard score recomputes from that.

## Notes
- Login persists across sessions (Firebase `browserLocalPersistence`).
- Works on mobile and desktop — no drag-and-drop here, just sliders/number inputs.
