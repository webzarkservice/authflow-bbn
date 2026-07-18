# BizBiteNow Backend — API Documentation

Base URL: `http://<host>/api`
Auth: Cookie (`token`, httpOnly) set on login, or Bearer flow via same JWT. Protected routes need valid `token` cookie which `authMiddleware` decodes into `req.user` (and in most seller-scoped routes, `req.seller`).

Tiers referenced below: `FREE`, `PRO`, `PLUS` (seller subscription tier — gates feature access).

---

# 🔐 AUTH APIs (Shared — `/api/users`)

These are common to Seller, Customer and Admin, mounted at `/api/users`.

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/users/admin/register-seller` | Admin only | Admin onboards a new seller |
| POST | `/users/login/init` | Public | Step 1 of login — checks role, sends OTP if Seller |
| POST | `/users/login` | Public | Step 2 — verifies PIN (+ OTP token for sellers) |
| POST | `/users/forgot-pin` | Public | Sends OTP for PIN reset |
| POST | `/users/reset-pin` | Public | Resets PIN using verificationToken |
| POST | `/users/change-pin` | Protected | Logged-in user changes PIN |
| POST | `/users/customer/register` | Protected (Seller token) | Seller registers a walk-in customer |
| POST | `/users/verify-access-token` | Public | MSG91 access token verify |
| POST | `/users/send-otp` | Public | Sends OTP via MSG91 |
| POST | `/users/verify-otp` | Public | Verifies OTP, issues short-lived `verificationToken` |
| POST | `/users/resend-otp` | Public | Resends OTP |

### POST `/users/admin/register-seller`
**Auth:** requires `req.user.role === "Admin"`
**Body:**
```json
{
  "name": "string", "email": "string", "phoneNumber": "string",
  "business_name": "string", "business_type": "string",
  "pin": "string/number", "address": "string", "city": "string",
  "state": "string", "tier": "FREE|PRO|PLUS (optional, default FREE)"
}
```
**Response 201:**
```json
{ "success": true, "message": "...", "user": {...}, "seller": { "id", "business_name" } }
```

### POST `/users/login/init`
**Body:** `{ "identifier": "phoneNumber" }`
- If user role ≠ Seller → returns `{ success, role, requiresOtp: false }` (client proceeds straight to PIN login).
- If role === Seller → internally triggers OTP send, response same shape as `/send-otp`.

### POST `/users/login`
**Body:**
```json
{ "identifier": "phone", "pin": "1234", "fcm_token": "optional", "verificationToken": "required only for Seller (from verify-otp)" }
```
**Response 200:**
```json
{
  "success": true, "is_new": false, "message": "...",
  "user": { "id", "name", "role", "phone" },
  "profile": { /* Admin: access_level/permissions | Seller doc | Customer doc */ },
  "token": "jwt"
}
```
Also sets `token` httpOnly cookie.

### POST `/users/forgot-pin`
Body: `{ "phoneNumber": "string" }` → Response: `{ success, message, role, reqId }`

### POST `/users/reset-pin`
Body: `{ "verificationToken", "newPin", "confirmPin" }` (verificationToken must have `purpose: "FORGOT_PIN"`)

### POST `/users/change-pin` (Protected)
Body: `{ "currentPin", "newPin", "confirmPin" }`

### POST `/users/customer/register` (Protected — Seller token)
Body:
```json
{
  "customer_name", "customer_phone", "pin", "birthday",
  "mohalla", "delivery_address", "verificationToken"
}
```
Response 201/200: `{ success, message, customer: {id,name,phone,stamps_earned,rewards}, customer_token }` + sets cookie.

### OTP flow (all Public)
- `POST /users/send-otp` — Body: `{ "mobile" or "identifier" }` → `{ success, message, data }`
- `POST /users/verify-otp` — Body: `{ "reqId", "otp", "purpose": "REGISTER|LOGIN|FORGOT_PIN" }` → `{ success, verificationToken, mobile }`
- `POST /users/resend-otp` — Body: `{ "reqId" }`
- `POST /users/verify-access-token` — Body: `{ "accessToken" }`

---

# 🏪 SELLER APIs

## 1. Seller Profile — `/api/seller`
All protected (`authMiddleware`, uses `req.user.seller_id`).

| Method | Endpoint | Description |
|---|---|---|
| PUT | `/seller/profile/branding` | Upload logo/banner (multipart: `logo`, `banner[]` max 5) + store_name/tagline/description |
| POST | `/seller/complete-onboarding` | Fill full business profile (tier-gated fields) |
| POST | `/seller/logout` | Clears auth cookie |
| GET | `/seller/me` | Get own seller profile |
| PUT | `/seller/me/location` | Set store lat/lng |
| DELETE | `/seller/me` | Deactivate seller (soft delete) |
| PUT | `/seller/me` | Update a profile section |

### PUT `/seller/profile/branding`
multipart/form-data: `logo` (1 file), `banner` (up to 5), plus text fields `store_name`, `tagline`, `description`.
Response: `{ success, message, store_profile }`

### POST `/seller/complete-onboarding`
Body (all optional except business_type/address/city/state):
```json
{
  "business_type", "cuisine_categories", "fssai_license", "pan_number",
  "business_registration_number", "business_email", "alternate_phone",
  "website", "whatsapp_business", "google_maps_url", "address", "city", "state",
  "tagline", "description", "theme_color", "custom_domain",
  "business_hours", "delivery_settings", "payment_settings",
  "tax_settings", "notifications", "security"
}
```
⚠️ FREE tier is blocked from: `theme_color`, `custom_domain`, `delivery_settings`, `tax_settings`, UPI in `payment_settings.accepted_methods`.
Response: `{ success, message, seller }`

### GET `/seller/me`
Response: `{ success, seller }`

### PUT `/seller/me/location`
Body: `{ "latitude": number, "longitude": number }` → Response: `{ success, message, store_location }`

### DELETE `/seller/me`
Deactivates seller + linked user, clears cookie. Response: `{ success, message, seller }`

### PUT `/seller/me`
Body: `{ "section": "store_profile|business_info|contact_info|business_hours|delivery_settings|payment_settings|tax_settings|notifications|security", "data": { ...key:value } }`
Response: `{ success, message, seller }`

---

## 2. Menu Categories — `/api/menu-categories`
Protected unless noted. Uses `req.seller` (seller_id + tier). Tier limits: FREE=5, PRO=20, PLUS=∞ categories. Reorder needs PRO/PLUS.

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/menu-categories/create` | Protected | Create category |
| GET | `/menu-categories/get` | Protected | List seller's categories |
| PUT | `/menu-categories/reorder` | Protected (PRO/PLUS) | Bulk reorder via `ordered_ids` |
| PUT | `/menu-categories/:id` | Protected | Update category |
| DELETE | `/menu-categories/:id` | Protected | Delete (blocked if products linked) |
| GET | `/menu-categories/full` | Protected | Full menu (seller dashboard) |
| GET | `/menu-categories/full/:seller_id` | Public | Full menu for storefront |

Body — create/update: `{ "name", "description", "display_order" }`
Body — reorder: `{ "ordered_ids": ["catId1","catId2",...] }`
`getFullMenu` response: `{ success, menu: [{ category_id, name, description, display_order, products: [...] }] }` (includes an "Other Items" bucket for uncategorized products).

---

## 3. Products — `/api/product`
Uses `req.seller`.

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/product/add` | Protected | Create product (multipart `image`) |
| PUT | `/product/update/:id` | Protected | Update product (optional new `image`) |
| DELETE | `/product/delete/:id` | Protected | Delete product |
| GET | `/product/dashboard/all` | Protected | All seller products |
| GET | `/product/storefront/:seller_id` | Public | Same list, public view (reuses `getSellerProducts`) |
| GET | `/product/:seller_id/products` | Public | Storefront catalog (filter by `?category=`, `?search=`) |
| GET | `/product/:seller_id/categories` | Public | Distinct available categories for storefront |

Body — add/update: `name, price, description, category` (+ multipart `image`); update also allows `is_available` (boolean toggle).
Response (create): `{ success, message, product }`
Response (list): `{ success, count, products }`

---

## 4. Combos — `/api/combos`
Requires PRO/PLUS tier to create (`FREE` limit = 0). Uses `req.seller`.

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/combos/` | Protected (PRO/PLUS) | Create combo (multipart `image`) |
| GET | `/combos/` | Protected | List seller combos (with computed `savings`) |
| PUT | `/combos/:id` | Protected | Update combo |
| DELETE | `/combos/:id` | Protected | Delete combo |
| GET | `/combos/storefront/:seller_id` | Public | Active combos for storefront |

Body: `{ "name", "description", "items": [{"product_id","quantity"}] (JSON or array), "combo_price" }`
Rule: `combo_price` must be strictly less than sum of individual item prices.
Response: `{ success, message, combo, savings }`

---

## 5. Discounts — `/api/discounts`
Requires PRO/PLUS to create (limits: PRO=5, PLUS=∞ **active** codes). Uses `req.seller`.

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/discounts/` | Protected (PRO/PLUS) | Create discount code |
| GET | `/discounts/` | Protected | List all discounts |
| PUT | `/discounts/:id` | Protected | Update discount |
| DELETE | `/discounts/:id` | Protected | Soft-deactivate discount |
| POST | `/discounts/check` | Public | Validate code at checkout preview |

Body — create:
```json
{
  "code","description","discount_type":"percentage|flat","discount_value",
  "max_discount_cap","min_order_value","usage_limit_total",
  "usage_limit_per_customer","valid_from","valid_until"
}
```
Body — `/discounts/check`: `{ "seller_id","code","customer_phone","cart_total" }` → `{ success, discount_amount, final_amount }`

---

## 6. Festive Deals — `/api/festive-deals`
Requires PRO/PLUS to create (limits on **simultaneously active** deals: PRO=3, PLUS=∞). Uses `req.seller`.

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/festive-deals/` | Protected (PRO/PLUS) | Create deal (multipart `banner_image`) |
| GET | `/festive-deals/` | Protected | List seller's deals |
| PUT | `/festive-deals/:id` | Protected | Update deal |
| DELETE | `/festive-deals/:id` | Protected | Delete deal |
| GET | `/festive-deals/storefront/:seller_id` | Public | Currently active deals |

Body:
```json
{
  "title","description","discount_type":"percentage|flat","discount_value",
  "max_discount_cap","min_order_value",
  "applies_to_all_products": true/false,
  "applicable_products": ["productId",...] (required if applies_to_all_products=false),
  "start_date","end_date"
}
```

---

## 7. Tables (Dine-in QR) — `/api/tables`
Seller/admin managed. Uses `req.seller`.

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/tables/create` | Protected | Create table + generates QR + ordering URL |
| GET | `/tables/list` | Protected | List all tables with regenerated QR |
| PUT | `/tables/toggle/:table_id` | Protected | Toggle table active/inactive |
| GET | `/tables/resolve/:token` | Public | Customer scans QR → resolves seller + menu |

Body — create: `{ "table_number": "string/number" }`
Response — create: `{ success, message, table, ordering_url, qr_code (base64) }`
Response — resolve: `{ success, seller_id, table_number, store_name, upi_available, menu }`

---

## 8. Delivery Boys — `/api/deliveryBoy`
**PLUS tier only.** Uses `req.seller` + `requirePlus` middleware.

| Method | Endpoint | Description |
|---|---|---|
| POST | `/deliveryBoy/create` | Add delivery boy `{ "name","phoneNumber" }` |
| GET | `/deliveryBoy/list` | List all delivery boys |
| PUT | `/deliveryBoy/update/:id` | Update `{ "name","phoneNumber","is_available" }` |
| DELETE | `/deliveryBoy/delete/:id` | Remove delivery boy |

---

## 9. Orders — Seller-side (`/api/orders`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/orders/assign` | Protected (PLUS) | Assign order to a delivery boy |
| GET | `/orders/list` | Protected | List orders (optional `?order_type=`) — history window by tier (FREE=today, PRO=30 days, PLUS=all) |
| PUT | `/orders/update/:id` | Protected | Update `delivery_status` |
| DELETE | `/orders/delete/:id` | Protected | Delete an order |
| GET | `/orders/dashboard/counters` | Protected (PRO+) | Aggregate stats (totalOrders, unassigned/assigned/delivered counts, revenue) |
| GET | `/orders/dashboard/clusters` | Protected (PLUS) | Group pending orders by `mohalla` |
| PUT | `/orders/dine-in/mark-paid/:order_id` | Protected | Mark a dine-in order as paid (cash/manual confirm) |
| GET | `/orders/verify/:token` | Public (no auth — delivery-boy magic link) | Marks order Delivered + handles loyalty stamp/reward |

### POST `/orders/assign`
Body: `{ "order_id", "delivery_boy_id" }`
Response: `{ success, message, delivery_link, navigation_link, data:{order_id,status} }`

### GET `/orders/list`
Query: `?order_type=dine-in|delivery|takeaway` (optional)
Response: `{ success, count, orders }`

### PUT `/orders/update/:id`
Body: `{ "delivery_status" }`

### GET `/orders/dashboard/counters`
Response: `{ success, counters: { totalOrders, unassignedCount, assignedCount, deliveredCount, totalRevenue } }`

### GET `/orders/dashboard/clusters`
Response: `{ success, clusters: [{ _id: mohalla, pendingOrdersCount, orders:[{order_id,customer_name,amount}] }] }`

### GET `/orders/verify/:token`
Public HTML response (delivery boy taps link) — not JSON. Marks delivered, updates loyalty stamps, may issue reward coupon on PLUS tier.

---

# 🛒 CUSTOMER APIs

## 1. Customer Profile — `/api/customer`

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/customer/register` | Protected (Seller context) | Same as `userAuth.registerCustomer` — seller adds walk-in customer |
| GET | `/customer/mohallas/:seller_id` | Public | List available delivery localities for a seller |
| PUT | `/customer/update` | Protected | ⚠️ uses `req.params.id` internally but route has no `:id` — check route/controller mismatch before use |
| DELETE | `/customer/delete` | Protected | Same `:id` mismatch caveat as above |
| POST | `/customer/loyalty/status` | Protected (seller token) | Get stamps/rewards progress |
| POST | `/customer/loyalty/claim` | Protected (seller token) | Redeem 10 stamps → coupon |
| POST | `/customer/address/save` | Protected (seller token) | Save/update delivery address & mohalla |

> ⚠️ Note for Raghav: `router.put("/update", ...)` and `router.delete("/delete", ...)` don't declare an `:id` param, but `updateCustomer`/`deleteCustomer` read `req.params.id`. Route should probably be `/update/:id` and `/delete/:id`. Flagging this so client integration doesn't silently break.

### GET `/customer/mohallas/:seller_id`
Response: `{ success, seller_id, count, mohendra_locations: ["Vikas Nagar", ...] }` *(hardcoded list currently — not DB driven)*

### POST `/customer/loyalty/status`
Body: `{ "customer_phone" }` (seller_id comes from token)
Response: `{ success, customer_name, stamps_earned, max_stamps, stamps_remaining_for_reward, rewards_history }`

### POST `/customer/loyalty/claim`
Body: `{ "customer_phone" }`
Response: `{ success, message, coupon_code, reward_item, remaining_stamps }` (needs ≥10 stamps)

### POST `/customer/address/save`
Body: `{ "customer_name","customer_phone","mohalla","delivery_address" }`
Response: `{ success, message, customer }`

---

## 2. Orders — Customer-side (`/api/orders`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/orders/create` | Protected (customer/seller token) | Place a delivery/takeaway/dine-in order directly |
| GET | `/orders/customer-orders` | Protected | Get logged-in customer's order history |
| POST | `/orders/reorder` | Protected (PLUS) | Re-place a previous order |
| POST | `/orders/checkout/initiate` | Protected | Start Razorpay payment (online orders) |
| POST | `/orders/checkout/verify` | Public | Verify Razorpay payment signature |
| POST | `/orders/dine-in/create` | Public | Place dine-in order via QR `table_token` |
| POST | `/orders/webhook/razorpay` | Public (raw body, signature-verified) | Razorpay webhook — subscription/tier upgrade |

### POST `/orders/create`
Body:
```json
{
  "customer_name","customer_phone","items":[{"product_id","quantity"}],
  "payment_method": "COD|ONLINE|CASH|UPI_MANUAL",
  "order_type": "delivery|dine-in|takeaway (default delivery)",
  "delivery_address","latitude","longitude","mohalla",  // required if order_type=delivery
  "coupon_code",          // OR discount_code, never both
  "discount_code",
  "is_scheduled": false,
  "scheduled_for": "ISO date (required if is_scheduled true)"
}
```
Business rules:
- FREE tier sellers: `ONLINE` payment blocked (COD/CASH only).
- Delivery orders run a geofence check against seller's `delivery_radius_km` + `grace_radius_km`; `blocked` zone → 400.
- Only one of `coupon_code` (loyalty reward coupon) or `discount_code` (seller discount) allowed per order.
- Delivery fee auto-applied if `total < pro_settings.min_order_value` (non-FREE sellers).

Response 201:
```json
{
  "success": true, "message": "...", "order": {...},
  "savings": { "festive_deal_savings": number },
  "delivery_warning": "string|null"
}
```

### GET `/orders/customer-orders`
Body: `{ "customer_phone" }` → Response: `{ success, count, orders }`

### POST `/orders/reorder`
Body: `{ "previous_order_id","payment_method","coupon_code","discount_code" }` — rebuilds order from a past order id, revalidates prices/geofence/tier restrictions.

### POST `/orders/checkout/initiate`
Body: same as `/orders/create` minus `order_type`/scheduling (delivery-only flow). If `payment_method === "COD"` it internally forwards to `createOrder`.
Response 200: `{ success, message, razorpay_order_id, amount, currency, order_id, savings, delivery_warning }`

### POST `/orders/checkout/verify`
Body: `{ "razorpay_order_id","razorpay_payment_id","razorpay_signature","order_id" }`
Response: `{ success, message, order }`

### POST `/orders/dine-in/create`
Body: `{ "table_token","customer_name","customer_phone","items":[{"product_id","quantity"}],"payment_method":"CASH|UPI_MANUAL" }`
Response 201: `{ success, message, order, payment: { upi_link, qr_code, reference } | null }`

### POST `/orders/webhook/razorpay`
Raw JSON body, header `x-razorpay-signature` verified via HMAC using `RAZORPAY_WEBHOOK_SECRET`. Handles `order.paid`, `payment.captured`, `subscription.activated` to upgrade seller `tier`. No customer-facing response body beyond `{status:"ok"}`.

---

## 3. Storefront (Public, no auth) — Customer browsing

| Method | Endpoint | Description |
|---|---|---|
| GET | `/menu-categories/full/:seller_id` | Full categorized menu |
| GET | `/product/:seller_id/products` | Product catalog (`?category=`, `?search=`) |
| GET | `/product/:seller_id/categories` | Available category list |
| GET | `/combos/storefront/:seller_id` | Active combos |
| GET | `/festive-deals/storefront/:seller_id` | Active festive deals |
| GET | `/tables/resolve/:token` | Resolve QR table token → seller + menu |
| GET | `/customer/mohallas/:seller_id` | Delivery locality list |
| POST | `/discounts/check` | Validate a discount code before checkout |

---

# ⚠️ Known Issues / Notes to Fix Later
1. **`/api/customer/update` & `/api/customer/delete`** routes are missing `:id` param but controllers expect `req.params.id` — will 404/undefined currently. Add `:id` to route path.
2. **Inconsistent auth context**: some controllers read `req.seller` (product, menuCategory, combo, discount, festiveDeal, table, deliveryBoy, order.assign/list/dashboard) while others read `req.user` (order.create/customer-orders, customer.controller, userAuth). Confirm `authMiddleware` sets **both** `req.user` and `req.seller`, or standardize on one.
3. **`payment.routes.js`** duplicates the Razorpay webhook handler already mounted at `/api/orders/webhook/razorpay` — but `payment.routes.js` itself isn't `app.use()`'d in `app.js`, so it's currently dead code.
4. **GET `/product/storefront/:seller_id`** reuses `getSellerProducts` (dashboard-style, no `is_available` filter) instead of `getStorefrontProducts` — may leak unavailable products publicly.
5. `getAvailableMohallas` returns a hardcoded list, not DB-driven — fine for MVP, flag for real launch.
