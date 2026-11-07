# Ear & Ember Bakehouse: turning on e-commerce

The site ships as a static demo: `commerce.demo` is `true`, so every order, WhatsApp and call button only shows a "not available in demo site" notice. Set `demo: false` to turn on WhatsApp ordering. No backend exists yet. The page already contains the full checkout (customer form, pickup or delivery, order submission, payment redirect), and it switches on from the config block near the top of `index.html`:

```js
commerce: { demo: false, mode: "api", api: { baseUrl: "/api" }, ... }   // was demo: true, mode: "whatsapp", baseUrl: ""
catalog:  { pricesConfirmed: true, ... }                   // once prices are real
```

No markup changes are needed. If `baseUrl` is empty, the page stays on WhatsApp ordering.

## What changes in API mode

| | `whatsapp` (now) | `api` |
|---|---|---|
| Menu | `catalog.products` in the config | `GET /products`, with the config list as fallback if it fails |
| Cart button | Opens a pre-filled WhatsApp message | Checkout form, then `POST /orders` |
| Placeholder banner | Shown | Hidden |
| Backend down or error | n/a | Cart is kept and the customer is offered WhatsApp (`fallbackToWhatsApp`) |

Custom cake enquiries (`catalog.enquiries`) stay on WhatsApp in both modes.

## API contract

If the backend is on a different domain from the site, it must allow this site's origin through CORS, including the `Idempotency-Key` request header. Prices are whole LKR.

### `GET /products`

```json
{ "products": [
  { "id": "sliced-loaf-rolls", "name": "Artisanal Sliced Loaf & Rolls", "price": 450,
    "categories": ["breads"], "description": "...", "image": "https://...", "alt": "...", "available": true }
] }
```

- `id`, `name` and `price` are required. Products missing any of them are dropped.
- `categories` uses the ids in `catalog.categories`: `breads`, `pastries`, `sweets`, `cakes`, `crowd`. Each one is a grain on the wheat ear and a menu view at `#/menu/<id>`.
- `available: false` shows "Sold out today" and blocks adding to the cart.
- `image` must be `https://`, `/...` or `assets/...`.

### `POST /orders`

Header: `Idempotency-Key: <uuid>`. It stays the same across retries of an unchanged cart.

```json
{ "items": [{ "productId": "sliced-loaf-rolls", "qty": 2 }],
  "customer": { "name": "...", "phone": "0771234567" },
  "fulfilment": { "method": "pickup", "showroom": "Old Mill Road", "address": null },
  "notes": "", "displayedTotal": 900, "currency": "LKR" }
```

- `method` is `pickup` or `delivery`. Pickup always has a `showroom`, delivery always has an `address`, and delivery's `showroom` may be `null`.
- Success is any 2xx response with an `orderId`:

```json
{ "orderId": "EE-1042", "status": "received", "total": 900, "paymentUrl": "https://..." }
```

- `paymentUrl` is optional and must be `https://`. If present, the customer is redirected there. If absent, they see a confirmation with `orderId` and `total`, which suits cash on delivery or pay at the showroom.
- For errors, a 4xx with `{ "message": "..." }` shows that message to the customer, so write it for them. Anything else shows a generic "try again or use WhatsApp" message.

## Rules the backend must enforce

The browser can't be trusted, so these can't live in the page:

1. **Work out the total from your own prices.** The request has no unit prices on purpose. `displayedTotal` only tells you what the customer saw, so reject or flag a mismatch and never charge it.
2. **Validate everything:** product exists and is available, quantity is 1 to 99, phone format, valid showroom, address present for delivery.
3. **Honor the idempotency key** so a retry can't create a second order. Record a key only once an order has been created, because a customer may correct their details after a failed attempt and resend with the same key.
4. **Mark orders paid only from the gateway's signed webhook,** never from the customer's redirect back to the site.
5. **Rate limit `POST /orders`,** and treat customer names, phones and addresses as personal data.

## Still to build

This is the part behind "Full System Ready upon Launch", and none of it exists yet:

- The API above, a database, and a way for each showroom to receive orders
- An admin screen for products, prices, stock and order status
- A payment gateway with a Sri Lankan LKR merchant account, plus webhook handling
- A "return from payment" page: the site currently has nothing to show when the gateway sends the customer back, and it should read the order status from the backend
- Delivery rules: areas, fees and cut-off times. The page currently tells customers delivery costs are confirmed at checkout
- Customer order notifications (SMS or WhatsApp)

## Decisions for the owner

- Pay online, or cash on delivery and pay at the showroom? Cash-only needs no gateway, so it's a much smaller first version.
- Delivery from which showrooms, to which areas, at what cost?
- Who handles incoming orders at each showroom, and how do they want to see them?
