# VIPTA Global — shipping request tracker

Single static page (`index.html`). No server, no build step, no Cloud Functions.
Firebase Auth + Firestore handle everything directly from the browser.

## 1. Create the Firebase project

1. Go to https://console.firebase.google.com → **Add project** → name it (e.g. `vipta-global`).
2. In the project, go to **Authentication → Sign-in method** → enable **Email/Password**.
3. Go to **Firestore Database** → **Create database** → start in **production mode**.
4. Go to **Project settings → General → Your apps → Web app (</>)** → register an app (no Hosting needed) → copy the `firebaseConfig` object.

Note: this app deliberately does **not** use Firebase Storage. As of February 2026, Storage requires the paid Blaze plan even for a default bucket — skipping it keeps the whole project on the free Spark plan. Payment proof is a text reference field only (no screenshot upload).

## 2. Wire up the config

Open `index.html`, find the `firebaseConfig` object near the top of the `<script>` block, and paste in your real values (`apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId`).

## 3. Deploy the security rules

In the Firebase console: **Firestore Database → Rules** → paste in the contents of `firestore.rules` → Publish.

(Or use the Firebase CLI: `firebase deploy --only firestore:rules` if you set up a `firebase.json`.)

## 4. Seed the starting pricing

The app will fall back to sensible defaults from the VIPTA price sheet, but you should set the real rates once:
1. Sign up a normal account in the app (this becomes your first `admin_us` — see step 5).
2. Log in, go to the **Pricing** tab, and confirm/adjust the numbers.
3. Click **Save pricing** — this creates `settings/pricing` in Firestore.

## 5. Create the two admin accounts

Anyone can sign up through the app, but **everyone starts as a `customer`** — admin roles are never self-selectable (this avoids the same bug BiteDash had). To promote someone:

1. Have the person sign up normally in the app (US ops person, then Zimbabwe ops person).
2. In the Firebase console, go to **Firestore Database → `users` collection** → find their document (matches their `uid` from **Authentication**).
3. Edit the `role` field:
   - US-side person → `admin_us`
   - Zimbabwe-side person → `admin_zim`

They'll see the admin view (all requests + pricing) next time they load the app.

## 6. Deploy to GitHub Pages

1. Push this folder to a repo (e.g. `fingerprintacoustic/vipta-global`).
2. Repo → **Settings → Pages** → Source: deploy from `main` branch, root folder.
3. Once you buy VIPTA's domain, add a `CNAME` file in this folder containing just the domain (e.g. `viptaglobal.com`), and point the domain's DNS (via Dynadot, same as your other projects) at GitHub Pages.

## How the flow works

- **Customer**: submits a product link → sees an 8-stage tracker → once quoted, arranges payment (EcoCash/InnBucks/bank/cash) and optionally notes a reference number.
- **admin_us**: sends quotes (auto-priced from the Pricing tab, editable per request), confirms payment received, marks purchased, marks shipped.
- **admin_zim**: marks arrived in Zimbabwe, out for delivery, delivered.
- Each admin can only move the stages that belong to their side — no manual coordination needed to keep the customer's tracker current.

## What's deliberately NOT here yet (v1 scope)

- No in-app payment gateway (Paynow etc.) — payment confirmation is manual, by design, to keep this serverless and free to run.
- No payment screenshot upload — Firebase Storage now requires the paid Blaze plan, so proof of payment is a reference number/note only; the admin's manual "confirm payment" action is always the real source of truth anyway.
- No SMS/WhatsApp notifications — customers check status by opening the app. Worth revisiting once there's Twilio infrastructure already in place elsewhere.
- No admin self-signup for admin roles — must be granted manually in the Firebase console per person.
