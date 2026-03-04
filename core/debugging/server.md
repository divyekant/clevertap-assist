# CleverTap Server API — Debugging Reference

> Platform-specific debugging guide for the CleverTap REST API (server-side).
> For cross-platform issues (identity, data constraints, dashboard lag), see `common.md`.

---

## Diagnostic Checklist

Run through these checks in order when something is not working:

1. **Passcode, not Token** — Are you using the **Passcode** (from Settings > Project > Account Credentials), not the API Token? These are different credentials. The Token is for client SDKs only.
2. **Header names** — Is the Account ID header exactly `X-CleverTap-Account-Id` and the Passcode header exactly `X-CleverTap-Passcode`? (Case-sensitive on some proxies.)
3. **Content-Type** — Is the header set to `application/json; charset=utf-8`?
4. **Base URL region** — Does the API base URL match your account's region? (e.g., `in1.api.clevertap.com` for India, `sg1.api.clevertap.com` for Singapore, `api.clevertap.com` for default US.)
5. **Payload structure** — Is the JSON body wrapped in a `{ "d": [ ... ] }` envelope?
6. **Identity field** — Does every record include exactly one identity field (`identity`, `FBID`, `GPID`, or `objectId`)?
7. **Timestamp format** — If using the `ts` field, is it in seconds (not milliseconds)?
8. **Response body** — Even on HTTP 200, does the response body show `"status": "success"` or does it contain `unprocessed` records?
9. **Rate limits** — Are you staying within the 15 concurrent requests limit? Are you batching records (up to 1,000 per request)?
10. **Dry run first** — For new integrations, test with `?dryRun=1` appended to the upload URL to validate payloads without persisting data.

---

## Symptoms, Causes, and Fixes

### Authentication failures

| Symptom | Likely Cause | Fix |
|---|---|---|
| HTTP `401 Unauthorized` | Using the API Token instead of the Passcode | Replace `X-CleverTap-Passcode` value with the Passcode from Settings > Project > Account Credentials (not the Token) |
| HTTP `401` — Passcode is correct | Wrong Account ID | Verify `X-CleverTap-Account-Id` matches the Account ID on the dashboard |
| HTTP `401` — both credentials correct | Wrong region base URL | If your dashboard is at `in1.dashboard.clevertap.com`, use `https://in1.api.clevertap.com`. Using the wrong region returns 401 even with valid credentials. |
| HTTP `401` — everything looks correct | Header name misspelled or case mismatch through a proxy | Verify exact header names: `X-CleverTap-Account-Id` (capital I in Id) and `X-CleverTap-Passcode` (capital P). Some reverse proxies normalize header casing. |
| HTTP `401` — intermittent | Passcode was rotated on the dashboard | Re-fetch the Passcode from Settings > Project; update your server configuration |

### Rate limiting

| Symptom | Likely Cause | Fix |
|---|---|---|
| HTTP `429 Too Many Requests` | More than 15 concurrent requests to the API | Reduce concurrency; implement a request queue or semaphore limiting to 15 parallel requests |
| Frequent `429` despite low request volume | Individual event uploads instead of batches | Batch records: send up to 1,000 records per request instead of one record per request |
| `429` during bulk import | Sending thousands of requests without throttling | Chunk data into batches of 1,000 records; add a concurrency limiter (max 15) with exponential backoff on 429 responses |

### Partial success / unprocessed records

| Symptom | Likely Cause | Fix |
|---|---|---|
| HTTP 200 but `"status": "partial"` | Some records in the batch failed validation | Inspect the `unprocessed` array in the response; each entry has a `code` and `error` field explaining the failure |
| `unprocessed` error: "Event name is mandatory" | Missing `evtName` field in an event record | Ensure every event record has `"evtName": "..."` |
| `unprocessed` error: "Identity is mandatory" | Missing identity field in a record | Add `"identity": "..."` (or `FBID`, `GPID`, `objectId`) to each record |
| `unprocessed` error: "Invalid event data" | Property key exceeds 120 characters, or property value exceeds 512 bytes | Shorten property keys and values; check data constraints |
| `unprocessed` error: "Invalid profile data" | Profile data contains unsupported types or operations | Verify profile data values are strings, numbers, booleans, or supported operations (`$incr`, `$add`, `$remove`) |
| All records in `unprocessed` | JSON structure wrong (e.g., records not inside `"d"` array) | Ensure the request body is `{ "d": [ record1, record2, ... ] }` |

### Events not appearing in dashboard

| Symptom | Likely Cause | Fix |
|---|---|---|
| API returns success but events not in dashboard | Timestamp in milliseconds instead of seconds, causing events to be dated far in the future | Convert timestamps: divide JavaScript `Date.now()` by 1000; use `int(time.time())` in Python; use `System.currentTimeMillis() / 1000` in Java |
| Events not in dashboard — timestamp is correct | Identity field does not match any existing profile and a new profile was created under a different identity | Verify the `identity` value matches what the client SDK uses for the same user |
| Events not in dashboard — region mismatch | API URL points to a different region than the dashboard you are checking | Ensure the base URL region matches the dashboard region |
| Events in dashboard but under wrong user | Identity value has leading/trailing whitespace or case difference | Trim and normalize identity values before sending |
| Delayed appearance | Normal dashboard processing lag (up to 5-10 minutes) | Wait and refresh; real-time events panel in the dashboard shows events faster than aggregate reports |

### Request failures

| Symptom | Likely Cause | Fix |
|---|---|---|
| HTTP `400 Bad Request` | Malformed JSON body | Validate your JSON (use `JSON.parse` / `json.loads` on the body before sending); check for trailing commas, unquoted keys, or encoding issues |
| HTTP `500 Internal Server Error` | Server-side issue at CleverTap | Retry with exponential backoff; if persistent, check [status.clevertap.com](https://status.clevertap.com) |
| Connection timeout | Network issue or firewall blocking outbound HTTPS to CleverTap API | Verify outbound HTTPS (port 443) to `*.api.clevertap.com` is allowed; check proxy/firewall rules |
| SSL/TLS errors | Outdated TLS version or missing CA certificates | Ensure your HTTP client supports TLS 1.2+; update CA certificate bundle |

---

## Debug Tooling

### Dry run mode

Append `?dryRun=1` to the upload URL to validate your payload without persisting any data. The response format is identical to a real upload, including any validation errors.

```
POST https://in1.api.clevertap.com/1/upload?dryRun=1
```

Use this for:
- Validating payload structure during development
- Testing authentication credentials
- Verifying record format before a bulk import

### HTTP response body inspection

Always inspect the full response body, not just the HTTP status code. A `200` response can still contain failures:

```json
{
  "status": "partial",
  "processed": 8,
  "unprocessed": [
    {
      "status": "fail",
      "code": 514,
      "error": "Event name is mandatory",
      "record": { ... }
    },
    {
      "status": "fail",
      "code": 515,
      "error": "Identity is mandatory",
      "record": { ... }
    }
  ]
}
```

### Common error codes in `unprocessed`

| Code | Meaning |
|---|---|
| 511 | Invalid identity field |
| 512 | Invalid event data |
| 513 | Invalid profile data |
| 514 | Event name is mandatory |
| 515 | Identity is mandatory |
| 516 | Record type is mandatory (must be `"event"` or `"profile"`) |
| 521 | Invalid timestamp |

### Request logging pattern

Log the request and response for every API call during development:

```javascript
// Node.js example
async function uploadWithLogging(ct, records) {
  console.log("[CT] Upload request:", JSON.stringify({ d: records }, null, 2));
  const result = await ct.upload(records);
  console.log("[CT] Upload response:", JSON.stringify(result, null, 2));
  if (result.unprocessed?.length > 0) {
    console.error("[CT] Unprocessed records:", result.unprocessed);
  }
  return result;
}
```

```python
# Python example
import json

def upload_with_logging(ct, records):
    print(f"[CT] Upload request: {json.dumps({'d': records}, indent=2)}")
    result = ct.upload(records)
    print(f"[CT] Upload response: {json.dumps(result, indent=2)}")
    if result.get("unprocessed"):
        print(f"[CT] Unprocessed: {result['unprocessed']}")
    return result
```

### cURL test command

Test your credentials and payload directly from the command line:

```bash
curl -X POST "https://YOUR_REGION.api.clevertap.com/1/upload?dryRun=1" \
  -H "X-CleverTap-Account-Id: YOUR_ACCOUNT_ID" \
  -H "X-CleverTap-Passcode: YOUR_PASSCODE" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '{
    "d": [{
      "type": "event",
      "identity": "test-user",
      "evtName": "Test Event",
      "evtData": {"key": "value"}
    }]
  }'
```

If this returns `{"status": "success", "processed": 1, "unprocessed": []}`, your credentials and region are correct.

---

## Known Issues

### Rate limit specifics

| Limit | Value |
|---|---|
| Max concurrent requests per account | 15 |
| Max records per request | 1,000 |
| Max event types per account | 512 |
| Max custom properties per event type | 256 |
| Max property key length | 120 characters |
| Max property value size | 512 bytes |
| Max event name length | 512 characters |

When the concurrency limit is hit, the API returns HTTP 429. Implement exponential backoff:

```
Attempt 1: wait 2 seconds
Attempt 2: wait 4 seconds
Attempt 3: wait 8 seconds
```

### Payload size limits

| Constraint | Limit |
|---|---|
| Request body size | No hard documented limit, but keep under 5 MB per request |
| Records per request | 1,000 |
| Recommended batch size | 500-1,000 records for optimal throughput |

For large data migrations, chunk into batches of 1,000 and limit to 15 concurrent uploads:

```
Total records: 100,000
Batch size: 1,000 records
Batches: 100
Concurrent uploads: 15
Estimated time: ~7 batches in sequence (100 / 15) with ~1s per batch = ~7 seconds
```

### Timestamp edge cases

| Issue | Details |
|---|---|
| JavaScript `Date.now()` is in milliseconds | Divide by 1000: `Math.floor(Date.now() / 1000)` |
| Python `time.time()` is a float in seconds | Convert to int: `int(time.time())` |
| Java `System.currentTimeMillis()` is in milliseconds | Divide by 1000: `System.currentTimeMillis() / 1000` or use `Instant.now().getEpochSecond()` |
| Timestamps far in the future | Events with future timestamps are accepted but may not appear in reports until that date passes. Double-check your conversion. |
| Timestamps far in the past | Events older than the account's data retention window may be silently dropped. |
| Omitting `ts` entirely | The server uses the current time. This is the safest option if you do not need historical timestamps. |

### Passcode vs. Token confusion

| Credential | Used By | Header Name | Where to Find |
|---|---|---|---|
| **Passcode** | Server API (REST) | `X-CleverTap-Passcode` | Dashboard > Settings > Project > Account Credentials |
| **Token** | Client SDKs (iOS, Android, Web, RN, Flutter) | N/A (configured in Info.plist / manifest / init) | Dashboard > Settings > Project > Account Credentials |

Using the Token in `X-CleverTap-Passcode` returns `401`. This is the most common authentication mistake for server-side integrations.
