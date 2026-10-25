# FrameFlux — AI Video Club starter

A responsive Netflix-inspired video membership website. The visual storefront, category filters, search, and sample cards work immediately. Real signup, private video uploads, and payments need the backend setup below; demo cards are placeholders, not real videos.

## 1. Supabase backend
1. Create a project at https://supabase.com/.
2. Run `supabase-schema.sql` in SQL Editor.
3. In Storage, create a **private** bucket called `videos` and a **public** bucket called `posters`.
4. In Project Settings → API, copy the Project URL and anon/public key.

## 2. Frontend config
Copy `config.example.js` to `config.js`; add your Supabase URL and anon key, Paystack public key, and desired price. `index.html` already loads `config.js`. Never place service-role or Paystack secret keys in browser files.

## 3. Make only yourself the uploader
1. Deploy and sign up with the email address only you control.
2. Run in Supabase SQL Editor, replacing the email: `update public.profiles set role='admin' where email='YOUR-EMAIL@example.com';`
3. Sign out and sign back in. Your account button will open Creator Studio. Database and storage policies restrict uploads to the admin account.

## 4. Payments
1. Set up Paystack at https://paystack.com/ and test with test keys first.
2. Deploy to Vercel and add server environment variables: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `PAYSTACK_SECRET_KEY`, `MONTHLY_PRICE_KOBO` (₦2,500 = `250000`), and optionally `PAYSTACK_CALLBACK_URL`.
3. Set the Paystack public key in `config.js`.
4. This starter verifies a successful one-time Paystack payment and grants 30 days of access. It is **not auto-renewing recurring billing**. For automatic renewals, implement a Paystack subscription plan and signed webhook handling before advertising recurring subscriptions.
5. Test success, failure, cancellation, and duplicate payment scenarios before accepting real money.

## 5. Deploy
Upload the files, including the `api` folder, `package.json`, and `vercel.json`, to a GitHub repo and import it at https://vercel.com/. Add the environment variables above in Vercel. GitHub Pages alone cannot run these server-side payment endpoints; deploy the full project to Vercel or another Node-capable host.

## 6. Before launch
- Starter upload limit is 200 MB. For larger videos, use Mux, Cloudflare Stream, or Bunny Stream.
- Private storage and expiring signed URLs reduce casual sharing but cannot prevent screen recording. Consider a specialist video platform for stronger protection.
- Add a privacy policy, terms, refund/cancellation details, support contact, and rights-cleared video content.

## Multi-creator platform upgrade (schema foundation)
The database schema now includes viewer/creator/admin roles, creator channel pages with creator-set monthly prices, per-creator subscription records, reactions, comments, and saved videos. This is the data foundation for the expanded product model.

Important: the current `index.html`/`app.js` interface and the original Paystack endpoints still implement the earlier single-library membership flow. Do not launch or accept real payments for creator subscriptions yet. The registration UI, creator dashboard/pricing editor, social-action UI, per-creator checkout and payment-webhook logic must be connected to these tables and tested before production. The SQL schema is intended for a fresh Supabase project; if you already ran the earlier schema, back up the project and use a reviewed migration rather than blindly rerunning this file.

### Creator subscription payment design
Each creator has a `creator_pages.monthly_price_kobo` value. At checkout, the server must fetch this value from Supabase, create a payment tied to the subscriber and creator, verify the Paystack transaction server-side, and activate only that specific `(subscriber_id, creator_id)` subscription. Never trust a price sent by the browser. Configure Paystack webhooks for renewals/cancellations if you later offer automatic recurring billing. Creator payouts/revenue sharing require a separate, compliant payout workflow; payment collection alone does not automatically pay creators.

### Social features and access controls
The schema stores one like/dislike reaction per signed-in user per video, comments, and private saved-video lists. The application still needs to connect its buttons to these tables. Private video access must be enforced by signed storage URLs or a video delivery provider—not merely by hiding a player button in the browser. Test every access policy using separate viewer, creator, and admin accounts before launch.


## 7. Multi-creator subscriptions and 10% platform commission

The project now includes server endpoints:
- `POST /api/creator-subscribe-initialize` with `{ "creator_id": "CREATOR_UUID" }`
- `POST /api/creator-subscribe-verify` with `{ "reference": "PAYSTACK_REFERENCE" }`

The initialize endpoint reads the creator's published page and price from Supabase (not from browser input). The verify endpoint checks the authenticated user, stored payment, Paystack transaction status, currency, email, and amount before recording a 30-day creator subscription. It records 10% of the gross subscription amount as the platform commission and 90% as the creator's payable balance.

**Important financial limitation:** this is commission accounting, not automatic creator payouts. Paystack transaction fees may also apply, and the actual settlement / creator payout flow must be designed and tested before production. Do not promise creators a specific net payout until payment processing fees, refunds, chargebacks, tax obligations, and payout timing are defined. Do not treat a browser redirect as proof of payment.

### Add these columns if you already created the earlier schema
Run this migration once in Supabase SQL Editor:
```sql
alter table public.payments
  add column if not exists platform_commission_kobo integer,
  add column if not exists creator_earning_kobo integer;
```

### Before you accept money
1. Add the columns above if the project database already exists.
2. Configure Vercel variables listed in section 4, including the Supabase service-role key and Paystack secret key. Never put these secrets in `config.js`.
3. Build and test the creator signup/channel editor and wire the front end to the new endpoints.
4. Add a signed Paystack webhook with idempotency for asynchronous confirmation, renewals and refunds before relying on it in production.
5. Test using Paystack test keys and separate viewer, creator, and admin accounts. Test creator price changes, repeat payments, failed payments, duplicate callbacks, cancellation, expired access, comments/reactions, and attempts to access another creator's private video.
6. Add a creator payout workflow. The `creator_earning_kobo` field is a ledger amount only; it does not move money to a creator's bank account.
