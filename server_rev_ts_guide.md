# Server Developer Guide: Revision and Timestamp Handling

This document explains how Nest devices use `revision` and `timestamp` fields for data synchronization, and what server implementations must do to maintain consistency.

---

## Overview

Nest devices use a dual-component versioning system to synchronize state with the cloud:

- **`timestamp`**: 64-bit integer, milliseconds since Unix epoch. Primary comparison key.
- **`revision`**: 32-bit signed integer, increments on each modification. Tiebreaker when timestamps are equal.

**The golden rule**: Timestamp wins. Revision is only consulted when timestamps are identical.

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

| Field | When Used | Meaning |
|-------|-----------|---------|
| `base_object_revision` | Normal buckets | Informational - device's current revision |
| `if_object_revision` | Conditional buckets | **Must match** server's revision or reject the PUT |

The `if_object_revision` field implements optimistic locking. If it doesn't match your stored revision, return HTTP 409 Conflict or similar - the device will re-fetch and retry.

---

## Conflict Resolution Rules

When the server receives data from a device, or when deciding what to send to a device, apply these rules **in order**:

### Rule 1: Compare Timestamps First

```
if (incoming_timestamp > stored_timestamp) → accept incoming data
if (incoming_timestamp < stored_timestamp) → reject as stale
if (incoming_timestamp == stored_timestamp) → go to Rule 2
```

### Rule 2: Compare Revisions (Tiebreaker)

Only when timestamps are exactly equal:

```
if (incoming_revision > stored_revision) → accept incoming data
if (incoming_revision <= stored_revision) → reject as stale
```

### Rule 3: Zero Timestamp Means "No Data"

A timestamp of `0` is a sentinel value meaning "I have no data for this bucket."

- **Device sends timestamp=0**: Device has no data. Server should push its current state.
- **Server has timestamp=0**: Server has no data. Accept whatever the device sends.

This occurs after device reboots or when a bucket is first created.

---

## Subscribe Request Handling

Devices send subscribe requests with their current rev/ts for each bucket:

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

For each requested bucket:

1. **If device timestamp is 0**: Always send your current data
2. **If your timestamp > device timestamp**: Send your current data
3. **If your timestamp < device timestamp**: Send nothing (or empty object) - device is ahead
4. **If timestamps equal and your revision > device revision**: Send your current data
5. **If timestamps equal and your revision <= device revision**: Send nothing - device is current

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

1. **Check conditional writes**: If `if_object_revision` is present, verify it matches your stored revision. If not, reject with 409.

2. **Apply conflict resolution**: Compare incoming rev/ts against stored values using the rules above.

3. **If accepting the update**:
   - Store the new value
   - Store the incoming revision (or increment your own - see note below)
   - Store the incoming timestamp (or use current server time - see note below)

4. **Return the new rev/ts** in the response so the device knows the write succeeded.

### Note on Rev/Ts Authority

You have two options:

**Option A - Trust device values**: Store exactly what the device sends. Simpler, but relies on device clocks being reasonably accurate.

**Option B - Server authoritative**: Ignore device rev/ts, use server time and increment your own revision counter. More consistent, but devices auto-correct clocks if skew exceeds 10 minutes anyway.

Either works. The key is consistency - pick one approach and stick with it.

---

## Time Synchronization

Include the current server time in response headers:

```
X-nl-service-timestamp: 1706900000000
```

This is Unix timestamp in **milliseconds** (same as `object_timestamp` fields). Devices use this to detect clock skew. If the device's clock differs from the server by more than 10 minutes, it will automatically correct itself.

---

## Edge Case: Server Offline Then Returns

**Scenario**: Server was down, device made local changes, server comes back.

**What happens**:
1. Device sends subscribe with its current (newer) rev/ts
2. Server compares and sees device timestamp > server timestamp
3. Server recognizes device has newer data
4. Device sends PUT with its changes
5. Server accepts (device timestamp wins)

**No special handling required** - the timestamp comparison naturally resolves this.

---

## Edge Case: Device Reboots

**Scenario**: Device loses power, reboots, reconnects.

**What happens**:
1. Device initializes with revision=0, timestamp=0
2. Device sends subscribe with rev=0, ts=0
3. Server sees timestamp=0 (sentinel for "no data")
4. Server pushes its current state to the device

**Server requirement**: Always push data when device timestamp is 0.

---

## Edge Case: Race Between PUT and Subscribe

**Scenario**: Device sends PUT, then receives subscribe response that was in-flight.

**What happens**: The device handles this internally with an "embargo" system - it buffers subscribe responses while a PUT is in progress, then applies conflict resolution after the PUT completes.

**Server requirement**: None. Just respond to requests normally. The device handles the race.

---

## Edge Case: Simultaneous Modifications

**Scenario**: Device and server both modify the same bucket at nearly the same time.

**Resolution**: Whichever write has the later timestamp wins. If timestamps are identical (unlikely but possible), higher revision wins. If both are identical, the receiver of the message keeps its version (effectively "last writer wins" at the network level).

**Server requirement**: Ensure your clock is reasonably accurate. Consider using NTP.

---

## Quick Reference: Response Codes

| Situation | HTTP Status |
|-----------|-------------|
| Bucket updated successfully | 200 OK |
| Conditional write revision mismatch | 409 Conflict |
| Bucket not found | 404 Not Found |
| Malformed request | 400 Bad Request |

On 404, devices will invalidate their local copy of that bucket.

On 400, devices will retry up to 3 times before giving up.

---

## Implementation Checklist

- [ ] Store `revision` (int32) and `timestamp` (int64) for each bucket
- [ ] Update both atomically on every bucket write
- [ ] Compare timestamp first, revision second
- [ ] Treat timestamp=0 as "no data" sentinel
- [ ] Support `object_revision` in subscribe responses
- [ ] Support both `base_object_revision` and `if_object_revision` in PUTs
- [ ] Implement conditional write rejection for `if_object_revision` mismatches
- [ ] Include `X-nl-service-timestamp` header in responses
- [ ] Return appropriate HTTP status codes (200, 400, 404, 409)

---

## Summary

| Principle | Rule |
|-----------|------|
| Primary comparison | Larger timestamp wins |
| Tiebreaker | Larger revision wins (when timestamps equal) |
| Equal rev and ts | Receiver keeps its data |
| Zero timestamp | Means "no data", always yields |
| Conditional writes | `if_object_revision` must match or reject |
| Clock sync | Send `X-nl-service-timestamp` header |

Follow these rules and your server will maintain consistency with Nest devices through any combination of reboots, network outages, and concurrent modifications.
