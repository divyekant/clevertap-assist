# CleverTap Server API — Python Integration Reference

> Self-contained integration guide for the CleverTap REST API using Python.
> Covers authentication, uploading events, uploading user profiles, and error handling.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Python** | 3.7+ |
| **requests** | `pip install requests` |
| **CleverTap account** | You need your **Account ID** and **Passcode** from the CleverTap dashboard (Settings > Project > Account Credentials) |

> **Passcode != Token.** Server-side API authentication uses the **Passcode**, not the API Token used in client SDKs. These are different credentials. Using the wrong one will return a 401 error.

---

## Authentication

Every request must include these headers:

```python
headers = {
    "X-CleverTap-Account-Id": "YOUR_ACCOUNT_ID",
    "X-CleverTap-Passcode": "YOUR_PASSCODE",
    "Content-Type": "application/json; charset=utf-8",
}
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

```python
import time
import requests


class CleverTap:
    def __init__(self, account_id: str, passcode: str, region: str = ""):
        subdomain = f"{region}." if region else ""
        self.base_url = f"https://{subdomain}api.clevertap.com"
        self.headers = {
            "X-CleverTap-Account-Id": account_id,
            "X-CleverTap-Passcode": passcode,
            "Content-Type": "application/json; charset=utf-8",
        }

    def upload(self, records: list[dict]) -> dict:
        """Upload a list of event or profile records."""
        response = requests.post(
            f"{self.base_url}/1/upload",
            headers=self.headers,
            json={"d": records},
        )
        response.raise_for_status()
        data = response.json()

        if data.get("unprocessed"):
            print(f"Warning: {len(data['unprocessed'])} unprocessed records")
            for rec in data["unprocessed"]:
                print(f"  - {rec.get('error', 'Unknown error')}")

        return data

    def upload_event(
        self,
        identity: str,
        event_name: str,
        event_data: dict | None = None,
        timestamp: float | None = None,
    ) -> dict:
        """Upload a single event."""
        record = {
            "type": "event",
            "identity": identity,
            "evtName": event_name,
            "evtData": event_data or {},
        }
        if timestamp is not None:
            record["ts"] = int(timestamp)
        return self.upload([record])

    def upload_events(self, events: list[dict]) -> dict:
        """Upload multiple events in a single request."""
        records = []
        for e in events:
            record = {
                "type": "event",
                "identity": e["identity"],
                "evtName": e["event_name"],
                "evtData": e.get("event_data", {}),
            }
            if "timestamp" in e:
                record["ts"] = int(e["timestamp"])
            records.append(record)
        return self.upload(records)

    def upload_profile(self, identity: str, profile_data: dict) -> dict:
        """Upload a single user profile."""
        record = {
            "type": "profile",
            "identity": identity,
            "profileData": profile_data,
        }
        return self.upload([record])

    def upload_profiles(self, profiles: list[dict]) -> dict:
        """Upload multiple user profiles in a single request."""
        records = [
            {
                "type": "profile",
                "identity": p["identity"],
                "profileData": p["profile_data"],
            }
            for p in profiles
        ]
        return self.upload(records)
```

---

## Upload Events

### Single event

```python
ct = CleverTap(
    account_id="YOUR_ACCOUNT_ID",
    passcode="YOUR_PASSCODE",
    region="YOUR_REGION",  # e.g. "in1", "sg1", or "" for default US
)

result = ct.upload_event(
    identity="user-12345",
    event_name="Product Viewed",
    event_data={
        "Product Name": "Wireless Headphones",
        "Category": "Electronics",
        "Price": 79.99,
    },
)

print(result)
# {"status": "success", "processed": 1, "unprocessed": []}
```

### Batch events (recommended)

Batch uploads reduce API calls. The API accepts up to 1,000 records per request.

```python
result = ct.upload_events([
    {
        "identity": "user-12345",
        "event_name": "Product Viewed",
        "event_data": {"Product Name": "Wireless Headphones", "Price": 79.99},
    },
    {
        "identity": "user-12345",
        "event_name": "Added to Cart",
        "event_data": {"Product Name": "Wireless Headphones", "Quantity": 1},
    },
    {
        "identity": "user-67890",
        "event_name": "Checkout Completed",
        "event_data": {"Amount": 149.99, "Currency": "USD"},
        "timestamp": int(time.time()),  # optional: defaults to current time on server
    },
])

print(f"Processed: {result['processed']}")
```

### Charged event (transactions)

```python
ct.upload_event(
    identity="user-12345",
    event_name="Charged",
    event_data={
        "Amount": 119.98,
        "Currency": "USD",
        "Charged ID": "order-12345",
        "Items": [
            {"Product Name": "Wireless Headphones", "Price": 79.99, "Quantity": 1},
            {"Product Name": "USB-C Cable", "Price": 39.99, "Quantity": 1},
        ],
    },
)
```

### Raw API call (without helper)

```python
import requests

response = requests.post(
    "https://in1.api.clevertap.com/1/upload",
    headers={
        "X-CleverTap-Account-Id": "YOUR_ACCOUNT_ID",
        "X-CleverTap-Passcode": "YOUR_PASSCODE",
        "Content-Type": "application/json; charset=utf-8",
    },
    json={
        "d": [
            {
                "type": "event",
                "identity": "user-12345",
                "evtName": "Product Viewed",
                "evtData": {
                    "Product Name": "Wireless Headphones",
                    "Category": "Electronics",
                    "Price": 79.99,
                },
            }
        ]
    },
)

data = response.json()
print(data)
```

---

## Upload User Profiles

### Single profile

```python
result = ct.upload_profile(
    identity="user-12345",
    profile_data={
        "Name": "Jane Doe",
        "Email": "jane@example.com",
        "Phone": "+14155551234",
        "Gender": "F",
        "Plan": "Premium",
    },
)
```

### Batch profiles

```python
result = ct.upload_profiles([
    {
        "identity": "user-12345",
        "profile_data": {
            "Name": "Jane Doe",
            "Email": "jane@example.com",
            "Plan": "Premium",
        },
    },
    {
        "identity": "user-67890",
        "profile_data": {
            "Name": "John Smith",
            "Email": "john@example.com",
            "Plan": "Free",
        },
    },
])
```

### Profile operations

```python
# Increment a numeric property
ct.upload_profile(
    identity="user-12345",
    profile_data={"Loyalty Points": {"$incr": 50}},
)

# Add to a multi-value property
ct.upload_profile(
    identity="user-12345",
    profile_data={"Interests": {"$add": ["Electronics", "Music"]}},
)

# Remove from a multi-value property
ct.upload_profile(
    identity="user-12345",
    profile_data={"Interests": {"$remove": ["Music"]}},
)

# Manage messaging subscriptions
ct.upload_profile(
    identity="user-12345",
    profile_data={
        "MSG-email": True,
        "MSG-sms": False,
        "MSG-whatsapp": True,
    },
)
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

### Robust error handling with retry

```python
import time
import requests


def upload_with_retry(ct: CleverTap, records: list[dict], max_retries: int = 3) -> dict:
    """Upload records with exponential backoff on rate limit errors."""
    for attempt in range(1, max_retries + 1):
        try:
            result = ct.upload(records)

            if result.get("unprocessed"):
                print(f"Warning: {len(result['unprocessed'])} records failed")
                for rec in result["unprocessed"]:
                    print(f"  Code {rec.get('code')}: {rec.get('error')}")

            return result

        except requests.exceptions.HTTPError as e:
            if e.response is not None and e.response.status_code == 429:
                if attempt < max_retries:
                    delay = 2 ** attempt
                    print(f"Rate limited. Retrying in {delay}s...")
                    time.sleep(delay)
                    continue
            raise

        except requests.exceptions.ConnectionError:
            if attempt < max_retries:
                delay = 2 ** attempt
                print(f"Connection error. Retrying in {delay}s...")
                time.sleep(delay)
                continue
            raise
```

### Dry run (validate without uploading)

Append `?dryRun=1` to the URL to validate payloads without persisting data. Useful for testing during development.

```python
response = requests.post(
    f"{ct.base_url}/1/upload?dryRun=1",
    headers=ct.headers,
    json={"d": records},
)
```

---

## Common Pitfalls

### 1. Passcode != Token

Server API authentication requires the **Passcode**, not the API Token. The Token is used by client-side SDKs. Using the wrong credential returns a 401 error. Find the Passcode under Settings > Project in the CleverTap dashboard.

### 2. Rate limits

The API allows a maximum of 15 concurrent requests per account. Exceeding this returns HTTP 429. Use batch uploads (up to 1,000 records per request) to stay within limits.

### 3. Batch uploads recommended

Prefer sending multiple records in a single request over individual API calls:

```python
# BAD: one API call per event
for event in events:
    ct.upload_event(**event)  # N API calls

# GOOD: one API call for all events
ct.upload_events(events)  # 1 API call (max 1,000 records)
```

For large datasets, chunk into batches of 1,000:

```python
def chunked(lst: list, size: int):
    for i in range(0, len(lst), size):
        yield lst[i : i + size]

for batch in chunked(events, 1000):
    ct.upload_events(batch)
```

### 4. Timestamp format

The `ts` field expects a UNIX epoch timestamp **in seconds** (integer). Python's `time.time()` returns seconds as a float, so convert with `int()`:

```python
import time

# CORRECT: integer seconds
{"ts": int(time.time())}

# ALSO CORRECT: from a datetime object
from datetime import datetime
{"ts": int(datetime(2024, 1, 15, 12, 0, 0).timestamp())}
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

```python
ct = CleverTap(
    account_id="YOUR_ACCOUNT_ID",
    passcode="YOUR_PASSCODE",
    region="YOUR_REGION",
)

# 1. Upload an event
ct.upload_event(
    identity="user-12345",
    event_name="Product Viewed",
    event_data={"Product Name": "Headphones", "Price": 79.99},
)

# 2. Upload a user profile
ct.upload_profile(
    identity="user-12345",
    profile_data={"Name": "Jane Doe", "Email": "jane@example.com", "Plan": "Premium"},
)

# 3. Batch upload events
ct.upload_events([
    {"identity": "user-12345", "event_name": "Page View", "event_data": {"Page": "/home"}},
    {"identity": "user-67890", "event_name": "Page View", "event_data": {"Page": "/pricing"}},
])

# 4. Validate without persisting (dry run)
# Append ?dryRun=1 to the upload URL
```
