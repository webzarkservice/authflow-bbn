# Auth API Documentation

Base path: `/api/users`

This API handles authentication and account management for three roles: Admin, Seller, and Customer. Sellers use a two-step login process with OTP verification. Admins and Customers log in directly with a PIN.

---

## 1. Seller Onboarding

### POST `/admin/register-seller`
**Access:** Admin only (requires auth token)

Creates a new Seller account. Only an Admin can call this endpoint.

**Required fields:**
- name
- email
- phoneNumber
- business_name
- business_type
- pin
- address
- city
- state

**Behavior:**
- Checks that the requester is an Admin.
- Checks if a user with the same email or phone number already exists.
- Hashes the PIN before saving.
- Creates a User record and a linked Seller profile.
- If the Seller profile creation fails, the User record is rolled back (deleted).

**Response:** Returns the created user and seller details. No login token is issued here, since the Admin is creating the account on behalf of the seller.

---

## 2. Login Flow

### POST `/login/init`
Starting point for login. Used to detect which role a phone number belongs to.

**Input:** `identifier` (phone number)

**Behavior:**
- Looks up the user by phone number.
- If the user is Admin or Customer, responds with `requiresOtp: false` so the frontend can go straight to the PIN screen.
- If the user is a Seller, this automatically triggers OTP sending.

### POST `/send-otp`
Sends an OTP to a phone number using the MSG91 widget service.

**Input:** `mobile` or `identifier`

### POST `/verify-otp`
Verifies the OTP entered by the user.

**Input:** `reqId`, `otp`, optional `purpose` (defaults to `REGISTER`)

**Behavior:**
- Verifies the OTP with MSG91.
- Verifies the resulting access token with MSG91 again.
- If both checks pass, issues an internal `verificationToken` (a JWT valid for 15 minutes) containing the verified mobile number and purpose.

**Important:** For login, the `purpose` should be explicitly set to `LOGIN` when calling this endpoint, since the later `/login` step checks for this value.

### POST `/resend-otp`
Resends an OTP using the original `reqId`.

### POST `/login`
Final step of login. Used by all roles.

**Input:** `identifier`, `pin`, and `verificationToken` (verificationToken is required only when the user is a Seller)

**Behavior:**
- If the user is a Seller, the verificationToken is checked: it must be valid, marked as verified, have purpose `LOGIN`, and match the phone number being logged in.
- Compares the entered PIN against the stored hashed PIN.
- If a Customer has an old unhashed PIN, it falls back to a plain comparison against the Customer record.
- Updates the FCM token if one is provided.
- Builds a JWT payload based on role (includes seller_id, customer_id, or tier where relevant).
- Sets an auth cookie and returns the token along with user info and role-specific profile data.

---

## 3. PIN Management

### POST `/forgot-pin`
Sends an OTP to the given phone number to start the PIN reset process.

**Input:** `phoneNumber`

### POST `/reset-pin`
Resets the PIN after OTP verification.

**Input:** `verificationToken`, `newPin`, `confirmPin`

**Behavior:**
- Verifies the token was issued for purpose `FORGOT_PIN`.
- Checks that `newPin` and `confirmPin` match.
- Updates the user's PIN with a new hashed value.

### POST `/change-pin`
**Access:** Logged-in user (requires auth token)

Changes the PIN for the currently logged-in user.

**Input:** `currentPin`, `newPin`, `confirmPin`

**Behavior:**
- Verifies the current PIN is correct.
- Rejects if the new PIN is the same as the current one.
- Saves the new hashed PIN.

---

## 4. Customer Registration

### POST `/customer/register`
**Access:** Requires auth token

Registers a new customer, or updates an existing one if already registered.

**Input:** `customer_name`, `customer_phone`, `pin`, `birthday`, `mohalla`, `delivery_address`, `verificationToken`

**Behavior:**
- Verifies the `verificationToken` and checks that the verified mobile number matches `customer_phone`.
- Looks up an existing user by phone number with role Customer.
- If not found, creates a new User and Customer record, with the PIN hashed.
- If found, updates the existing customer's details (name, phone, address, mohalla, birthday). If a Customer document is missing for some reason, it is created at this point.
- Generates a `customer_token`, sets it as a cookie, and returns it in the response along with customer details.

---

## 5. Utility Endpoint

### POST `/verify-access-token`
Verifies an MSG91 access token directly, without going through the full OTP flow. Used less frequently, mainly for direct token checks.

**Input:** `accessToken`

---

## Notes for Frontend Integration

- The Seller login flow always requires two calls before `/login`: first `/login/init` (or `/send-otp` directly), then `/verify-otp` to get the `verificationToken`.
- The `verificationToken` is short-lived (15 minutes), so the PIN screen should not be shown too long after OTP verification.
- Customer registration always requires a valid `verificationToken` tied to the customer's own phone number, even though the request itself is authenticated with a Seller or Admin session token.
