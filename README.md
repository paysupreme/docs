# Merkava Partner UAT Guide

## Base Configuration

**Base URL**: `https://merkava-uat.up.railway.app`

**Demo Partner Credentials**:
```json
{
  "apiKey": "api_key_B9DVzVM4RsNPoum09KI4swSB",
  "apiSecret": "api_secret_5OOfn8OVQbULxW7ypOV6geYoIceLRG7I2zIJjoiIdFwSfXYWMn1G2",
  "signingSecret": "sign_aaf92ef010cedb1b3edb1d69eab190b17031e56659f22ec2507b06f1970..."
}
```

---

## API Security: Request Signing

All JWT-protected endpoints require the following headers:

- `x-signature`: HMAC SHA256 signature (see below)
- `x-timestamp`: Current UNIX timestamp (in seconds)
- `x-partner-id`: Your partner UUID (from profile or login response)

**How to generate the signature:**

1. Concatenate the raw request body (as a string, or `''` for GET/empty body) and the `x-timestamp` value.
2. Compute the HMAC SHA256 digest using your `signingSecret` as the key.
3. Send the result as a lowercase hex string in the `x-signature` header.

**Example (JavaScript, using CryptoJS):**

```js
const CryptoJS = require('crypto-js');

const timestamp = Math.floor(Date.now() / 1000);
const requestBody = JSON.stringify({ /* your request body here, or '' for GET */ }) || '';
const signSecret = 'sign_aaf92ef010cedb1b3edb1d69eab190b17031e56659f22ec2507b06f1970...';

const signatureData = requestBody + timestamp;
const signature = CryptoJS.HmacSHA256(signatureData, signSecret).toString(CryptoJS.enc.Hex);

// Example headers to send:
const headers = {
  'Authorization': `Bearer ${accessToken}`,
  'x-signature': signature,
  'x-timestamp': timestamp,
  'x-partner-id': 'your-partner-uuid',
  'Content-Type': 'application/json',
};
```

**Notes:**
- The `x-timestamp` must be within ±5 minutes of server time.
- The signature is required for all JWT-protected endpoints (not for `/auth/login`).
- If the body is empty (e.g., GET), use an empty string for signing.
- If you use another language, use any HMAC SHA256 library.

---

## API Endpoints

### 1. Authentication

#### Login

**Endpoint**: `POST /api/v1/auth/login`

**Description**: Generates a bearer token for API authentication

**Request Headers**:
```
Content-Type: application/json
```

**Request Body**:
```json
{
  "apiKey": "api_key_B9DVzVM4RsNPoum09KI4swSB",
  "apiSecret": "api_secret_5OOfn8OVQbULxW7ypOV6geYoIceLRG7I2zIJjoiIdFwSfXYWMn1G2"
}
```

**Response (Success - 200)**:
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "partner": {
      "id": "89bfa6c6-45fc-45df-b70e-aa0be5714380",
      "name": "Demo Partner",
      "email": "demo@partner.com",
      "status": "ACTIVE"
    }
  },
  "message": "Login successful"
}
```

**Response (Error - 401)**:
```json
{
  "success": false,
  "error": "Invalid API credentials",
  "statusCode": 401
}
```

---

### 2. Profile

#### Get Partner Profile & Balance

**Endpoint**: `GET /api/v1/partners/me/profile`

**Description**: Retrieves partner profile information and current balance

**Request Headers**:
```
Authorization: Bearer {accessToken}
x-signature: {signature}
x-timestamp: {timestamp}
x-partner-id: {partnerId}
Content-Type: application/json
```

**Request Body**: None (GET request)

**Response (Success - 200)**:
```json
{
  "success": true,
  "data": {
    "id": "89bfa6c6-45fc-45df-b70e-aa0be5714380",
    "name": "Demo Partner",
    "email": "demo@partner.com",
    "phone": "+919876543210",
    "status": "ACTIVE",
    "balance": 50000.00,
    "reservedBalance": 5000.00,
    "availableBalance": 45000.00,
    "currency": "INR",
    "isActive": true,
    "rateConfig": {
      "payinRate": 2.5,
      "payoutRate": 3.0,
      "currency": "INR"
    },
    "webhookConfig": {
      "url": "https://your-webhook-url.com/webhook",
      "events": ["PAYMENT_SUCCESS", "PAYMENT_FAILED"]
    },
    "createdAt": "2025-01-15T10:30:00.000Z",
    "updatedAt": "2025-06-08T12:45:00.000Z"
  },
  "message": "Partner profile retrieved successfully"
}
```

**Response (Error - 401)**:
```json
{
  "success": false,
  "error": "Invalid or expired token",
  "statusCode": 401
}
```

---

### 3. Pay-in

#### Generate Payment Link

**Endpoint**: `POST /api/v1/payments/payment-links`

**Description**: Creates a payment link for collecting payments from customers

**Request Headers**:
```
Authorization: Bearer {accessToken}
x-signature: {signature}
x-timestamp: {timestamp}
x-partner-id: {partnerId}
Content-Type: application/json
```

**Request Body**:
```json
{
  "amount": 10,
  "purpose": "Product purchase INR10",
  "customer_name": "Doe Snow",
  "customer_phone": "9876545404",
  "customer_email": "don@example.com",
  "return_url": "{{your-return-Url}}/",
  "notify_url": "{{your-webhook-urk}}/webhooks"
}
```

**Response (Success - 201)**:
```json
{
    "data": {
        "success": true,
        "data": {
            "transaction_id": "2f3515cc-87da-426b-a859-9ee75eea21ee",
            "link_id": "PLNK_387e3efc_d125_43b1_a166_3194e1d7f007",
            "cf_link_id": "6501174",
            "link_url": "https://payments-test.cashfree.com/links/j8n4matu1tng",
            "link_qrcode": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAUAAAAFAAQMAAAD3XjfpAAAABlBMVEX///8AAABVwtN+AAAC10lEQVR4nOxasZHkMAyDx4FCleBS1NlZ25lKcQkKHWiMH5Davb37nY8+kszozoYTDgGC5OKOO+644//GRsVyfv9TsFbsC1l2/duGBGYAaTkjuZ4beSLpdb4AfAkwO5A80qL3/Yuys24ZgeVrJfPAQPuzBRaEmjLigYUVN/AFjAUtHOkHuUYGetEg6VFaxBe9Vxl9JNdkwK64sezruZUr1PRo8didZR+keQigxzMrRUqCJ2c+xHxAU1zLWGBRGV1r3fKFqBxub3kcCLhlvTclQTj0JD3IQ5JCfTE9UHm8Apl5bjTpXTvdflTQUEDsPMWZmvjsrpeERuRZpC2zA9dT7aX6Fwp1X3bpZZ4buGX1ITPsohHPSNeeoHmmYm8jAoFnMrIkhaxYWqTNKw/pMCYHKlHKYzZOAcm8GXoeMSZwl9DauNqwFbKmB+vWOeOGdWogXGixrzwSLaHxgJPrbZ4ZDujDmx4XWTKVkXmzLG3B7EAjl8qI5tNTXquk1xrOexseCChvZmqpFB3WZq3Vqp6Wd87MCURPVPSOQlvzfNqRDgXc2IIMu7qrCylsCbj06nllZ1KgqqfQ9hynr4gfreuw0W1QIIIXjhnyS1OcCWmoeC7BJgaa+bA82o5UX9GGt1BNZNBGBG79NhIL3Hu5fupJL6OpgVD6pLhuPpTH/GO0byMCPT228nPymNMCnDPx7VwwKVDTiXkz/wKJml0/teEZgdkM+4nE1fdi3n1pu+RfDWkgoKy5tZcX3+TNzK3xBmYVjZ1blTWZtNdezAb+IYEei/qMzIctAdFPbMoT2tRA3/X4Zc1cijt3vs7SGBJo12ZzYuozVyAfPruyQKN9mxzoP2DpO1L7eZOKxlbpGueGBqrP+DhnS8BdbVaO9NOPleYD0mvFdz0tMtuhmr/WYQMBO2ecJnYb8YNZdM5MDuyKW/HdZ8yw+1X+b2keAnjHHXfc8e/4EwAA//8TOi7RLtonIQAAAABJRU5ErkJggg==",
            "link_status": "ACTIVE",
            "link_amount": 75,
            "link_amount_paid": 0,
            "link_currency": "INR",
            "link_purpose": "TRY FINAL Redirect: Product purchase 69INR",
            "link_expiry_time": "2025-06-14T02:05:06.000Z",
            "customer_details": {
                "customer_name": "Failed2 Snow",
                "customer_phone": "9999999998",
                "customer_email": "don@example.com"
            }
        },
        "message": "Payment link created successfully",
        "metadata": {
            "timestamp": "2025-06-14T01:50:06.841Z",
            "partner_id": "c6fe6d63-8a7a-4b15-a44e-6cc1fa240216",
            "link_id": "PLNK_387e3efc_d125_43b1_a166_3194e1d7f007"
        }
    },
    "timestamp": "2025-06-14T01:50:06.842Z",
    "path": "/api/v1/payments/payment-links"
}
```

---

#### Check Transaction Status

**Endpoint**: `GET /api/v1/payments/txn/:transactionId/status`

**Description**: Retrieves the status and details of a specific transaction.

**Query Parameter:**
- `transactionId` (in URL path): The transaction ID to check status for.

**Request Headers**:
```
Authorization: Bearer {accessToken}
x-signature: {signature}
x-timestamp: {timestamp}
x-partner-id: {partnerId}
Content-Type: application/json
```

**Response (Success - 200)**:
```json
{
    "data": {
        "success": true,
        "data": {
            "transaction_id": "2f3515cc-87da-426b-a859-9ee75eea21ee",
            "status": "SUCCESS",
            "amount": 75,
            "currency": "INR",
            "metadata": {
                "link_id": "PLNK_387e3efc_d125_43b1_a166_3194e1d7f007",
                "customer_name": "Failed2 Snow",
                "customer_email": "don@example.com",
                "customer_phone": "9999999998"
            }
        },
        "message": "Transaction status retrieved successfully",
        "metadata": {
            "timestamp": "2025-06-14T03:17:18.494Z",
            "partner_id": "c6fe6d63-8a7a-4b15-a44e-6cc1fa240216",
            "transaction_id": "2f3515cc-87da-426b-a859-9ee75eea21ee"
        }
    },
    "timestamp": "2025-06-14T03:17:18.494Z",
    "path": "/api/v1/payments/txn/2f3515cc-87da-426b-a859-9ee75eea21ee/status"
}
```

---

#### Get Pay-in Stats

**Endpoint**: `GET /api/v1/payments/stats`

**Description**: Fetches summary statistics for the partner's pay-in and payment link transactions.

**Request Headers**:
```
Authorization: Bearer {accessToken}
x-signature: {signature}
x-timestamp: {timestamp}
x-partner-id: {partnerId}
Content-Type: application/json
```

**Response (Success - 200)**:
```json
{
  "total_count": 120,
  "total_completed": 80,
  "total_pending": 10,
  "total_expired": 5,
  "total_failed": 25
}
```

---

## Testing Flow

### Step 1: Authenticate
1. Use the login endpoint to get an access token
2. Save the `accessToken` from the response

### Step 2: Check Profile & Balance
1. Use the bearer token to call the profile endpoint
2. Verify your balance is sufficient for transactions

### Step 3: Create Payment Link
1. Use the bearer token to create a payment link
2. Use the returned `link_url` to test the payment flow
3. Monitor webhook events (if configured)

---

## Important Notes

### Authentication
- All endpoints except `/auth/login` require the `Authorization: Bearer {token}` header
- All JWT-protected endpoints require `x-signature`, `x-timestamp`, and `x-partner-id` headers
- Tokens expire after 1 hour (3600 seconds)
- Re-authenticate when you receive 401 errors

### Rate Limits
- API calls are rate-limited per partner
- Implement proper retry logic with exponential backoff

### Webhook Events
- Configure webhook URLs in your partner profile
- Handle webhook events for real-time payment status updates
- Common events: `PAYMENT_SUCCESS`, `PAYMENT_FAILED`, `PAYMENT_LINK_EXPIRED`

### Testing Best Practices
1. Always use unique `partner_reference` values
2. Test both success and failure scenarios
3. Verify webhook delivery and processing
4. Test payment link expiry behavior
5. Validate all required fields before making requests

---

## Support

For technical support or questions about the UAT environment, please contact the development team with:
- API endpoint used
- Request/response details
- Error messages (if any)
- Timestamp of the issue 
