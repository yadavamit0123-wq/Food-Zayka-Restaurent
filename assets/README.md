# StackFood — API Reference

API documentation for the customer apps and the restaurant (vendor) app. Conventions shared by
every module are documented once per audience; each module then documents its own endpoints.

All samples are real responses captured from the demo dataset, not hand-written examples.

## Contents

| Audience | Module | Endpoints | Covers |
|---|---|---|---|
| Customer | [BOGO Offers & Happy Hour](#bogo-offers--happy-hour) | 6 read + cart/checkout | Buy-one-get-one offers, bundle pricing, cart and order behaviour, zone-wise Happy Hour stores |
| Vendor app | [BOGO & Happy Hour](#vendor-app-api) | 12 | Joining, answering an admin request, reworking a rejection, leaving — with the enrolment state machine and the employee role guard. Per-endpoint reference with request/response samples. |

---

## Conventions

These follow the existing customer API, so nothing new is introduced for the clients.

**Headers** (same as `campaigns/*` and `restaurants/*`)

| Header | Required | Notes |
|---|---|---|
| `zoneId` | yes | JSON array, e.g. `[1]`. Everything is filtered to these zones. |
| `latitude`, `longitude` | yes | Used for the restaurant `open` flag and `distance`. |
| `X-localization` | no | Offer title/description are translated; defaults to `en`. |
| `Authorization` | see below | `Bearer <token>` for a logged-in customer. |

**Auth** — every endpoint runs behind `apiGuestCheck`, the same middleware as
`restaurants/visit-again`. Send **either** an `Authorization: Bearer <token>` header **or** a
`guest_id` query parameter. Sending neither returns `401`:

```json
{ "errors": [ { "code": "auth-001", "message": "Unauthorized." } ] }
```

A logged-in customer additionally gets a real `remaining_uses`; a guest gets the full
allowance because there is no usage history to read.

**Identifiers** — every `{id}` and `restaurant_id` accepts **an id or a slug**, matching
`basic_campaign_details`. `/bogo/offers/3` and `/bogo/offers/joy-bridges` are equivalent.

**Pagination** — `limit` (default 10) and `offset` (default 1, page number), responding with
the house envelope:

```json
{ "total_size": 2, "limit": 10, "offset": 1, "offers": [ … ] }
```

**Filtering** — an optional `order_type` (`delivery` | `take_away` | `dine_in`) restricts
results to offers that run for that type *and* restaurants that serve it. `delivery` and
`home_delivery` are treated as the same thing.

**Errors** — validation uses the house shape with `403`, missing records return `404`:

```json
{ "errors": [ { "code": "restaurant_id", "message": "Restaurant id is required" } ] }
```

---

## BOGO Offers & Happy Hour

The customer BOGO surface — discovery, the bundle bottom sheet, the cart, and checkout —
plus the zone-wise Happy Hour store list.

**Reading**

| # | Screen | Endpoint |
|---|---|---|
| 1 | Home card | `GET /api/v1/bogo/home` |
| 2 | Offer list | `GET /api/v1/bogo/offers` |
| 3 | Offer details + bottom sheet | `GET /api/v1/bogo/offers/{id\|slug}` |
| 4 | Store details | `GET /api/v1/restaurants/bogo-offers?restaurant_id={id\|slug}` |
| 5 | Happy Hour stores | `GET /api/v1/happy-hour/stores` |
| 6 | Home banner (running Happy Hour) | `GET /api/v1/happy-hour/running` |

**Cart and checkout** — no BOGO-specific endpoints. A bundle is added, changed and removed
through the cart verbs that already exist, addressed by `bogo_id` or `bogo_group_id`:

| Action | Endpoint | Addressed by |
|---|---|---|
| Add a bundle | `POST /api/v1/customer/cart/add` | `bogo_id` + `quantity` |
| Change quantity | `POST /api/v1/customer/cart/update` | `bogo_group_id` + `quantity` |
| Remove a bundle | `DELETE /api/v1/customer/cart/remove-item` | `bogo_group_id` |
| Read the cart | `GET /api/v1/customer/cart/list` | grouped, see below |
| Place the order | `POST /api/v1/customer/order/place` | unchanged |

### Discounts are resolved centrally

A restaurant-wide discount can come from two places: the admin's restaurant discount, and a
Happy Hour that is live right now. Both are a percentage off the whole restaurant, so they are
never stacked.

**A running Happy Hour always wins — even when its rate is the smaller of the two.** It is a
deliberately scheduled window, so while it is open it is the price the store is advertising;
letting a standing discount override it would mean the Happy Hour never takes effect at
exactly the stores that already discount heavily. When the window closes, the standing
restaurant discount resumes automatically.

A Happy Hour configured at 0% is treated as *no* Happy Hour rather than a 0% override, so a
misconfigured one cannot silently strip a discount the store already had.

All of it resolves in `Helpers::get_restaurant_discount()`, the single function every price
in the system already went through. Nothing computes a Happy Hour rate of its own, so:

- ordinary product and food pricing picks it up with no change at the call sites
- **a live Happy Hour discounts a BOGO bundle too**, because the bundle asks the same helper
- the amount is applied with `Helpers::checkAdminDiscount()`, so `min_purchase` and the cap
  behave exactly as they do for a restaurant discount

Happy Hour has no `max_discount` column, so it is reported as uncapped. (A `0` there would be
clamped to zero at checkout rather than left uncapped.)

---

### How a bundle is priced

A buy item's unit price is built from exactly three things and then **frozen** into the
enrolment when the restaurant joins:

```
unit price = food.price + variation price + add-on prices
```

#### Item-level discounts are not applied

A food's own `discount` / `discount_type` is **ignored inside a bundle** — BOGO always uses
the actual price. From the live dataset:

| Item | `food.price` | `food.discount` | Contributes to bundle |
|---|---|---|---|
| Meat Pizza | 400 | **30 (amount)** | 836 = 400 + variation + add-ons |
| FRIED RICE | 120 | 0 | 340 = 120 + variation |
| Medu Vada *(free)* | 95 | **10 %** | 0 |

Meat Pizza sells for 370 on the menu but contributes the full 400 here. This is deliberate:
the free item *is* the promotion, and stacking a per-item discount on top would discount
twice.

It does mean an asymmetry worth knowing about: on the ordinary menu an item discount competes
with the restaurant discount and the larger wins, whereas inside a bundle the item discount is
dropped and only the store-wide rate applies.

#### The three numbers

| Field | Meaning | When computed |
|---|---|---|
| `bundle_price` | buy items × qty, at the frozen unit price | frozen at enrolment |
| `original_price` | buy items **+ free items valued at today's price** | per request |
| `final_price` | `bundle_price` − store-wide discount | per request |

Worked from real data — a 25% Happy Hour on bundle 5:

```
836 + 340            = 1176   bundle_price
1176 + 165 (free)    = 1341   original_price   ← the struck-through figure
1176 − 25%  (294)    =  882   final_price      ← what the customer pays
```

The BOGO discount and the store-wide discount **compose rather than compete**: the percentage
applies to `bundle_price`, which already excludes the free item, so the free item stays free
on top of it.

Two consequences:

- **`original_price` can move while `bundle_price` cannot.** Free items are valued live, so a
  later price change shifts the strikethrough but not what the customer pays. A deleted food
  contributes `0` rather than breaking the total.
- **`min_purchase` is tested against the bundle price**, not the cart total, so a bundle below
  a Happy Hour's `min_order_amount` gets no discount even if the wider cart clears it.

---

### What counts as an available offer

Applied uniformly by every endpoint below:

- offer `status = 1`, inside its `start_date`/`end_date` window, and `total_uses` below
  `usage_limit_total` (an exhausted offer disappears entirely)
- at least one restaurant enrolment with `status = 'approved'`
- that restaurant is active and inside the request's `zoneId`
- when `order_type` is passed: the offer runs for it and the restaurant serves it
  (`dine_in` is read from `restaurant_config`, not the restaurants table)

A restaurant contributes **exactly one bundle per offer** — the pivot is unique on
`(bogo_offer_id, restaurant_id)`.

### When a Happy Hour counts as running

All five must hold, otherwise the store is not in a Happy Hour and no Happy Hour discount is
applied anywhere:

1. Happy Hour `status = 1`
2. its `zone_id` matches the restaurant's zone
3. the restaurant's enrolment is **`approved`** — `pending` and `rejected` do not count
4. the window is open **right now**
   - a permanent weekly rule → today's weekday is in `weekly_days` **and**
     `start_time ≤ now ≤ end_time`
   - anything else → a `happy_hour_dates` row exists for today whose time range covers now
5. `discount > 0`

The first match wins: a restaurant is treated as being in at most one Happy Hour at a time.
Because a window is a schedule rather than a column, step 4 is evaluated in PHP, and the
result is memoised per restaurant for the duration of the request.

---

### 1. Home card

```http
GET /api/v1/bogo/home
```

| Param | Default | Notes |
|---|---|---|
| `limit` | 6 | Preview size. |
| `order_type` | — | Optional filter. |

`total_offers` is the full count regardless of `limit`, so the card can show a badge while
previewing only a few.

```json
{
  "is_live": true,
  "title": "Hurry Up! BOGO Offer Is Live",
  "description": "Get the best value from your order with our exclusive BOGO deals. …",
  "total_offers": 2,
  "offers": [ { "…offer card…" } ]
}
```

---

### 2. Offer list

```http
GET /api/v1/bogo/offers?limit=10&offset=1
```

```json
{
  "total_size": 2, "limit": 10, "offset": 1,
  "offers": [
    {
      "id": 3, "slug": "joy-bridges", "title": "Joy Bridges", "description": "…",
      "image_full_url": "https://…/storage/campaign/….png",
      "buy_qty": 2, "get_qty": 1, "offer_label": "Buy 2 Get 1",
      "order_types": ["delivery", "take_away", "dine_in"],
      "start_date": "2026-08-11 11:11:00", "end_date": "2026-08-29 12:27:00",
      "valid_until": "29 Aug 2026"
    }
  ]
}
```

`image_full_url` and `valid_until` are `null` when the offer has no image or no end date.

---

### 3. Offer details + bottom sheet

```http
GET /api/v1/bogo/offers/{id|slug}?limit=10&offset=1
```

**One call serves both screens.** Each bundle carries its own `buy_items` / `free_items`, so
opening a bundle's bottom sheet costs no second request.

```json
{
  "id": 3, "slug": "joy-bridges", "offer_label": "Buy 2 Get 1",
  "valid_until": "29 Aug 2026",
  "usage_limit_per_customer": null, "remaining_uses": null,
  "total_size": 2, "limit": 1, "offset": 1,
  "bundles": [
    {
      "bundle_id": 5,
      "buy_count": 2, "get_count": 1,
      "item_thumbnails": ["https://…", null, null],
      "restaurant": { "id": 2, "name": "Hungry Puppets", "open": 1, "…standard object…": null },

      "bundle_price": 1176, "original_price": 1341,
      "discount_percentage": 25, "discount_amount": 294, "final_price": 882,
      "is_happy_hour": true,
      "happy_hour": { "id": 91, "slug": "sim-happy-hour", "title": "SIM Happy Hour",
                      "discount": 25, "end_time": "08:19:37" },

      "buy_items": [
        { "id": 18, "food_id": 67, "slug": "meat-pizza67", "name": "Meat Pizza",
          "image_full_url": "https://…", "quantity": 1, "price": 836, "is_free": false,
          "variations": [ { "name": "Size", "values": { "label": ["Small"] } } ],
          "variation_options": [], "variation_summary": "Size : Small",
          "add_ons": [ { "id": 31, "name": "Pepsi", "price": 18, "quantity": 2 } ],
          "avg_rating": 4.67, "rating_count": 5, "veg": 0, "is_available": true }
      ],
      "free_items": [ { "…same shape…": null, "is_free": true, "price": 165 } ]
    }
  ]
}
```

- **`bundle_id` is what the cart takes** — pass it as `bogo_id` to `POST /cart/add`.
- `buy_count` / `get_count` drive the **"2 + 1"** badge.
- **`final_price` is what the customer pays**; `original_price` is the struck-through value.
- `name`, `image_full_url` and a buy item's `price` are the **frozen copies** taken at enrolment.
- A get item's `price` is its normal value, **not** what the customer pays — use `is_free`.
- `variation_summary` is the prebuilt one-liner (`"Size : Small ; Toppings : Capsicum"`).

**404** when the offer does not exist, is not running, or has no approved restaurant in the
caller's zone.

---

### 4. Store details

```http
GET /api/v1/restaurants/bogo-offers?restaurant_id={id|slug}&limit=10&offset=1
```

Same envelope as the offer list, but each card also carries **this restaurant's** bundle —
including `bundle_id`, so the store screen can add to cart directly.

```json
{
  "total_size": 2, "limit": 10, "offset": 1,
  "offers": [
    { "id": 3, "slug": "joy-bridges", "offer_label": "Buy 2 Get 1",
      "bundle_id": 5, "bundle_price": 1176, "original_price": 1341, "final_price": 882,
      "is_happy_hour": true, "buy_count": 2, "get_count": 1 }
  ]
}
```

Returns `404` if the restaurant does not exist or sits outside the caller's zone.

---

### 5. Happy Hour stores (zone-wise)

```http
GET /api/v1/happy-hour/stores?limit=10&offset=1
```

Stores taking part in a Happy Hour in the caller's zone. A Happy Hour belongs to one zone and
reaches a restaurant only through an approved enrolment, so this list is zone-scoped by
definition.

| Param | Default | Notes |
|---|---|---|
| `running` | — | `1` returns only stores whose window is open right now. |

```json
{
  "total_size": 1, "limit": 10, "offset": 1,
  "restaurants": [
    {
      "id": 2, "name": "Hungry Puppets", "open": 1, "…standard restaurant object…": null,
      "is_running_now": true,
      "active_discount": 25,
      "happy_hour": { "id": 91, "title": "SIM Happy Hour", "discount": 25,
                      "min_order_amount": 0, "duration_type": "daily", "is_permanent": false,
                      "weekly_days": [], "start_time": "04:19:37", "end_time": "08:19:37",
                      "is_running_now": true },
      "happy_hours": [ "…every approved Happy Hour for this store…" ]
    }
  ]
}
```

- Default (no `running`) returns **every participating store** with an `is_running_now` flag.
- `happy_hour` is the one currently open, or `null`. `happy_hours` lists all approved ones.
- `active_discount` is what is actually in force via the central resolver.

### 6. Running Happy Hour (home banner)

```http
GET {{base_url}}/happy-hour/running
```

The one Happy Hour open in the caller's zone right now — for the home-screen banner and its
countdown. **Always one object or none, never a list.**

No parameters beyond the standard headers.

**200 OK — one is running**

```json
{
  "is_running": true,
  "happy_hour": {
    "id": 157,
    "slug": "evening-happy-hour",
    "title": "Evening Happy Hour",
    "short_description": "…",
    "cover_image_full_url": null,
    "icon_full_url": null,
    "discount": 25,
    "min_order_amount": 0,
    "duration_type": "daily",
    "is_permanent": false,
    "weekly_days": [],
    "start_time": "22:54:01",
    "end_time": "23:54:01",
    "start_date": "2026-08-11",
    "end_date": "2026-08-13",
    "is_running_now": true,
    "started_at": "2026-08-12 22:54:01",
    "ends_at": "2026-08-12 23:54:01",
    "remaining_seconds": 1199,
    "store_count": 1
  }
}
```

**200 OK — nothing is running**

```json
{ "is_running": false, "happy_hour": null }
```

Not `404` — the app polls this and hides the banner on `is_running: false`.

| Field | Type | Description |
|---|---|---|
| `is_running` | boolean | show the banner or not |
| `discount` | number | the percentage for the "25% OFF!" headline |
| `started_at` / `ends_at` | string | `Y-m-d H:i:s` in app time |
| `remaining_seconds` | **integer** | what the countdown ticks down from; `1199` renders as `19:59` |
| `store_count` | integer | approved, active stores taking part — for "Tap to View Offers" |

Tap-through goes to `GET /happy-hour/stores?running=1`, the participating stores.

#### Why exactly one is safe

A zone can never have two Happy Hours open at the same moment. The admin side refuses a
schedule that collides with an existing one — verified live against every route in:

| Attempt | Result |
|---|---|
| overlapping, containing, or duplicate window on the same date | `409` |
| a multi-day range clashing on only one of its days | `409` |
| permanent weekly over a dated one, **and the reverse order** | `409` |
| a second permanent weekly on the same weekday | `409` |
| custom days clashing with a dated or another custom | `409` |
| **editing** an existing one onto an occupied slot | `409`, and it keeps its old slot |
| a **switched-off** record's slot | still blocked, so re-enabling cannot double-book |
| the same slot in a different zone | allowed — zones are independent |

Windows that merely touch (16:00–17:00 and 17:00–18:00) are allowed and provably never
overlap. A window cannot cross midnight either: a start at or after 23:00 is rejected, so
"one per zone" holds within each calendar day.

The one gap: the check is not a database constraint, so two admins submitting overlapping
schedules at the same instant could in principle both pass. The endpoint stays correct
regardless — it returns a single object either way.

---

## Cart

The most important surface, so it is documented field by field.

A bundle is stored **exactly like ordinary items** — one `carts` row per enrolled food — with
three tag columns. The free row is stored at **price 0**, so cart totals need no special-casing.

```
id   item_id  price  qty  bogo_offer_id  bogo_group_id  is_free_item
217  57       160    1    NULL           NULL           0     ← ordinary
218  67       836    2    3              fe875446…      0     ← bundle: buy
219  59       340    2    3              fe875446…      0     ← bundle: buy
220  69       0      2    3              fe875446…      1     ← bundle: free
```

There are **no BOGO-specific endpoints** — the existing cart verbs take a bundle identifier.

---

### `POST /customer/cart/add`

Two modes, chosen by which identifier is present.

**Mode A — ordinary item** (unchanged)

| Param | Type | Required | Notes |
|---|---|---|---|
| `guest_id` | integer | required unless authenticated | Guest owner of the cart. |
| `item_id` | integer | **yes** | Food or ItemCampaign id. |
| `model` | string | **yes** | `Food` \| `ItemCampaign`. |
| `price` | numeric | **yes** | Unit price. |
| `quantity` | integer ≥ 1 | **yes** | |
| `restaurant_id` | integer\|slug | **yes** | |
| `add_on_ids` | array | **yes in practice** | Omitting it raises a 500 — see Known issues. |
| `add_on_qtys` | array | **yes in practice** | Parallel to `add_on_ids`. |
| `variations` | array | no | Selected variations. Omitting stores `"null"` and breaks `/cart/list` — see Known issues. |
| `variation_options` | array | no | Must be an **array**, not a JSON string. |
| `reel_id` | integer | no | |

**Mode B — BOGO bundle**

| Param | Type | Required | Notes |
|---|---|---|---|
| `guest_id` | integer | required unless authenticated | |
| `bogo_id` | integer | **yes** | The **`bundle_id`** from any read endpoint. Presence of this key selects bundle mode. |
| `quantity` | integer ≥ 1 | no (default 1) | Number of **whole bundles**. Every enrolled line, bought and free, is multiplied by it. |
| `contact_person_number` | string | no | Guest phone, used to check the per-customer usage limit before the order exists. |

**Response — Mode A:** `200` with the cart array (same shape as `GET /cart/list`).

**Response — Mode B:** `200`

| Field | Type | Notes |
|---|---|---|
| `message` | string | `"Offer added to cart"` |
| `bogo_group_id` | string (uuid) | Identifies the group for update/remove. |
| `quantity` | integer | Bundles added. |
| `data` | array | The cart, same shape as `GET /cart/list`. |

**Errors:** `404` `bogo_offer` — unknown/unapproved bundle or outside the zone · `403`
`bogo_offer` — an availability rule failed (message is the reason) · `403` `restaurant` —
the cart already holds another restaurant's items · `401` `auth-001` — no `guest_id` or token.

---

### `GET /customer/cart/list`

| Param | Type | Required | Notes |
|---|---|---|---|
| `guest_id` | integer | required unless authenticated | |
| `restaurant_id` | integer\|slug | **yes** | Scopes the cart to one restaurant. |

**Response:** `200`, an array of entries. **Every entry has the same keys.** A bundle collapses
to one entry, so 4 rows read as 2 entries.

| Field | Type | Ordinary | Bundle |
|---|---|---|---|
| `id` | integer\|null | cart row id | **`null`** — a bundle is not one row |
| `user_id` | integer | owner | owner |
| `is_guest` | boolean | | |
| `restaurant_id` | integer | | the bundle's restaurant |
| `item_id` | integer\|null | food id | **`null`** |
| `item_type` | string\|null | `App\Models\Food` | **`null`** |
| `item` | object\|null | the food | **`null`** |
| `reel_id` | integer\|null | | **`null`** |
| `price` | number | unit price | **the bundle price** |
| `quantity` | integer | units | **number of bundles** |
| `variations` | array\|null | selected | **`null`** |
| `variation_options` | array\|null | selected | **`null`** |
| `add_on_ids` | array\|null | | **`null`** |
| `add_on_qtys` | array\|null | | **`null`** |
| `bogo_offer_id` | integer\|null | `null` | the offer id |
| `bogo_group_id` | string\|null | `null` | the group uuid |
| `is_free_item` | integer\|null | `0` | **`null`** |
| `created_at` | string (ISO 8601) | row created | representative row's timestamp |
| `updated_at` | string (ISO 8601) | row updated | representative row's timestamp |
| `restaurant` | object | standard restaurant | standard restaurant |
| `bogo_details` | object\|null | **`null`** | see below |

Everything item-specific is `null` on a bundle entry — it describes a bundle, not a food.
`price × quantity` is still this line's cart total, so the existing client total formula holds.

**`bogo_details`**

| Field | Type | Notes |
|---|---|---|
| `bogo_group_id` | string (uuid) | Pass to update/remove. |
| `bogo_offer_id` | integer | |
| `bundle_id` | integer | The enrolment id — pass as `bogo_id` to re-add. |
| `offer_title` | string | Shown as the line's title. |
| `offer_slug` | string | |
| `quantity` | integer | Bundles in the cart. |
| `bundle_price` | number | Price of one bundle. |
| `total_price` | number | `bundle_price × quantity`. |
| `is_available` | boolean | `false` blocks checkout. |
| `unavailable_reason` | string\|null | Human-readable reason when unavailable. |
| `buy_items` | array of cart rows | The "Buying Item" lines. |
| `free_items` | array of cart rows | The "Free Item" lines, each at `price` 0. |

Each entry of `buy_items` / `free_items` is a full cart row (`id`, `item_id`, `item`, `price`,
`quantity`, `variations`, `add_on_ids`, `is_free_item`, …) — the individual foods, unlike the
parent entry.

```json
[
  { "id": 462, "item_id": 57, "item_type": "App\\Models\\Food", "price": 160, "quantity": 1,
    "item": { "…food…" }, "bogo_offer_id": null, "bogo_group_id": null, "is_free_item": 0,
    "bogo_details": null },

  { "id": null, "item_id": null, "item_type": null, "item": null,
    "price": 1176, "quantity": 2, "restaurant_id": 2,
    "variations": null, "variation_options": null, "add_on_ids": null, "add_on_qtys": null,
    "bogo_offer_id": 3, "bogo_group_id": "a76a9271-94ea-415a-8d3a-51771b37bfe8",
    "is_free_item": null,
    "bogo_details": {
      "bogo_group_id": "a76a9271-…", "bogo_offer_id": 3, "bundle_id": 5,
      "offer_title": "Joy Bridges", "offer_slug": "joy-bridges",
      "quantity": 2, "bundle_price": 1176, "total_price": 2352,
      "is_available": true, "unavailable_reason": null,
      "buy_items":  [ { "item_id": 67, "quantity": 2, "price": 836, "…" : null },
                      { "item_id": 59, "quantity": 2, "price": 340, "…" : null } ],
      "free_items": [ { "item_id": 69, "quantity": 2, "price": 0,   "…" : null } ] } }
]
```

---

### `POST /customer/cart/update`

| Param | Type | Required | Notes |
|---|---|---|---|
| `guest_id` | integer | required unless authenticated | |
| `cart_id` | integer | ordinary only | The row to change. |
| `bogo_group_id` | string | bundle only | Presence selects bundle mode. |
| `quantity` | integer ≥ 1 | **yes** | Units, or **bundles** in bundle mode. |
| `price` | numeric | ordinary only | |
| `restaurant_id` | integer\|slug | ordinary only | |

A bundle is **atomic** — its rows are rewritten from the enrolment, keeping the same
`bogo_group_id`.

**Response:** ordinary → `200` with the cart array. Bundle → `200` `{ message, bogo_group_id,
quantity, data }`.

**Errors:** `403` `bogo_offer` when `cart_id` points at a bundle row ("Use the offer actions to
change this bundle") · `403` `bogo_offer` when a rule fails at the new quantity · `404`
`bogo_group_id` unknown group · `404` `cart` unknown row.

---

### `DELETE /customer/cart/remove-item`

| Param | Type | Required | Notes |
|---|---|---|---|
| `guest_id` | integer | required unless authenticated | |
| `cart_id` | integer | ordinary only | |
| `bogo_group_id` | string | bundle only | Removes the **whole** group. |
| `restaurant_id` | integer\|slug | ordinary only | |

**Response:** `200` with the remaining cart. **Errors:** `403` `bogo_offer` when `cart_id` is a
bundle row · `404` `bogo_group_id` unknown group.

---

### `DELETE /customer/cart/remove`

| Param | Type | Required |
|---|---|---|
| `guest_id` | integer | required unless authenticated |
| `restaurant_id` | integer\|slug | yes |

Clears the whole cart, bundles included. `200` with `[]`.

---

### `GET /customer/cart/get-all`

Unchanged. Returns **raw, ungrouped rows** grouped by restaurant — bundle rows appear
individually with their tag columns, and there is no `bogo_details`.

```json
[ { "restaurant": { "id": 2, "name": "Hungry Puppets", "…" : null }, "carts": [ "…rows…" ] } ]
```

---

### `POST /customer/cart/add-multiple`

Unchanged, ordinary items only. Takes `item_list` (array of add payloads) — **not** `cart`.
A bundle cannot be added through it.

---

### When a bundle cannot be added or ordered

Checked at **add-to-cart and again at placement**, so a limit reached while the cart sat idle
cannot slip through. Returned as `403` on the write, or as `unavailable_reason` on the cart
entry:

| Reason | Message |
|---|---|
| offer switched off | This offer is no longer available |
| not started | This offer has not started yet |
| ended | This offer has already ended |
| restaurant left the offer | This restaurant is no longer part of the offer |
| restaurant unavailable | This restaurant is currently unavailable |
| whole-offer cap reached | This offer has reached its usage limit |
| per-customer cap reached | You can only use this offer N more times |
| food gone / disabled / out of stock | `<food>` is no longer available / is currently unavailable / Only N Quantity is available for `<food>` |

Free items are checked too — a bundle whose free item cannot be fulfilled is not orderable.
A stale bundle **stays in the cart** flagged with its reason and blocks checkout, rather than
disappearing on its own.

---

## Placing an order

Nothing BOGO-specific to call. `POST /customer/order/place` carries the bundle through:

- `order_details` rows keep `bogo_offer_id`, `bogo_group_id` and `is_free_item`
- the **free line is charged 0** — the order does not re-price it from the menu
- one `bogo_offer_usages` row per bundle group, with `quantity` = number of bundles (not
  lines), keyed to the user or, for a guest, the phone number
- the offer's `total_uses` is incremented by the same amount
- placement is refused with `403` if any bundle in the cart has become unorderable

```json
{
  "order_id": 100182,
  "details": [
    { "food_id": 67, "quantity": 1, "price": 800, "is_free_item": 0, "bogo_group_id": "6f1c…" },
    { "food_id": 59, "quantity": 1, "price": 340, "is_free_item": 0, "bogo_group_id": "6f1c…" },
    { "food_id": 69, "quantity": 1, "price": 0,   "is_free_item": 1, "bogo_group_id": "6f1c…" },
    { "food_id": 57, "quantity": 1, "price": 160, "is_free_item": 0, "bogo_group_id": null }
  ],
  "usage": { "bogo_offer_id": 3, "quantity": 1, "is_guest": 1, "phone": "+8801700009999" }
}
```

### Happy Hour on the order

When a Happy Hour is running, the order records it and the money follows:

| Field | Value |
|---|---|
| `restaurant_discount_amount` | the discount |
| `discount_on_product_by` | `vendor` — the restaurant bears it |
| `happy_hour_id` | the Happy Hour that produced it |

The restaurant bears the **full** discount as one `discount_on_product` expense; no share is
attributed to the admin, and the admin's commission is charged on the post-discount total, as
it is for any order. Reports pick it up through the existing vendor product-discount figure,
and `happy_hour_id` makes it separately attributable.

### What the free item costs, and who pays for it

A free line is charged 0, so without help the give-away would leave no trace at all — the
order would simply never mention the item that left the kitchen. `orders.bogo_discount_amount`
records its menu value.

| Field | Value |
|---|---|
| `bogo_discount_amount` | menu value of the free lines, `0` when there is no bundle |

It is deliberately **not** part of `restaurant_discount_amount`. That column reduces what the
customer pays; this does not — the customer was never billed for the item.

The restaurant bears it in full, exactly as it bears a Happy Hour: one `discount_on_product`
expense created by `vendor`, and it is added into the transaction's
`discount_amount_by_restaurant`. No money moves as a result — the customer was not charged for
the item and the admin never took commission on it — so `admin_expense`, `admin_commission`
and `restaurant_amount` are all untouched by it.

### Tax applies to the buy items only

A free line reaches the tax module priced `0` and discounted `0`, so its taxable base is `0`
and it contributes nothing. Tax is charged on the buy items after the discount:

```
bundle 800 + 340 + free, plus 2 x 160, Happy Hour 25%
   food 1,460 + add-ons 36                    = 1,496
   less Happy Hour 365                        = 1,131      <- taxed
   tax at 10%                                 =   113.10
   free item (menu value 165) is not taxed
```

Order-level totals were already right; what this affects is the **per-item** base, which is
what a `product_wise` or `category_wise` tax setup charges against, and what item-wise
discount reports sum.

---

# Vendor app API

The restaurant app's side of BOGO and Happy Hour, mirroring the vendor panel exactly. **12
endpoints.** A restaurant never creates or edits an offer — it joins one the admin published,
answers one the admin assigned it, reworks a rejected selection, or leaves.

Every sample below is a real request and response captured from the demo dataset.

## Environment

| Variable | Value |
|---|---|
| `base_url` | `https://stackfood-admin.test/api/v1` |
| `token` | the vendor's `auth_token`, returned by `POST {{base_url}}/auth/vendor/login` |
| `vendor_type` | `owner` or `employee` |

## Headers — every request

| Key | Value | Required | Notes |
|---|---|---|---|
| `Authorization` | `Bearer {{token}}` | ✅ | missing or unknown → `401` |
| `vendorType` | `owner` or `employee` | ✅ | missing → `403 vendor_type` |
| `Accept` | `application/json` | ✅ | |
| `Content-Type` | `application/json` | on POST | |
| `X-localization` | `en` | optional | response language |

## Permissions

Both groups are guarded by the same module check the panel uses, with the keys `bogo_offer`
and `happy_hour` — the ones the vendor role screen already writes.

- **Owner** (`vendorType: owner`) always passes.
- **Employee** (`vendorType: employee`) must have the module in its role.

```json
403  { "errors": [ { "code": "permission_denied", "message": "Access denied" } ] }
```

The module list an employee holds comes back on login as `role`, and on
`GET /vendor/profile` as `roles` — hide the menu with it, but expect the server to refuse too.

## Error format

Every failure uses the same envelope, so one handler covers all of them:

```json
{ "errors": [ { "code": "status", "message": "…" } ] }
```

| HTTP | Meaning |
|---|---|
| `401` | no token, or a token that matches no vendor |
| `403` | refused — permission, validation, or a rule of the state machine |
| `404` | no such offer for this restaurant (includes another zone's happy hour) |

## The enrolment state machine

Both features share one pivot and one state machine. `enrollment_state` collapses the stored
`status` and `requested_by` into a single word, so the app never has to combine them:

| `enrollment_state` | Means | Buttons to show |
|---|---|---|
| `not_joined` | no row yet | **Join** *(when `is_eligible`)* |
| `pending` | restaurant asked, admin decides | **Leave** |
| `admin_requested` | admin assigned, restaurant decides | **Approve** · **Deny** · **Edit & Accept** (BOGO) |
| `approved` | live | **Leave** |
| `rejected` | admin refused; `rejection_reason` is set | **Resubmit** (BOGO) · **Leave** |

Every response carries a resolved `actions` object:

```json
{ "actions": { "can_join": false, "can_resubmit": true, "can_respond": true, "can_leave": false } }
```

**Render buttons from `actions`, not from `status`.** The panel and the API resolve them from
the same code, so the two can never disagree.

**Leaving an unanswered admin request is refused.** Deleting the row would discard the foods
the admin chose and leave no record of the refusal, so the restaurant must approve or deny.

---

# BOGO — `{{base_url}}/vendor/bogo-offer`

## 1. List offers

```http
GET {{base_url}}/vendor/bogo-offer/list
```

**Query params**

| Key | Type | Required | Default | Description |
|---|---|---|---|---|
| `search` | string | ✗ | — | matches the offer title |
| `limit` | integer | ✗ | `25` | page size |
| `offset` | integer | ✗ | `1` | page **number**, not a row offset |
| `type` | string | ✗ | `all` | `all` · `not_joined` · `pending` · `admin_requested` · `approved` · `rejected` |

```bash
curl -X GET "{{base_url}}/vendor/bogo-offer/list?limit=1&offset=1" \
  -H "Authorization: Bearer {{token}}" \
  -H "vendorType: owner" \
  -H "Accept: application/json"
```

**200 OK**

```json
{
  "total_size": 5,
  "limit": 1,
  "offset": 1,
  "all": 5,
  "not_joined": 5,
  "pending": 0,
  "admin_requested": 0,
  "approved": 0,
  "rejected": 0,
  "offers": [
    {
      "id": 11,
      "slug": "ar-probe-3",
      "title": "AR probe",
      "description": null,
      "image_full_url": null,
      "buy_qty": 2,
      "get_qty": 1,
      "order_types": ["delivery", "take_away", "dine_in"],
      "usage_limit_total": null,
      "usage_limit_per_customer": null,
      "total_uses": 0,
      "start_date": "2026-08-10",
      "end_date": "2026-09-11",
      "status": 1,
      "is_eligible": true,
      "ineligible_reason": null,
      "enrollment_state": "not_joined",
      "enrollment": null,
      "actions": {
        "can_join": true,
        "can_resubmit": false,
        "can_respond": false,
        "can_leave": false
      }
    }
  ]
}
```

**Response fields**

| Field | Type | Description |
|---|---|---|
| `total_size` | integer | rows matching the filter |
| `limit` / `offset` | integer | echoed back |
| `all`, `not_joined`, `pending`, `admin_requested`, `approved`, `rejected` | integer | tab counts, computed **before** `type` is applied, so the tabs keep their totals while one is open |
| `offers[]` | array | see below |
| `offers[].id` | integer | offer id |
| `offers[].slug` | string | usable in place of the id on `details` |
| `offers[].title` | string | |
| `offers[].description` | string \| null | |
| `offers[].image_full_url` | string \| null | |
| `offers[].buy_qty` | integer | units required on the **buy** side |
| `offers[].get_qty` | integer | units given free on the **get** side |
| `offers[].order_types` | string[] | `delivery` · `take_away` · `dine_in` |
| `offers[].usage_limit_total` | integer \| null | null = unlimited |
| `offers[].usage_limit_per_customer` | integer \| null | null = unlimited |
| `offers[].total_uses` | integer | bundles ordered so far |
| `offers[].start_date` / `end_date` | string \| null | `Y-m-d`; null end = open ended |
| `offers[].status` | integer | `1` running |
| `offers[].is_eligible` | boolean | may this restaurant join |
| `offers[].ineligible_reason` | string \| null | why not — show this instead of guessing |
| `offers[].enrollment_state` | string | see the state machine |
| `offers[].enrollment` | object \| null | null until the restaurant has a row |
| `offers[].actions` | object | four booleans |

An offer whose order types the restaurant cannot serve stays **visible but not joinable** —
`is_eligible: false` with a reason, rather than vanishing with no explanation.

## 2. Offer details

```http
GET {{base_url}}/vendor/bogo-offer/details/{id}
```

**Path params**

| Key | Type | Required | Description |
|---|---|---|---|
| `id` | integer \| string | ✅ | offer id **or** slug |

Same object as a list row, plus the frozen selection under `enrollment.buy_items` and
`enrollment.get_items`.

**200 OK**

```json
{
  "id": 3,
  "slug": "joy-bridges",
  "title": "Joy Bridges",
  "description": "Omnis repellendus Nisi laboriosam quia laborum…",
  "image_full_url": "https://stackfood-admin.test/storage/bogo_offer/2026-08-10-6a7a9a42893ab.webp",
  "buy_qty": 2,
  "get_qty": 1,
  "order_types": ["delivery", "take_away", "dine_in"],
  "usage_limit_total": null,
  "usage_limit_per_customer": null,
  "total_uses": 0,
  "start_date": "2026-08-11",
  "end_date": "2026-08-29",
  "status": 1,
  "is_eligible": true,
  "ineligible_reason": null,
  "enrollment_state": "pending",
  "enrollment": {
    "id": 112,
    "status": "pending",
    "requested_by": "restaurant",
    "rejection_reason": null,
    "bundle_price": 280,
    "joined_at": "2026-08-12 23:03:28",
    "buy_items": [
      {
        "food_id": 57,
        "food_name": "Steak Kebabs",
        "food_image_full_url": null,
        "quantity": 1,
        "price": 160,
        "variations": [],
        "variation_options": [],
        "add_on_ids": [],
        "add_on_qtys": [],
        "add_ons": []
      },
      {
        "food_id": 59,
        "food_name": "FRIED RICE",
        "food_image_full_url": null,
        "quantity": 1,
        "price": 120,
        "variations": [],
        "variation_options": [],
        "add_on_ids": [],
        "add_on_qtys": [],
        "add_ons": []
      }
    ],
    "get_items": [
      {
        "food_id": 62,
        "food_name": "Thai Fried Rice",
        "food_image_full_url": null,
        "quantity": 1,
        "price": 0,
        "variations": [],
        "variation_options": [],
        "add_on_ids": [],
        "add_on_qtys": [],
        "add_ons": []
      }
    ]
  },
  "actions": {
    "can_join": false,
    "can_resubmit": false,
    "can_respond": false,
    "can_leave": true
  }
}
```

**`enrollment` fields**

| Field | Type | Description |
|---|---|---|
| `id` | integer | enrolment row id |
| `status` | string | `pending` · `approved` · `rejected` — raw; prefer `enrollment_state` |
| `requested_by` | string | `restaurant` or `admin` |
| `rejection_reason` | string \| null | set when the admin refused |
| `bundle_price` | float | what the buy side costs; the get side is free |
| `joined_at` | string | `Y-m-d H:i:s` |
| `buy_items[]` / `get_items[]` | array | the **frozen** snapshot — a later edit to the food does not move a live offer |

**Item fields** (both sides)

| Field | Type | Description |
|---|---|---|
| `food_id` | integer | |
| `food_name` | string | name as it was when submitted |
| `food_image_full_url` | string \| null | |
| `quantity` | integer | |
| `price` | float | unit price incl. variation and add-ons; **always `0`** on the get side |
| `variations` | array | chosen variations |
| `variation_options` | array | |
| `add_on_ids` | integer[] | |
| `add_on_qtys` | integer[] | parallel to `add_on_ids` |
| `add_ons[]` | array | the same add-ons resolved to `{id, name, price}` |

An `admin_requested` enrolment the admin created without picking foods returns empty
`buy_items` / `get_items` — the restaurant can accept it as-is or use **Resubmit** to supply
its own selection.

**404**

```json
{ "errors": [ { "code": "bogo_offer", "message": "Not found" } ] }
```

## 3. Menu for the combination builder

```http
GET {{base_url}}/vendor/bogo-offer/foods
```

**Query params**

| Key | Type | Required | Description |
|---|---|---|---|
| `search` | string | ✗ | matches the food name |

Own menu only — the restaurant comes from the token, never from the request, so one vendor
cannot enumerate another's foods. Capped at 50 rows; use `search` to narrow.

**200 OK**

```json
{
  "foods": [
    {
      "id": 57,
      "name": "Steak Kebabs",
      "category_name": "Varieties",
      "description": "Steak Kebabs are the one of the tastiest summertime dinners!…",
      "image_full_url": null,
      "price": 160,
      "discounted_price": 136,
      "discount": 15,
      "discount_type": "percent",
      "veg": 0,
      "maximum_cart_quantity": 0,
      "variations": [],
      "add_ons": [],
      "is_available": true
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `price` | float | menu price |
| `discounted_price` | float | after the food's own discount — **not** what the bundle uses |
| `discount` / `discount_type` | float / string | `percent` or `amount` |
| `maximum_cart_quantity` | integer | `0` = no cap; otherwise caps this food's total per side |
| `variations` | array | `[{name, type, min, max, required, values:[{label, optionPrice}]}]` |
| `add_ons` | array | `[{id, name, price}]` |
| `is_available` | boolean | the food's status |

## 4. Join an offer

```http
POST {{base_url}}/vendor/bogo-offer/join/{id}
```

**Body**

| Key | Type | Required | Description |
|---|---|---|---|
| `buy_items` | array | ✅ | quantities must total the offer's `buy_qty` |
| `get_items` | array | ✅ | quantities must total the offer's `get_qty` |

**Item object** — one entry per line; the same food may appear on several lines:

| Key | Type | Required | Description |
|---|---|---|---|
| `food_id` | integer | ✅ | must belong to this restaurant |
| `quantity` | integer | ✅ | ≥ 1 |
| `variations` | array | ✗ | chosen variations |
| `variation_options` | array | ✗ | |
| `add_on_ids` | integer[] | ✗ | |
| `add_on_qtys` | integer[] | ✗ | parallel to `add_on_ids`; defaults to 1 each |

```bash
curl -X POST "{{base_url}}/vendor/bogo-offer/join/3" \
  -H "Authorization: Bearer {{token}}" \
  -H "vendorType: owner" \
  -H "Content-Type: application/json" \
  -d '{
        "buy_items": [ {"food_id": 57, "quantity": 1},
                       {"food_id": 59, "quantity": 1} ],
        "get_items": [ {"food_id": 62, "quantity": 1} ]
      }'
```

**200 OK**

```json
{
  "message": "Your request has been sent for approval",
  "bundle_price": 280,
  "enrollment_state": "pending"
}
```

| Field | Type | Description |
|---|---|---|
| `message` | string | ready to show |
| `bundle_price` | float | computed server-side from the buy side |
| `enrollment_state` | string | always `pending` — the admin is the gate |

**403 — quantities do not match the offer**

```json
{
  "errors": [
    { "code": "buy_items", "message": "You must add buy foods matching the offer quantity" },
    { "code": "get_items", "message": "You must add get foods matching the offer quantity" }
  ]
}
```

**403 — already joined**

```json
{ "errors": [ { "code": "restaurant", "message": "You have already joined this offer" } ] }
```

**Every refusal**

| `code` | When |
|---|---|
| `offer` | the offer is switched off or has ended |
| `order_type` | the restaurant serves none of the offer's order types |
| `restaurant` | an enrolment already exists — use `resubmit` or `leave` |
| `buy_items` / `get_items` | side quantities do not total `buy_qty` / `get_qty` |
| `food` | a food is not this restaurant's, or a quantity is below 1 |
| `cart_item_limit` | a food's total on one side exceeds its `maximum_cart_quantity` |
| `combination` | this exact buy+get combination is already used on another live offer |

## 5. Resubmit a selection

```http
POST {{base_url}}/vendor/bogo-offer/resubmit/{id}
```

Same body as **Join**. Allowed only when `actions.can_resubmit` is true — that is, when the
enrolment is `rejected` or `admin_requested`.

**200 OK**

```json
{
  "message": "Your request has been resubmitted",
  "bundle_price": 410,
  "enrollment_state": "pending"
}
```

A resubmit always lands as the **restaurant's own** pending request, even when reworking a row
the admin created: the selection is now the restaurant's, so the admin becomes the approver
rather than the restaurant being asked to approve its own edit. Any `rejection_reason` is
cleared.

**403**

```json
{ "errors": [ { "code": "status",
                "message": "Only a rejected or admin requested enrolment can be edited" } ] }
```

## 6. Respond to an admin request

```http
POST {{base_url}}/vendor/bogo-offer/respond/{id}
```

**Body**

| Key | Type | Required | Description |
|---|---|---|---|
| `status` | string | ✅ | `approved` or `rejected` |
| `rejection_reason` | string | ✗ | max 255; kept only when rejecting |

```bash
curl -X POST "{{base_url}}/vendor/bogo-offer/respond/3" \
  -H "Authorization: Bearer {{token}}" \
  -H "vendorType: owner" \
  -H "Content-Type: application/json" \
  -d '{"status": "approved"}'
```

**200 OK**

```json
{ "message": "You have joined the BOGO offer" }
```

Rejecting returns `"You have declined the BOGO offer"` and emails the admin.

**403 — nothing to answer**

```json
{ "errors": [ { "code": "status",
                "message": "There is no pending admin request for this offer" } ] }
```

Only an **admin-initiated pending** row can be answered — a restaurant cannot approve a request
it raised itself.

## 7. Leave an offer

```http
DELETE {{base_url}}/vendor/bogo-offer/leave/{id}
```

**200 OK**

```json
{ "message": "You have left the BOGO offer" }
```

The enrolment and its frozen items are deleted together.

**403 — an unanswered admin request**

```json
{ "errors": [ { "code": "status",
                "message": "Approve or deny the admin request instead of leaving" } ] }
```

---

# Happy Hour — `{{base_url}}/vendor/happy-hour`

Scoped to the restaurant's **zone**. A happy hour in another zone returns `404`, not `403` — it
does not reach this restaurant at all.

**There is no `resubmit`.** There is no food selection to rework, so a rejected restaurant
simply joins again and the same pivot row is reused. `actions.can_resubmit` is always `false`.

## 8. List happy hours

```http
GET {{base_url}}/vendor/happy-hour/list
```

Same query params as the BOGO list (`search`, `limit`, `offset`, `type`).

**200 OK**

```json
{
  "total_size": 2,
  "limit": 1,
  "offset": 1,
  "all": 2,
  "not_joined": 2,
  "pending": 0,
  "admin_requested": 0,
  "approved": 0,
  "rejected": 0,
  "happy_hours": [
    {
      "id": 134,
      "slug": "capture-happy-hour",
      "title": "CAPTURE Happy Hour",
      "short_description": "Evening window",
      "cover_image_full_url": null,
      "icon_full_url": null,
      "zone_id": 1,
      "zone_name": "All over the World",
      "discount": 20,
      "min_order_amount": 0,
      "is_permanent": false,
      "duration_type": "daily",
      "weekly_days": [],
      "start_date": "2026-08-11",
      "end_date": "2026-08-15",
      "start_time": "22:43:41",
      "end_time": "23:43:41",
      "is_running_now": true,
      "status": 1,
      "is_eligible": true,
      "ineligible_reason": null,
      "enrollment_state": "not_joined",
      "enrollment": null,
      "actions": {
        "can_join": true,
        "can_resubmit": false,
        "can_respond": false,
        "can_leave": false
      }
    }
  ]
}
```

**Response fields**

| Field | Type | Description |
|---|---|---|
| `discount` | float | percent off, applied to the whole restaurant while the window is open |
| `min_order_amount` | float | |
| `is_permanent` | boolean | a permanent weekly rule has no end date and never expires |
| `duration_type` | string | `daily` · `weekly` · `custom` |
| `weekly_days` | string[] | day names, for a permanent weekly rule |
| `start_date` / `end_date` | string \| null | `Y-m-d`; null on a permanent rule |
| `start_time` / `end_time` | string | `H:i:s`; the window is a fixed one hour |
| `is_running_now` | boolean | **is the window open at this moment** — resolved server-side, since a permanent rule matches on weekday and everything else on a dated row |
| `zone_name` | string \| null | |

## 9. Happy hour details

```http
GET {{base_url}}/vendor/happy-hour/details/{id}
```

`id` accepts the id **or** the slug. Adds the upcoming schedule:

```json
{
  "id": 134,
  "title": "CAPTURE Happy Hour",
  "is_running_now": true,
  "enrollment_state": "not_joined",
  "enrollment": null,
  "actions": { "can_join": true, "can_resubmit": false,
               "can_respond": false, "can_leave": false },
  "dates": [
    { "applicable_date": "2026-08-12", "start_time": "22:43:41",
      "end_time": "23:43:41", "status": true }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `dates[]` | array | today onward only; a permanent weekly rule returns `[]` and is described by `weekly_days` |
| `dates[].applicable_date` | string | `Y-m-d` |
| `dates[].start_time` / `end_time` | string | `H:i:s` |
| `dates[].status` | boolean | whether that occurrence is active |

## 10. Join a happy hour

```http
POST {{base_url}}/vendor/happy-hour/join/{id}
```

No body — there is nothing to select.

**200 OK**

```json
{ "message": "Your request has been sent for approval", "enrollment_state": "pending" }
```

**403**

```json
{ "errors": [ { "code": "happy_hour",
                "message": "You have already requested to join this happy hour" } ] }
```

| `code` | When |
|---|---|
| `happy_hour` | switched off, ended, already requested, or already joined |

## 11. Respond to an admin request

```http
POST {{base_url}}/vendor/happy-hour/respond/{id}
```

**Body** — identical to the BOGO respond: `status` (`approved` \| `rejected`) and an optional
`rejection_reason`.

```bash
curl -X POST "{{base_url}}/vendor/happy-hour/respond/127" \
  -H "Authorization: Bearer {{token}}" \
  -H "vendorType: owner" \
  -H "Content-Type: application/json" \
  -d '{"status": "rejected", "rejection_reason": "Not this week"}'
```

**200 OK**

```json
{ "message": "You have declined the happy hour" }
```

A declined restaurant may join again later; the same row is reused rather than duplicated.

## 12. Leave a happy hour

```http
DELETE {{base_url}}/vendor/happy-hour/leave/{id}
```

**200 OK**

```json
{ "message": "You have left the happy hour" }
```

Allowed at any time, including while the window is live — the same rule the admin side follows
for removing a restaurant. Refused only for an unanswered admin request.

---

## Endpoint summary

| # | Method | Path | Purpose |
|---|---|---|---|
| 1 | GET | `/vendor/bogo-offer/list` | offers + my state + tab counts |
| 2 | GET | `/vendor/bogo-offer/details/{id}` | one offer + frozen selection |
| 3 | GET | `/vendor/bogo-offer/foods` | own menu for the builder |
| 4 | POST | `/vendor/bogo-offer/join/{id}` | join with a buy/get selection |
| 5 | POST | `/vendor/bogo-offer/resubmit/{id}` | rework a rejected or admin-picked selection |
| 6 | POST | `/vendor/bogo-offer/respond/{id}` | approve or deny an admin request |
| 7 | DELETE | `/vendor/bogo-offer/leave/{id}` | leave |
| 8 | GET | `/vendor/happy-hour/list` | zone happy hours + my state |
| 9 | GET | `/vendor/happy-hour/details/{id}` | one happy hour + schedule |
| 10 | POST | `/vendor/happy-hour/join/{id}` | join |
| 11 | POST | `/vendor/happy-hour/respond/{id}` | approve or deny an admin request |
| 12 | DELETE | `/vendor/happy-hour/leave/{id}` | leave |


---

## Known issues

Known and **not fixed**. Both pre-date BOGO; the first is reachable through it, the second is
not BOGO-specific at all.

1. **Add-ons are not discounted at checkout.** The discount applies to `product_price`, which
   excludes add-ons, while `final_price` is quoted off the whole `bundle_price`. A bundle
   containing add-ons is therefore quoted slightly below what it bills (in the sample data,
   882 quoted vs 891 charged).

2. **`discount_on_food` means two different things.** A product's own discount is written per
   unit; a restaurant-wide discount or Happy Hour is written per line
   (`checkAdminDiscount()` is passed `price x quantity`). Everything downstream multiplies by
   quantity — `sum(discount_on_food * quantity)` in the admin sales reports, and
   `getFinalCalculatedTax()` when `discount_type` is `discount_on_product` — so a line with
   `quantity > 1` under a restaurant-wide discount is counted twice over. This is unrelated to
   BOGO: it reproduces on a plain single-item order with `quantity 2` and no offer, and it
   affects every discounted multi-quantity order in the platform. Left alone because changing
   it moves numbers on existing reports.

Two more sit in the **ordinary** cart path and pre-date this work entirely — they reproduce on
a clean cart with no offer involved, and clients only avoid them by always sending the fields:

3. **`add_to_cart` 500s when `add_on_ids` is omitted** — `array_combine(): Argument #1 must be
   of type array, null given`.

4. **`add_to_cart` with no `variations` stores the literal `"null"`**, and `GET /cart/list`
   then 500s at `Helpers::cart_product_data_formatting()` with
   `foreach() argument must be of type array|object, null given`.

Also outstanding: POS and admin order-edit do not handle bundles, and refunds do not release
a recorded usage.
