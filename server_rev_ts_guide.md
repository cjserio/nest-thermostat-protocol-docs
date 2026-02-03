# Server Developer Guide: Revision and Timestamp Handling

This document explains how to handle `revision` and `timestamp` fields when implementing a server that communicates with Nest devices.

---

## Overview

Nest devices use a versioning system to synchronize state with the cloud:

- **`timestamp`**: 64-bit integer, milliseconds since Unix epoch. **This is the sole authority for sync decisions.**
- **`revision`**: 32-bit signed integer, increments on each modification. The server must store and return this value, but it is only validated for conditional writes (when `if_object_revision` is present).

**The golden rule**: Timestamp alone determines which data is newer. Revision is not used for sync decisions.

---

## Data Model Requirements

For each bucket (e.g., `device.SERIAL123`, `shared.SERIAL123`), the server must store:

| Field | Type | Description |
|-------|------|-------------|
| `revision` | int32 | Increments on every write operation |
| `timestamp` | int64 | Milliseconds since epoch at time of write |
| `value` | object | The actual bucket data |

Both fields must be updated atomically with every bucket modification.

---

## JSON Field Names

Different field names are used in different contexts:

### Subscribe Responses (Server → Device)

Always use `object_revision` and `object_timestamp`:

```json
{
  "device.SERIAL123": {
    "object_revision": 42,
    "object_timestamp": 1706900000000,
    "value": { ... }
  }
}
```

### PUT Requests (Device → Server)

The device sends one of two revision fields depending on bucket type:

| Field | When Used | Purpose |
|-------|-----------|---------|
| `base_object_revision` | Most buckets | Informational only - no server action required |
| `if_object_revision` | Certain buckets | **Conditional write** - must match server's revision or reject |

When `if_object_revision` is present, the server must verify it matches the stored revision before accepting the write. If it doesn't match, return HTTP 409 Conflict. This prevents race conditions when multiple sources might modify the same bucket.

---

## Sync Decision Rules

When deciding whether to accept incoming data or what to send to a device, use **only the timestamp**:

### Rule 1: Compare Timestamps

```
if (incoming_timestamp > stored_timestamp) → accept incoming data
if (incoming_timestamp < stored_timestamp) → reject as stale
if (incoming_timestamp == stored_timestamp) → already synced, no action needed
```

### Rule 2: Zero Timestamp Means "No Data"

A timestamp of `0` is a sentinel value meaning "I have no data for this bucket."

- **Device sends timestamp=0**: Device has no data. Server should send its current state.
- **Server has timestamp=0**: Server has no data. Accept whatever the device sends.

This occurs after device reboots or when a bucket is first created.

---

## Subscribe Request Handling

Devices send subscribe requests with their current revision and timestamp for each bucket:

```json
{
  "objects": [
    {
      "object_key": "device.SERIAL123",
      "object_revision": 42,
      "object_timestamp": 1706900000000
    }
  ]
}
```

### Server Response Logic

For each requested bucket, compare timestamps:

1. **If device timestamp is 0**: Always send your current data
2. **If your timestamp > device timestamp**: Send your current data
3. **If your timestamp < device timestamp**: Send nothing - device has newer data
4. **If timestamps are equal**: Send nothing - already synced

For long-polling implementations: if no buckets have updates, hold the connection open until data changes or timeout.

---

## PUT Request Handling

When a device sends a PUT:

```json
{
  "object_revision": 43,
  "object_timestamp": 1706900001000,
  "value": { ... }
}
```

### Server Processing

1. **Check for conditional write**: If `if_object_revision` is present, verify it matches your stored revision. If not, reject with 409 Conflict.

2. **Compare timestamps**: Use the sync decision rules above. Only accept if the incoming timestamp is greater than your stored timestamp (or your timestamp is 0).

3. **If accepting the update**:
   - Store the new value
   - Store the incoming revision (or increment your own)
   - Store the incoming timestamp (or use current server time)

4. **Return the new revision and timestamp** in the response.

### Note on Rev/Ts Authority

You have two options:

**Option A - Trust device values**: Store exactly what the device sends. Simpler, but relies on device clocks being reasonably accurate.

**Option B - Server authoritative**: Use server time and increment your own revision counter. More consistent, but devices auto-correct clocks if skew exceeds 10 minutes anyway.

Either approach works. The key is consistency - pick one and stick with it.

---

## Time Synchronization

Include the current server time in response headers:

```
X-nl-service-timestamp: 1706900000000
```

This is Unix timestamp in **milliseconds**. Devices use this to detect and correct clock skew. The threshold for automatic correction is 10 minutes.

---

## Edge Cases

### Server Offline Then Returns

**Scenario**: Server was down, device made local changes, server comes back.

**Resolution**: Timestamp comparison handles this automatically. The device's newer timestamp wins.

### Device Reboots

**Scenario**: Device loses power, reboots, reconnects.

**What happens**: Device sends timestamp=0, indicating it has no data. Server must push its current state.

**Server requirement**: Always send data when device timestamp is 0.

### Simultaneous Modifications

**Scenario**: Device and server both modify the same bucket at nearly the same time.

**Resolution**: Whichever write has the later timestamp wins. If timestamps happen to be identical, both sides consider themselves synced and no overwrite occurs.

**Server requirement**: Ensure your clock is accurate. Use NTP.

---

## HTTP Response Codes

| Situation | HTTP Status |
|-----------|-------------|
| Bucket updated successfully | 200 OK |
| Conditional write failed (`if_object_revision` mismatch) | 409 Conflict |
| Bucket not found | 404 Not Found |
| Malformed request | 400 Bad Request |

**Verified device behavior:**
- **400**: Device retries up to 2 times (3 total attempts) before giving up

---

## Implementation Checklist

- [ ] Store `revision` (int32) and `timestamp` (int64) for each bucket
- [ ] Update both atomically on every bucket write
- [ ] Use timestamp as the sole authority for sync decisions
- [ ] Treat timestamp=0 as "no data" sentinel
- [ ] Include `object_revision` and `object_timestamp` in subscribe responses
- [ ] Handle both `base_object_revision` (ignore) and `if_object_revision` (validate) in PUTs
- [ ] Return 409 Conflict when `if_object_revision` doesn't match
- [ ] Include `X-nl-service-timestamp` header in responses
- [ ] Return appropriate HTTP status codes

---

## Summary

| Principle | Server Action |
|-----------|---------------|
| Sync decisions | Compare timestamps only |
| Equal timestamps | No action needed - already synced |
| Timestamp = 0 | Sender has no data, receiver should provide theirs |
| `if_object_revision` present | Validate against stored revision, reject with 409 if mismatch |
| `base_object_revision` present | Informational only, no validation needed |
| Clock sync | Send `X-nl-service-timestamp` header in responses |

Follow these rules and your server will maintain consistency with Nest devices through reboots, network outages, and concurrent modifications.
