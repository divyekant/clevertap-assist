# CleverTap Server API — Node.js Integration Reference

> Self-contained integration guide for the CleverTap REST API using Node.js.
> Covers authentication, uploading events, uploading user profiles, and error handling.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Node.js** | v18+ (for native `fetch`) or v14+ with `node-fetch` / `axios` |
| **CleverTap account** | You need your **Account ID** and **Passcode** from the CleverTap dashboard (Settings > Project > Account Credentials) |

> **Passcode != Token.** Server-side API authentication uses the **Passcode**, not the API Token used in client SDKs. These are different credentials. Using the wrong one will return a 401 error.

---

## Authentication

Every request must include these headers:

```javascript
const headers = {
  "X-CleverTap-Account-Id": "YOUR_ACCOUNT_ID",
  "X-CleverTap-Passcode": "YOUR_PASSCODE",
  "Content-Type": "application/json; charset=utf-8"
};
```

---

## Base URL

The base URL depends on your CleverTap account region. If your dashboard URL contains `in1.dashboard.clevertap.com`, use `in1.api.clevertap.com`.

| Region | Base URL |
|---|---|
| Default (US) | `https://api.clevertap.com` |
| India | `https://in1.api.clevertap.com` |
| Singapore | `https://sg1.api.clevertap.com` |
| Asia Pacific (Sydney) | `https://aps3.api.clevertap.com` |
| Middle East | `https://mec1.api.clevertap.com` |

---

## Helper Class

A minimal wrapper around the CleverTap REST API:

```javascript
class CleverTap {
  constructor({ accountId, passcode, region = "" }) {
    const subdomain = region ? `${region}.` : "";
    this.baseUrl = `https://${subdomain}api.clevertap.com`;
    this.headers = {
      "X-CleverTap-Account-Id": accountId,
      "X-CleverTap-Passcode": passcode,
      "Content-Type": "application/json; charset=utf-8"
    };
  }

  async upload(records) {
    const res = await fetch(`${this.baseUrl}/1/upload`, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify({ d: records })
    });

    if (!res.ok) {
      const text = await res.text();
      throw new Error(`CleverTap API error ${res.status}: ${text}`);
    }

    const data = await res.json();

    if (data.unprocessed && data.unprocessed.length > 0) {
      console.warn("Unprocessed records:", data.unprocessed);
    }

    return data;
  }

  async uploadEvent({ identity, eventName, eventData = {}, timestamp }) {
    const record = {
      type: "event",
      identity,
      evtName: eventName,
      evtData: eventData
    };
    if (timestamp) {
      record.ts = Math.floor(timestamp / 1000); // Convert ms to seconds
    }
    return this.upload([record]);
  }

  async uploadEvents(events) {
    const records = events.map((e) => ({
      type: "event",
      identity: e.identity,
      evtName: e.eventName,
      evtData: e.eventData || {},
      ...(e.timestamp ? { ts: Math.floor(e.timestamp / 1000) } : {})
    }));
    return this.upload(records);
  }

  async uploadProfile({ identity, profileData }) {
    const record = {
      type: "profile",
      identity,
      profileData
    };
    return this.upload([record]);
  }

  async uploadProfiles(profiles) {
    const records = profiles.map((p) => ({
      type: "profile",
      identity: p.identity,
      profileData: p.profileData
    }));
    return this.upload(records);
  }
}
```

---

## Upload Events

### Single event

```javascript
const ct = new CleverTap({
  accountId: "YOUR_ACCOUNT_ID",
  passcode: "YOUR_PASSCODE",
  region: "YOUR_REGION" // e.g. "in1", "sg1", or "" for default US
});

const result = await ct.uploadEvent({
  identity: "user-12345",
  eventName: "Product Viewed",
  eventData: {
    "Product Name": "Wireless Headphones",
    "Category": "Electronics",
    "Price": 79.99
  }
});

console.log(result);
// { status: "success", processed: 1, unprocessed: [] }
```

### Batch events (recommended)

Batch uploads reduce API calls. The API accepts up to 1,000 records per request.

```javascript
const result = await ct.uploadEvents([
  {
    identity: "user-12345",
    eventName: "Product Viewed",
    eventData: { "Product Name": "Wireless Headphones", "Price": 79.99 }
  },
  {
    identity: "user-12345",
    eventName: "Added to Cart",
    eventData: { "Product Name": "Wireless Headphones", "Quantity": 1 }
  },
  {
    identity: "user-67890",
    eventName: "Checkout Completed",
    eventData: { "Amount": 149.99, "Currency": "USD" },
    timestamp: Date.now() // optional: defaults to current time on server
  }
]);

console.log(`Processed: ${result.processed}`);
```

### Charged event (transactions)

```javascript
await ct.uploadEvent({
  identity: "user-12345",
  eventName: "Charged",
  eventData: {
    "Amount": 119.98,
    "Currency": "USD",
    "Charged ID": "order-12345",
    "Items": [
      { "Product Name": "Wireless Headphones", "Price": 79.99, "Quantity": 1 },
      { "Product Name": "USB-C Cable", "Price": 39.99, "Quantity": 1 }
    ]
  }
});
```

### Raw API call (without helper)

```javascript
const response = await fetch("https://in1.api.clevertap.com/1/upload", {
  method: "POST",
  headers: {
    "X-CleverTap-Account-Id": "YOUR_ACCOUNT_ID",
    "X-CleverTap-Passcode": "YOUR_PASSCODE",
    "Content-Type": "application/json; charset=utf-8"
  },
  body: JSON.stringify({
    d: [
      {
        type: "event",
        identity: "user-12345",
        evtName: "Product Viewed",
        evtData: {
          "Product Name": "Wireless Headphones",
          "Category": "Electronics",
          "Price": 79.99
        }
      }
    ]
  })
});

const data = await response.json();
console.log(data);
```

---

## Upload User Profiles

### Single profile

```javascript
const result = await ct.uploadProfile({
  identity: "user-12345",
  profileData: {
    "Name": "Jane Doe",
    "Email": "jane@example.com",
    "Phone": "+14155551234",
    "Gender": "F",
    "Plan": "Premium"
  }
});
```

### Batch profiles

```javascript
const result = await ct.uploadProfiles([
  {
    identity: "user-12345",
    profileData: {
      "Name": "Jane Doe",
      "Email": "jane@example.com",
      "Plan": "Premium"
    }
  },
  {
    identity: "user-67890",
    profileData: {
      "Name": "John Smith",
      "Email": "john@example.com",
      "Plan": "Free"
    }
  }
]);
```

### Profile operations

```javascript
// Increment a numeric property
await ct.uploadProfile({
  identity: "user-12345",
  profileData: {
    "Loyalty Points": { "$incr": 50 }
  }
});

// Add to a multi-value property
await ct.uploadProfile({
  identity: "user-12345",
  profileData: {
    "Interests": { "$add": ["Electronics", "Music"] }
  }
});

// Remove from a multi-value property
await ct.uploadProfile({
  identity: "user-12345",
  profileData: {
    "Interests": { "$remove": ["Music"] }
  }
});

// Manage messaging subscriptions
await ct.uploadProfile({
  identity: "user-12345",
  profileData: {
    "MSG-email": true,
    "MSG-sms": false,
    "MSG-whatsapp": true
  }
});
```

---

## User Identity

The `identity` field in the upload payload links events and profiles to a user. It can be:

- An email address: `"identity": "jane@example.com"`
- A phone number: `"identity": "+14155551234"`
- A custom ID: `"identity": "user-12345"`

Alternative identity fields (use exactly one per record):

| Field | Description |
|---|---|
| `identity` | Email, phone, or custom identifier (most common) |
| `FBID` | Facebook ID |
| `GPID` | Google Play ID |
| `objectId` | CleverTap internal Global Object ID |

If an identity does not exist, the API automatically creates a new user profile.

---

## Error Handling

### HTTP status codes

| Status | Meaning |
|---|---|
| `200` | Request processed (check response body for per-record status) |
| `400` | Bad request (malformed JSON, missing required fields) |
| `401` | Authentication failed (invalid Account ID or Passcode) |
| `429` | Rate limit exceeded (max 15 concurrent requests per account) |
| `500` | Server error |

### Response format

```json
{
  "status": "success",
  "processed": 2,
  "unprocessed": []
}
```

Partial failure:

```json
{
  "status": "partial",
  "processed": 1,
  "unprocessed": [
    {
      "status": "fail",
      "code": 514,
      "error": "Event name is mandatory",
      "record": { ... }
    }
  ]
}
```

### Robust error handling

```javascript
async function uploadWithRetry(ct, records, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const result = await ct.upload(records);

      if (result.unprocessed?.length > 0) {
        console.error("Some records failed:", result.unprocessed);
      }

      return result;
    } catch (error) {
      if (error.message.includes("429") && attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        console.warn(`Rate limited. Retrying in ${delay}ms...`);
        await new Promise((resolve) => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
}
```

### Dry run (validate without uploading)

Append `?dryRun=1` to the URL to validate payloads without persisting data. Useful for testing during development.

```javascript
const res = await fetch(`${baseUrl}/1/upload?dryRun=1`, {
  method: "POST",
  headers,
  body: JSON.stringify({ d: records })
});
```

---

## Common Pitfalls

### 1. Passcode != Token

Server API authentication requires the **Passcode**, not the API Token. The Token is used by client-side SDKs. Using the wrong credential returns a 401 error. Find the Passcode under Settings > Project in the CleverTap dashboard.

### 2. Rate limits

The API allows a maximum of 15 concurrent requests per account. Exceeding this returns HTTP 429. Use batch uploads (up to 1,000 records per request) to stay within limits.

### 3. Batch uploads recommended

Prefer sending multiple records in a single request over individual API calls:

```javascript
// BAD: one API call per event
for (const event of events) {
  await ct.uploadEvent(event); // N API calls
}

// GOOD: one API call for all events
await ct.uploadEvents(events); // 1 API call (max 1,000 records)
```

For large datasets, chunk into batches of 1,000:

```javascript
function chunk(arr, size) {
  const chunks = [];
  for (let i = 0; i < arr.length; i += size) {
    chunks.push(arr.slice(i, i + size));
  }
  return chunks;
}

for (const batch of chunk(events, 1000)) {
  await ct.uploadEvents(batch);
}
```

### 4. Timestamp format

The `ts` field expects a UNIX epoch timestamp **in seconds**, not milliseconds. JavaScript's `Date.now()` returns milliseconds, so divide by 1000:

```javascript
// WRONG: milliseconds
{ ts: Date.now() }

// CORRECT: seconds
{ ts: Math.floor(Date.now() / 1000) }
```

If `ts` is omitted, the server uses the current time.

### 5. Region must match account

Using the wrong region base URL will return authentication errors even with valid credentials. Check your CleverTap dashboard URL to determine your region.

### 6. Data constraints

- Event names: max 512 characters
- Property keys: max 120 characters
- Property values: max 512 bytes
- Max 256 custom properties per event type
- Max 512 unique event types per account
- Max 1,000 records per API request

---

## Quick Reference

```javascript
const ct = new CleverTap({
  accountId: "YOUR_ACCOUNT_ID",
  passcode: "YOUR_PASSCODE",
  region: "YOUR_REGION"
});

// 1. Upload an event
await ct.uploadEvent({
  identity: "user-12345",
  eventName: "Product Viewed",
  eventData: { "Product Name": "Headphones", "Price": 79.99 }
});

// 2. Upload a user profile
await ct.uploadProfile({
  identity: "user-12345",
  profileData: { "Name": "Jane Doe", "Email": "jane@example.com", "Plan": "Premium" }
});

// 3. Batch upload events
await ct.uploadEvents([
  { identity: "user-12345", eventName: "Page View", eventData: { "Page": "/home" } },
  { identity: "user-67890", eventName: "Page View", eventData: { "Page": "/pricing" } }
]);

// 4. Validate without persisting (dry run)
// Append ?dryRun=1 to the upload URL
```
