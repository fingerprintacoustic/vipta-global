# VIPTA Global — shipping request tracker

Single static page (`index.html`). No custom backend, no build step, no Cloud Functions.
Firebase Auth + Firestore + Storage handle everything directly from the browser.

## 1. Create the Firebase project

1. Go to https://console.firebase.google.com → **Add project** → name it (e.g. `vipta-global`).
2. In the project, go to **Authentication → Sign-in method** → enable **Email/Password**.
3. Go to **Firestore Database** → **Create database** → start in **production mode**.
4. Go to **Storage** → **Get started** → start in **production mode**. This now requires the **Blaze (pay-as-you-go)** plan — as of February 2026, Google requires Blaze for any Storage bucket, even a mostly-free one. Realistic cost at VIPTA's volume (weigh-in photos for maybe a few hundred packages a month) should stay near $0; set a Google Cloud budget alert as a safety net (Billing → Budgets & alerts → create a low threshold like $5–10).
5. Go to **Project settings → General → Your apps → Web app (</>)** → register an app (no Hosting needed) → copy the `firebaseConfig` object.

## 2. Wire up the config

Open `index.html`, find the `firebaseConfig` object near the top of the `<script>` block, and paste in your real values (`apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId`).

## 3. Deploy the security rules

In the Firebase console:
- **Firestore Database → Rules** → paste in the contents of `firestore.rules` → Publish.
- **Storage → Rules** → paste in the contents of `storage.rules` → Publish.

(Or use the Firebase CLI: `firebase deploy --only firestore:rules,storage:rules` if you set up a `firebase.json`.)

## 4. Seed the starting pricing

The app falls back to sensible defaults from the VIPTA price sheet, but the client should set their real, current rates themselves:
1. Once their `admin_us` account is promoted (see step 5), they log in and go to the **Pricing** tab.
2. Confirm/adjust the numbers, click **Save pricing** — this writes to `settings/pricing` in Firestore and immediately affects every future quote.

## 5. Create the admin accounts

Anyone can sign up through the app, but **everyone starts as a `customer`** — admin roles are never self-selectable (this avoids the same bug BiteDash had). To promote someone:

1. Have the person sign up normally in the app (US ops person, then Zimbabwe ops person).
2. In the Firebase console, go to **Firestore Database → `users` collection** → find their document (matches their `uid` from **Authentication**).
3. Edit the `role` field:
   - US-side person → `admin_us`
   - Zimbabwe-side person → `admin_zim`

They'll see the admin view (Requests / Pricing / Announcement tabs) next time they load the app.

## 6. Deploy to GitHub Pages

1. Push this folder to a repo (e.g. `fingerprintacoustic/vipta-global`).
2. Repo → **Settings → Pages** → Source: deploy from `main` branch, root folder. (Free GitHub accounts need the repo set to **Public** for Pages to work — private Pages requires a paid GitHub plan.)
3. Once you buy VIPTA's domain, add a `CNAME` file in this folder containing just the domain (e.g. `viptaglobal.com`), and point the domain's DNS (via Dynadot, same as your other projects) at GitHub Pages.

## How the flow works

- **Customer**: picks a request type (self-purchase / buy-on-my-behalf / subcontractor), submits a product link, sees an 8-stage tracker. Once quoted, arranges payment (EcoCash/InnBucks/bank/cash) and notes a reference number.
- **admin_us**: sends itemized quotes (a single request can hold several products, each independently priced from the Pricing tab, editable per line item), confirms payment received, marks purchased, then — before marking shipped — captures a **weigh-in**: a photo plus the actual measured weight for each item. Photos are stored per request/item ID, so there's no manual matching required regardless of what order packages get opened. If an item's category is weight-based, the shipping fee automatically recalculates from the actual weight; flat-rate items (phones/tablets) keep their fixed price regardless of weight.
- **admin_zim**: marks arrived in Zimbabwe, out for delivery, delivered.
- Each admin can only move the stages that belong to their side — no manual coordination needed to keep the customer's tracker current.
- **Export to Excel**: a button on the admin Requests tab downloads a CSV (opens natively in Excel) with one row per item — customer, phone, request type, category, quoted vs. actual weight, cost, shipping fee, photo link, status, payment info. Useful for dispute records or handing data to the client's own bookkeeping.

## Request types (internal admin classification only)

Every request has a `requestType`, but **customers never see or choose this** — it's set via a small dropdown that only appears in the admin view, deliberately kept out of the customer-facing form so customers can't see that some other customers operate as subcontractors:
- `self_purchase` (default) — customer already owns the item, just needs shipping
- `buy_for_me` — customer wants VIPTA to purchase the item on their behalf
- `subcontractor` — a reseller shipping on behalf of their own customers through VIPTA

Either admin can tag/re-tag a request's type at any time, independent of its shipping stage. It's included in the CSV export for internal reporting.

Note: pricing logic doesn't yet differ by request type (all three are quoted the same way today). If VIPTA wants different handling per type — e.g. a purchasing commission for `buy_for_me`, or a wholesale rate for `subcontractor` — that's a well-scoped Phase 2 addition once their actual policy for each is settled.

## What's deliberately NOT here yet (v1 scope)

- No in-app payment gateway (Paynow etc.) — payment confirmation is manual, by design, to keep the core payment flow simple.
- No SMS/WhatsApp notifications — customers check status by opening the app. Worth revisiting once there's Twilio infrastructure already in place elsewhere.
- No admin self-signup for admin roles — must be granted manually in the Firebase console per person.
- No differentiated pricing per request type (see above) — flagged for a future phase.
