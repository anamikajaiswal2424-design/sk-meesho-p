# Payment / MongoDB setup

Updated target: sk-meesho-cashfree-2-main. This uses the same receipt-tracking source as FINAL_main_project.

## UPI kahan change karni hai
MongoDB Compass -> database `test` -> collection `settings` -> existing document -> `upi.id`.
`upi.name` is the receiver display name. Set `upi.Phonepe`, `upi.Gpay`, `upi.Paytm`, `upi.Bhim` to true for the payment buttons you want enabled. QR always uses `upi.id`.

## Multiple pixels kahan add karne hain
In the SAME settings document, use `facebookPixel.id` (capital P, as in your screenshot). Lowercase `facebookpixel` is also supported.
Example:
```json
{"facebookPixel":{"id":"4258296857766357,928355373187526,27901500929552426","enabled":true}}
```
Use your real pixel IDs, separated by commas; duplicate IDs are removed. `enabled:false` disables tracking. Only PageView is emitted on ordinary pages / SPA route changes. The success page emits only Purchase after server verification, with actual amount and order ID. Other Meta events in the existing bundle are disabled. Custom code, events array and CAPI token are deliberately not exposed/executed.
Purchase repeats are suppressed per pixel/order in this browser; clearing browser storage or using another browser can permit another event. The stable event ID is supplied for deduplication.

## Database flow
1. POST `/api/create-order`: validate cart against `products.json`, calculate amount on server, generate a new UUID order ID and private token, save PENDING order in `test.paymentOrders`.
2. UPI deep link / QR includes the same order ID in `tn` (comment) and `tr` (reference).
3. POST `/api/verify`: authenticate order token; expected amount comes from saved order, never the browser.
4. Match `paytm_db.transactions` by exact order comment (legacy remark fallback only if comment is absent), amount and SUCCESS. Reject ambiguous matches or receipts already assigned to another order.
5. Atomically claim the receipt, idempotently save SUCCESS transaction in `test.transactions`, update order, then permit Payment Done.
6. Success page checks the saved transaction again before displaying success, clearing cart or emitting Purchase.

IMPORTANT: The existing receiver/payment tracking process must continue importing real bank/Paytm receipts into `paytm_db.transactions`. This website reads that ledger; it does not read a bank account or create fake SUCCESS receipts. The payer's payment note must retain the order ID. No amount-only fallback is allowed.

## Deployment
This package needs Node serverless APIs (Vercel), not static-only hosting.
- Run `npm install`.
- Set the server environment variable `MONGODB_URI` to the exact MongoDB URL supplied in your request. Your local `.env.local` is configured; secrets are excluded from the distributable ZIP.
- The databases are explicitly `test` and `paytm_db` even though the URL has no database name.
- Allow the deployment server's network in MongoDB Atlas.
- Deploy/redeploy the folder after setting the environment variable.
- Cashfree credentials are no longer needed for this direct UPI tracking flow.
- Keep `products.json` in sync with storefront prices/products; it is the server pricing authority. The supplied project contains a compiled frontend rather than its original source; frontend patches are in both supplied JS copies.

## Validation
`npm test`: server pricing, 1000 unique IDs, mismatched comment/amount/status rejection, receipt reuse rejection, invalid token rejection, ambiguous matches, failed save handling, repeat verification, SPA PageView, success-only Purchase and per-order/pixel deduplication.
Live MongoDB read-only connectivity and settings checked. No real payment was performed; a real small payment should be checked after deployment on the target UPI apps and receipt importer.
MongoDB atomic update behavior: https://www.mongodb.com/docs/drivers/node/current/crud/compound-operations/

## Hidden Rs 1 testing product
Open `/product/Dx` directly, for example https://sk-meesho.vercel.app/product/Dx . This product is not listed on the home product grid. Internal product ID: `900001`. Server price in `products.json`: Rs 1 per quantity. It uses the usual product -> cart -> checkout -> UPI verification -> Payment Done flow. For a Rs 1 test, keep quantity 1 and remove other cart items. The direct URL is hidden from the listing but accessible to anyone who knows it; there is no login restriction.

## Five-character Order IDs and QR verification
New IDs contain exactly 5 uppercase letters/digits, e.g. A7DX2. MongoDB's unique _id reserves every ID permanently; collisions retry and existing IDs are never overwritten. Older MS_ orders remain valid. The finite ID space is 36^5; allocation fails safely if no unused ID can be reserved. QR note/reference use the same short ID. Polling continues for up to 10 minutes after PayNow, with a manual restart option. A real imported SUCCESS receipt with matching comment and amount is still mandatory.
Diagnosis on the supplied cluster on 13 October: test.paymentOrders had zero orders; latest paytm_db.transactions update was 8 October, and the screenshot order was absent. Check the deployment MONGODB_URI and receipt importer. Do not infer success solely from scanning a QR or matching Rs 1.

Updated cluster diagnosis: user confirmed milan.fkzozcg.mongodb.net; settings/orders are in test and receipts in paytm_db (the URI database name Milan does not change these explicit databases). The screenshot payment receipt was present with SUCCESS and Rs 1; its comment had MS without the underscore. Legacy MS_ note matching now accepts that specific transformation, while still requiring correct amount/status and an unclaimed receipt. Local server connection updated to the supplied Milan cluster; deployment must keep the same MONGODB_URI.
