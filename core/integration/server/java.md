# CleverTap Server API — Java Integration Reference

> Self-contained integration guide for the CleverTap REST API using Java.
> Covers authentication, uploading events, uploading user profiles, and error handling.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Java** | 11+ (for `java.net.http.HttpClient`) |
| **JSON library** | `org.json:json` or `com.google.code.gson:gson` (examples use `org.json`) |
| **CleverTap account** | You need your **Account ID** and **Passcode** from the CleverTap dashboard (Settings > Project > Account Credentials) |

> **Passcode != Token.** Server-side API authentication uses the **Passcode**, not the API Token used in client SDKs. These are different credentials. Using the wrong one will return a 401 error.

### Maven dependency (org.json)

```xml
<dependency>
    <groupId>org.json</groupId>
    <artifactId>json</artifactId>
    <version>20231013</version>
</dependency>
```

### Gradle

```groovy
implementation 'org.json:json:20231013'
```

---

## Authentication

Every request must include these headers:

```java
String accountId = "YOUR_ACCOUNT_ID";
String passcode = "YOUR_PASSCODE";

HttpRequest.Builder requestBuilder = HttpRequest.newBuilder()
    .header("X-CleverTap-Account-Id", accountId)
    .header("X-CleverTap-Passcode", passcode)
    .header("Content-Type", "application/json; charset=utf-8");
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

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import org.json.JSONArray;
import org.json.JSONObject;

public class CleverTap {

    private final String baseUrl;
    private final String accountId;
    private final String passcode;
    private final HttpClient httpClient;

    public CleverTap(String accountId, String passcode, String region) {
        String subdomain = (region != null && !region.isEmpty()) ? region + "." : "";
        this.baseUrl = "https://" + subdomain + "api.clevertap.com";
        this.accountId = accountId;
        this.passcode = passcode;
        this.httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();
    }

    public JSONObject upload(JSONArray records) throws Exception {
        JSONObject payload = new JSONObject().put("d", records);

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(baseUrl + "/1/upload"))
                .header("X-CleverTap-Account-Id", accountId)
                .header("X-CleverTap-Passcode", passcode)
                .header("Content-Type", "application/json; charset=utf-8")
                .POST(HttpRequest.BodyPublishers.ofString(payload.toString()))
                .timeout(Duration.ofSeconds(30))
                .build();

        HttpResponse<String> response = httpClient.send(
                request, HttpResponse.BodyHandlers.ofString());

        if (response.statusCode() != 200) {
            throw new RuntimeException(
                    "CleverTap API error " + response.statusCode() + ": " + response.body());
        }

        JSONObject result = new JSONObject(response.body());
        JSONArray unprocessed = result.optJSONArray("unprocessed");
        if (unprocessed != null && unprocessed.length() > 0) {
            System.err.println("Warning: " + unprocessed.length() + " unprocessed records");
        }

        return result;
    }

    public JSONObject uploadEvent(String identity, String eventName,
                                   JSONObject eventData) throws Exception {
        return uploadEvent(identity, eventName, eventData, -1);
    }

    public JSONObject uploadEvent(String identity, String eventName,
                                   JSONObject eventData, long timestampSeconds) throws Exception {
        JSONObject record = new JSONObject()
                .put("type", "event")
                .put("identity", identity)
                .put("evtName", eventName)
                .put("evtData", eventData != null ? eventData : new JSONObject());

        if (timestampSeconds > 0) {
            record.put("ts", timestampSeconds);
        }

        return upload(new JSONArray().put(record));
    }

    public JSONObject uploadEvents(List<JSONObject> events) throws Exception {
        JSONArray records = new JSONArray();
        for (JSONObject event : events) {
            JSONObject record = new JSONObject()
                    .put("type", "event")
                    .put("identity", event.getString("identity"))
                    .put("evtName", event.getString("eventName"))
                    .put("evtData", event.optJSONObject("eventData") != null
                            ? event.getJSONObject("eventData") : new JSONObject());

            if (event.has("timestamp")) {
                record.put("ts", event.getLong("timestamp"));
            }
            records.put(record);
        }
        return upload(records);
    }

    public JSONObject uploadProfile(String identity, JSONObject profileData) throws Exception {
        JSONObject record = new JSONObject()
                .put("type", "profile")
                .put("identity", identity)
                .put("profileData", profileData);

        return upload(new JSONArray().put(record));
    }

    public JSONObject uploadProfiles(List<JSONObject> profiles) throws Exception {
        JSONArray records = new JSONArray();
        for (JSONObject profile : profiles) {
            JSONObject record = new JSONObject()
                    .put("type", "profile")
                    .put("identity", profile.getString("identity"))
                    .put("profileData", profile.getJSONObject("profileData"));
            records.put(record);
        }
        return upload(records);
    }
}
```

---

## Upload Events

### Single event

```java
CleverTap ct = new CleverTap(
        "YOUR_ACCOUNT_ID",
        "YOUR_PASSCODE",
        "YOUR_REGION"  // e.g. "in1", "sg1", or "" for default US
);

JSONObject result = ct.uploadEvent(
        "user-12345",
        "Product Viewed",
        new JSONObject()
                .put("Product Name", "Wireless Headphones")
                .put("Category", "Electronics")
                .put("Price", 79.99)
);

System.out.println(result);
// {"status":"success","processed":1,"unprocessed":[]}
```

### Batch events (recommended)

Batch uploads reduce API calls. The API accepts up to 1,000 records per request.

```java
List<JSONObject> events = List.of(
        new JSONObject()
                .put("identity", "user-12345")
                .put("eventName", "Product Viewed")
                .put("eventData", new JSONObject()
                        .put("Product Name", "Wireless Headphones")
                        .put("Price", 79.99)),
        new JSONObject()
                .put("identity", "user-12345")
                .put("eventName", "Added to Cart")
                .put("eventData", new JSONObject()
                        .put("Product Name", "Wireless Headphones")
                        .put("Quantity", 1)),
        new JSONObject()
                .put("identity", "user-67890")
                .put("eventName", "Checkout Completed")
                .put("eventData", new JSONObject()
                        .put("Amount", 149.99)
                        .put("Currency", "USD"))
                .put("timestamp", System.currentTimeMillis() / 1000)
);

JSONObject result = ct.uploadEvents(events);
System.out.println("Processed: " + result.getInt("processed"));
```

### Charged event (transactions)

```java
ct.uploadEvent(
        "user-12345",
        "Charged",
        new JSONObject()
                .put("Amount", 119.98)
                .put("Currency", "USD")
                .put("Charged ID", "order-12345")
                .put("Items", new JSONArray()
                        .put(new JSONObject()
                                .put("Product Name", "Wireless Headphones")
                                .put("Price", 79.99)
                                .put("Quantity", 1))
                        .put(new JSONObject()
                                .put("Product Name", "USB-C Cable")
                                .put("Price", 39.99)
                                .put("Quantity", 1)))
);
```

### Raw API call (without helper)

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import org.json.JSONArray;
import org.json.JSONObject;

HttpClient client = HttpClient.newHttpClient();

JSONObject payload = new JSONObject().put("d", new JSONArray().put(
        new JSONObject()
                .put("type", "event")
                .put("identity", "user-12345")
                .put("evtName", "Product Viewed")
                .put("evtData", new JSONObject()
                        .put("Product Name", "Wireless Headphones")
                        .put("Category", "Electronics")
                        .put("Price", 79.99))
));

HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://in1.api.clevertap.com/1/upload"))
        .header("X-CleverTap-Account-Id", "YOUR_ACCOUNT_ID")
        .header("X-CleverTap-Passcode", "YOUR_PASSCODE")
        .header("Content-Type", "application/json; charset=utf-8")
        .POST(HttpRequest.BodyPublishers.ofString(payload.toString()))
        .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.body());
```

---

## Upload User Profiles

### Single profile

```java
JSONObject result = ct.uploadProfile(
        "user-12345",
        new JSONObject()
                .put("Name", "Jane Doe")
                .put("Email", "jane@example.com")
                .put("Phone", "+14155551234")
                .put("Gender", "F")
                .put("Plan", "Premium")
);
```

### Batch profiles

```java
List<JSONObject> profiles = List.of(
        new JSONObject()
                .put("identity", "user-12345")
                .put("profileData", new JSONObject()
                        .put("Name", "Jane Doe")
                        .put("Email", "jane@example.com")
                        .put("Plan", "Premium")),
        new JSONObject()
                .put("identity", "user-67890")
                .put("profileData", new JSONObject()
                        .put("Name", "John Smith")
                        .put("Email", "john@example.com")
                        .put("Plan", "Free"))
);

JSONObject result = ct.uploadProfiles(profiles);
```

### Profile operations

```java
// Increment a numeric property
ct.uploadProfile("user-12345",
        new JSONObject().put("Loyalty Points", new JSONObject().put("$incr", 50)));

// Add to a multi-value property
ct.uploadProfile("user-12345",
        new JSONObject().put("Interests",
                new JSONObject().put("$add",
                        new JSONArray().put("Electronics").put("Music"))));

// Remove from a multi-value property
ct.uploadProfile("user-12345",
        new JSONObject().put("Interests",
                new JSONObject().put("$remove",
                        new JSONArray().put("Music"))));

// Manage messaging subscriptions
ct.uploadProfile("user-12345",
        new JSONObject()
                .put("MSG-email", true)
                .put("MSG-sms", false)
                .put("MSG-whatsapp", true));
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

```java
import java.net.http.HttpResponse;
import org.json.JSONObject;

public class CleverTapRetry {

    public static JSONObject uploadWithRetry(
            CleverTap ct, JSONArray records, int maxRetries) throws Exception {

        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                JSONObject result = ct.upload(records);

                JSONArray unprocessed = result.optJSONArray("unprocessed");
                if (unprocessed != null && unprocessed.length() > 0) {
                    System.err.println("Warning: " + unprocessed.length() + " records failed");
                    for (int i = 0; i < unprocessed.length(); i++) {
                        JSONObject rec = unprocessed.getJSONObject(i);
                        System.err.printf("  Code %s: %s%n",
                                rec.opt("code"), rec.opt("error"));
                    }
                }

                return result;

            } catch (RuntimeException e) {
                if (e.getMessage().contains("429") && attempt < maxRetries) {
                    long delay = (long) Math.pow(2, attempt) * 1000;
                    System.err.printf("Rate limited. Retrying in %dms...%n", delay);
                    Thread.sleep(delay);
                    continue;
                }
                throw e;

            } catch (java.io.IOException e) {
                if (attempt < maxRetries) {
                    long delay = (long) Math.pow(2, attempt) * 1000;
                    System.err.printf("Connection error. Retrying in %dms...%n", delay);
                    Thread.sleep(delay);
                    continue;
                }
                throw e;
            }
        }

        throw new RuntimeException("Max retries exceeded");
    }
}
```

### Dry run (validate without uploading)

Append `?dryRun=1` to the URL to validate payloads without persisting data. Useful for testing during development.

```java
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create(baseUrl + "/1/upload?dryRun=1"))
        .header("X-CleverTap-Account-Id", accountId)
        .header("X-CleverTap-Passcode", passcode)
        .header("Content-Type", "application/json; charset=utf-8")
        .POST(HttpRequest.BodyPublishers.ofString(payload.toString()))
        .build();
```

---

## Common Pitfalls

### 1. Passcode != Token

Server API authentication requires the **Passcode**, not the API Token. The Token is used by client-side SDKs. Using the wrong credential returns a 401 error. Find the Passcode under Settings > Project in the CleverTap dashboard.

### 2. Rate limits

The API allows a maximum of 15 concurrent requests per account. Exceeding this returns HTTP 429. Use batch uploads (up to 1,000 records per request) to stay within limits.

### 3. Batch uploads recommended

Prefer sending multiple records in a single request over individual API calls:

```java
// BAD: one API call per event
for (JSONObject event : events) {
    ct.uploadEvent(/* ... */);  // N API calls
}

// GOOD: one API call for all events
ct.uploadEvents(events);  // 1 API call (max 1,000 records)
```

For large datasets, chunk into batches of 1,000:

```java
import java.util.ArrayList;
import java.util.List;

public static <T> List<List<T>> partition(List<T> list, int size) {
    List<List<T>> partitions = new ArrayList<>();
    for (int i = 0; i < list.size(); i += size) {
        partitions.add(list.subList(i, Math.min(i + size, list.size())));
    }
    return partitions;
}

for (List<JSONObject> batch : partition(events, 1000)) {
    ct.uploadEvents(batch);
}
```

### 4. Timestamp format

The `ts` field expects a UNIX epoch timestamp **in seconds**, not milliseconds. Java's `System.currentTimeMillis()` returns milliseconds, so divide by 1000:

```java
// WRONG: milliseconds
record.put("ts", System.currentTimeMillis());

// CORRECT: seconds
record.put("ts", System.currentTimeMillis() / 1000);

// ALSO CORRECT: from Instant
import java.time.Instant;
record.put("ts", Instant.now().getEpochSecond());
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

```java
CleverTap ct = new CleverTap("YOUR_ACCOUNT_ID", "YOUR_PASSCODE", "YOUR_REGION");

// 1. Upload an event
ct.uploadEvent("user-12345", "Product Viewed",
        new JSONObject()
                .put("Product Name", "Headphones")
                .put("Price", 79.99));

// 2. Upload a user profile
ct.uploadProfile("user-12345",
        new JSONObject()
                .put("Name", "Jane Doe")
                .put("Email", "jane@example.com")
                .put("Plan", "Premium"));

// 3. Batch upload events
ct.uploadEvents(List.of(
        new JSONObject()
                .put("identity", "user-12345")
                .put("eventName", "Page View")
                .put("eventData", new JSONObject().put("Page", "/home")),
        new JSONObject()
                .put("identity", "user-67890")
                .put("eventName", "Page View")
                .put("eventData", new JSONObject().put("Page", "/pricing"))
));

// 4. Validate without persisting (dry run)
// Append ?dryRun=1 to the upload URL
```
