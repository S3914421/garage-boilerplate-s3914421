# Deploy to Vercel — Step by Step

Taking your app live. Vercel's free **Hobby** tier runs Next.js server-side rendering with no
billing account required, and your Firebase project stays on the free Spark plan.

This is the beginner walkthrough. The full pipeline — Firestore rules, GitHub Actions, the
optional backend Cloud Function — is in [CI-CD.md](CI-CD.md).

**Before you start:** the app should already run locally (`pnpm run dev`) against a real
Firebase project. If it doesn't, finish [GUIDE.md §2](GUIDE.md) first.

---

## What you're actually deploying

Only `frontend/` goes to Vercel. Firebase (Auth + Firestore) is already hosted by Google, so
your deployed site talks to the exact same Firebase project your local dev server does.

```
Your repo ──push──► Vercel  (builds + hosts frontend/)
                      │
                      └──► Firebase Auth + Firestore  (already live, unchanged)
```

The `backend/` Cloud Function is **not** deployed here and is not required — most features
never touch it, and it needs the paid Blaze plan.

---

## Step 1 — Push your code to GitHub

Vercel deploys from a GitHub repo, so your work has to be pushed first:

```bash
git push -u origin main
```

---

## Step 2 — Import the project

1. Go to [vercel.com](https://vercel.com) and sign in **with GitHub**
2. Click **Add New… → Project**
3. Find this repository in the list and click **Import**

## Step 3 — Set the root directory

This repo is a pnpm workspace with two packages, so Vercel needs to be told which one is the
website:

- **Root Directory** → click **Edit** → select **`frontend`**

Leave Framework Preset, Build Command, and Output Directory alone — Vercel auto-detects
Next.js once the root directory is set.

## Step 4 — Add the environment variables

**Vercel does not read your root `.env` file.** Every variable has to be added by hand, under
the exact same name it has in `.env`. In the import screen (or later under
**Settings → Environment Variables**), add:

| Variable | Where to get the value |
|----------|------------------------|
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | your root `.env` — same value |
| `NEXT_PUBLIC_FIREBASE_API_KEY` | Firebase `firebaseConfig` → `apiKey` |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | `firebaseConfig` → `authDomain` |
| `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | `firebaseConfig` → `messagingSenderId` |
| `NEXT_PUBLIC_FIREBASE_APP_ID` | `firebaseConfig` → `appId` |
| `NEXT_PUBLIC_APP_NAME` | your app's display name |
| `NEXT_PUBLIC_APP_URL` | your Vercel production URL — you don't know it yet, see Step 6 |
| `FIREBASE_SERVICE_ACCOUNT_KEY_BASE64` | the base64 service-account string from your `.env` |

Optional: `NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID` (only if Google Analytics is enabled on the
Firebase web app).

Apply each one to **Production** — and to **Preview** too if you want pull-request preview
deployments to work.

> ⚠️ `FIREBASE_SERVICE_ACCOUNT_KEY_BASE64` is a secret. It must **not** have a
> `NEXT_PUBLIC_` prefix — anything with that prefix is compiled into the browser bundle and
> visible to every visitor.

## Step 5 — Deploy

Click **Deploy** and wait for the build. Vercel gives you a URL like
`your-app.vercel.app` when it finishes.

## Step 6 — Point `NEXT_PUBLIC_APP_URL` at the real URL

Now that you know the production URL:

1. **Settings → Environment Variables** → set `NEXT_PUBLIC_APP_URL` to
   `https://your-app.vercel.app`
2. **Deployments** tab → the latest deployment → **⋯ → Redeploy**

The redeploy is required. Vercel does **not** apply new or changed env vars to deployments
that already exist.

## Step 7 — Authorize the domain in Firebase

Firebase Auth rejects sign-ins from domains it doesn't know:

- Firebase Console → **Authentication → Settings → Authorized domains** → **Add domain**
- Add `your-app.vercel.app`

## Step 8 — Deploy your Firestore security rules

Your rules live in `firebase/firestore.rules`, and until they're deployed the live site is
running whatever rules the Firebase console currently has:

```bash
npx firebase-tools login
```

```bash
npx firebase-tools deploy --only firestore:rules
```

This is free — no Blaze plan needed. (If you merge to `main` via GitHub, `deploy.yml` also
does this automatically once the repository secrets from [CI-CD.md](CI-CD.md) are set.)

## Step 9 — Smoke-test it

Open your Vercel URL and:

1. Sign up with a new account
2. Confirm the user appears in Firebase Console → **Authentication → Users**
3. Confirm the `users/{uid}` document appears in **Firestore Database**
4. Create something in your feature and confirm it reads back

If all four work, you're live.

---

## After the first deploy

**Every push to `main` auto-deploys to production.** There is no approval gate on Vercel's
side — treat merging to `main` as shipping. Pull requests get their own preview URLs.

---

## Troubleshooting

| Symptom | Cause & fix |
|---------|-------------|
| "Firebase web config is incomplete" | A `NEXT_PUBLIC_FIREBASE_*` variable is missing in Vercel. Add it, then **redeploy** — new env vars don't apply retroactively. |
| Changed an env var, nothing changed | Redeploy. `NEXT_PUBLIC_*` values are baked in at build time. |
| `auth/unauthorized-domain` on sign-in | The Vercel domain isn't in Firebase → Authentication → Settings → Authorized domains. See Step 7. |
| Build fails: "No Next.js version detected" | **Root Directory** isn't set to `frontend`. Settings → General → Root Directory. |
| "Missing or insufficient permissions" from Firestore | Your rules aren't deployed, or they don't allow the read/write. See Step 8. |
| Pages load but server-side data is empty | `FIREBASE_SERVICE_ACCOUNT_KEY_BASE64` is missing or malformed in Vercel. Re-copy it as one unbroken line, then redeploy. |
| Sign-in shows a success toast but stays on the sign-in page | `/api/auth/session` returned 401, so no `__session` cookie was set and the proxy bounced the redirect. The toast fires before the session is confirmed, so it lies. Read the deployment's Runtime Logs for the real cause — the two below are the common ones. |
| Runtime log: `FIREBASE_SERVICE_ACCOUNT_KEY_BASE64 environment variable is not set` | The variable is absent **or empty**. A variable created with a blank value still appears in the Vercel list, so seeing the name there is not proof it has a value. Delete it and re-create it rather than guessing. |
| Runtime log: `invalid_grant: Invalid JWT Signature` / `app/invalid-credential` | The variable exists and is well-formed, but holds a **revoked** key — typically after rotating the service account key without updating Vercel. Paste the current key and redeploy. |
| Changed the variable but the error is identical | Vercel bakes environment variables in at build time. Editing a variable changes nothing until a **new deployment** exists. Confirm one was actually created — if the newest deployment predates your edit, it is still serving the old value. |
| The backend API 404s | Expected — `backend/` is not deployed to Vercel. It's an optional Firebase Cloud Function requiring the Blaze plan; see [CI-CD.md](CI-CD.md). |

---

## Rotating the service account key

Do this in order. Reversing steps 2 and 5 takes production down.

1. Google Cloud → Service Accounts → the `firebase-adminsdk-*` account → **KEYS** →
   **ADD KEY** → Create new key → JSON. Leave the old keys alone for now.
2. Base64-encode it and update the value in all three places: the local `.env`
   (then `pnpm run env:sync`), the Vercel environment variable, and the GitHub
   Actions secret.
3. **Redeploy.** Until a new deployment exists, production is still running the
   old key.
4. Confirm the new deployment is `READY` and sign-in still works.
5. Only now delete the old keys.
6. Test sign-in once more. This is the step that actually proves anything: while
   the old key still exists, both keys work, so a passing test tells you nothing
   about which one production is using. If sign-in survives step 5, the new key
   is genuinely live.

To encode without the key ever passing through a terminal or a chat window:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\path\to\key.json")) | Set-Clipboard
```

Delete the downloaded JSON afterwards — the value now lives in `.env`, which is
gitignored.

---

## Related

- [CI-CD.md](CI-CD.md) — the full pipeline: GitHub Actions, Firestore rules, the optional backend
- [ENV-VARS.md](ENV-VARS.md) — every environment variable, what it does, and which are secret
- [GUIDE.md](GUIDE.md) — the beginner's guide from clone to first feature
