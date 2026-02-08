# Nest Cloud Protocol Reference

Protocol specification for implementing a server that communicates with Nest thermostat devices.

**Revision**: 1.8
**Last updated**: 2026-02-07

---

## Table of contents

- [Overview](#overview)
- [Endpoints](#endpoints)
  - [POST /{czid}/subscribe](#post-czidsubscribe)
  - [POST /{czid}/put](#post-czidput)
  - [POST /entry](#post-entry)
- [Authentication (provisional)](#authentication-provisional)
  - [Device identification](#device-identification)
  - [Credential types](#credential-types)
  - [Credential provisioning](#credential-provisioning)
  - [Recommended approach for home servers](#recommended-approach-for-home-servers)
  - [Entry key](#entry-key)
- [Pairing](#pairing)
  - [Pairing flow](#pairing-flow)
  - [Complete pairing on the device](#complete-pairing-on-the-device)
  - [Maintain pairing state across reconnections](#maintain-pairing-state-across-reconnections)
- [Connection lifecycle](#connection-lifecycle)
  - [Server push wake mechanism](#server-push-wake-mechanism)
  - [Service tickle](#service-tickle-administrative-only)
- [Server implementation notes](#server-implementation-notes)
  - [Session ID behavior](#session-id-behavior)
  - [Overlapping subscriptions](#overlapping-subscriptions)
  - [Batching multiple pushes](#batching-multiple-pushes)
- [Versioning and synchronization](#versioning-and-synchronization)
- [Response headers](#response-headers)
  - [X-nl-suspend-time-max](#x-nl-suspend-time-max)
  - [X-nl-defer-device-window](#x-nl-defer-device-window)
  - [X-nl-disable-defer-window](#x-nl-disable-defer-window)
  - [X-nl-service-timestamp](#x-nl-service-timestamp)
- [Timing reference](#timing-reference)
  - [Concurrent PUT and subscribe](#concurrent-put-and-subscribe)
- [Battery behavior](#battery-behavior)
- [Bucket types](#bucket-types)
  - [Display wake behavior](#display-wake-behavior)
- [Home/Away mode](#homeaway-mode)
  - [Set away mode](#set-away-mode)
  - [Set home mode](#set-home-mode)
  - [Timestamp requirement](#timestamp-requirement)
  - [Why not the away field?](#why-not-the-away-field)
  - [Deliver the structure bucket](#deliver-the-structure-bucket)
  - [Choose the structure key](#choose-the-structure-key)
  - [Device bucket away fields (read-only)](#device-bucket-away-fields-read-only)
  - [Eco temperatures](#eco-temperatures)
  - [Exiting eco mode](#exiting-eco-mode)
- [Error handling](#error-handling)
- [Implementation checklist](#implementation-checklist)
- [Examples](#examples)
- [Appendix: URL port requirement](#appendix-url-port-requirement)
- [Document status](#document-status)
- [Changelog](#changelog)

---

## Overview

The Nest thermostat uses a long-polling HTTP protocol to maintain bidirectional communication with cloud servers. The device initiates connections to the server, subscribes to state changes, and receives pushed updates. This design allows the device to sleep for extended periods while remaining instantly responsive to server-initiated commands.

### Key concepts

- **Subscribe connection**: A persistent HTTP connection where the device waits for server updates
- **Server push**: The ability to wake a sleeping device by sending data on an open connection
- **Chunked encoding**: Required for server push capability; enables the server to hold connections open indefinitely
- **Buckets**: Named data containers (e.g., `device`, `shared`, `schedule`) that hold device state

### Protocol characteristics

| Characteristic | Value |
|----------------|-------|
| Transport | HTTP/1.1 over TLS |
| Encoding | JSON request/response bodies |
| Connection model | Device-initiated, server-held |
| Push mechanism | Chunked transfer encoding |
| Keep-alive | Device sends TCP keep-alives during sleep |

---

## Endpoints

> **Note**: Server URLs must always include an explicit port (e.g., `:443`). See [Appendix: URL port requirement](#appendix-url-port-requirement).

### POST /{czid}/subscribe

Long-poll endpoint for server-to-device communication. The device maintains an open connection to receive pushed updates.

#### Request

```http
POST /{czid}/subscribe HTTP/1.1
Host: your-server.example.com
Content-Type: application/json
X-nl-protocol-version: 1

{
  "chunked": true,
  "session": "session_id",
  "device": {"object_key": "device.SERIAL", "object_revision": 123, "object_timestamp": 1234567890},
  "shared": {"object_key": "shared.SERIAL", "object_revision": 456, "object_timestamp": 1234567890}
}
```

#### Request headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-nl-protocol-version` | Yes | Protocol version. Always `1`. |
| `X-nl-device-swversion` | No | Device software version string. |
| `X-nl-longest-wake` | No | Vestigial. Running max of subscribe connection durations (seconds); never resets. Server ignores. |
| `X-nl-client-id` | No | Sent when device has no valid session. |

#### Request body

| Field | Type | Description |
|-------|------|-------------|
| `chunked` | boolean | Always `true`. Device requests chunked response. |
| `session` | string | Device session identifier. The device reuses this value across requests; don't use it as a unique subscription key. See [Session ID behavior](#session-id-behavior). |
| `{bucket}` | object | One entry per bucket with current revision info. |
| `{bucket}.object_key` | string | Bucket identifier, typically `{bucket}.{serial}`. |
| `{bucket}.object_revision` | integer | Device's current revision number for this bucket. |
| `{bucket}.object_timestamp` | integer | Unix timestamp of last update. |

#### Response

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 600
X-nl-service-timestamp: 1707148800000

{"shared": {"object_revision": 457, "object_timestamp": 1707148800000, "object_key": "shared.09AA01AB12345678", "value": {"target_temperature": 72.0}}}
```

#### Response headers

| Header | Required | Description |
|--------|----------|-------------|
| `Transfer-Encoding` | Yes | Must be `chunked` for server push capability. |
| `X-nl-suspend-time-max` | Yes | Maximum seconds before device will reconnect. See [Suspend Time](#x-nl-suspend-time-max). |
| `X-nl-service-timestamp` | Recommended | Server timestamp in milliseconds since Unix epoch. Used for sync decisions. |
| `X-nl-set-client-credentials` | No | Set device credentials as `"userid password"`. |
| `X-nl-defer-device-window` | No | Delay window for device-initiated updates. See [X-nl-defer-device-window](#x-nl-defer-device-window). |
| `X-nl-disable-defer-window` | No | Temporarily disable defer delay. See [X-nl-defer-device-window](#x-nl-defer-device-window). |

#### Response body

Return a JSON object containing updated bucket data. Only include buckets that have changed since the device's reported revision.

To indicate no updates, hold the connection open without sending body data. Don't send an empty object.

**Response format**: Wrap bucket data in an `objects` array with a nested `value` object:

```json
{
  "objects": [
    {
      "object_revision": 458,
      "object_timestamp": 1707148800000,
      "object_key": "shared.SERIAL",
      "value": {
        "target_temperature": 21.5
      }
    }
  ]
}
```

> **Note**: Subscribe responses use a different format than PUT requests. Subscribe responses wrap data in `objects` array with nested `value`; PUT requests send bucket data at the top level with fields inline. See [PUT endpoint](#post-czidput) for the PUT format.

> **WARNING: JSON Field Ordering is Critical**
>
> The device's JSON parser expects `object_revision` and `object_timestamp` fields to appear **BEFORE** `object_key` in response objects. Incorrect field ordering can cause parsing failures, sync issues, or the device ignoring updates entirely.
>
> **Correct:**
> ```json
> {"object_revision": 458, "object_timestamp": 1707148800000, "object_key": "shared.SERIAL", "value": {...}}
> ```
>
> **Incorrect (will cause issues):**
> ```json
> {"object_key": "shared.SERIAL", "object_revision": 458, "object_timestamp": 1707148800000, "value": {...}}
> ```
>
> This applies to both flat response format and the `objects` array format.
>
> **Implementation note**: Most JSON serialization libraries don't guarantee field order. You may need to use an ordered dictionary, manual string building, or a library that preserves insertion order.

#### Errors

| Status | Meaning | Device Behavior |
|--------|---------|-----------------|
| 200 | Success | Process response, resubscribe |
| 401 | Unauthorized | Re-authenticate |
| 5xx | Server error | Retry with backoff |

---

### POST /{czid}/put

Device-to-server state updates. The device sends local state changes to the server.

#### Request

```http
POST /{czid}/put HTTP/1.1
Host: your-server.example.com
Content-Type: application/json
X-nl-protocol-version: 1

{
  "session": "session_id",
  "shared.09AB1234": {
    "object_key": "shared.09AB1234",
    "base_object_revision": 457,
    "target_temperature": 21.5,
    "target_temperature_type": "heat"
  }
}
```

#### Request headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-nl-protocol-version` | Yes | Protocol version. Always `1`. |
| `X-nl-client-id` | No | Sent when device has no valid session. |
| `X-nl-device-swversion` | No | Device software version string. |
| `Content-Type` | Yes | Must be `application/json`. |

#### Request body

The request body contains bucket data at the top level, keyed by bucket identifier.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session` | string | Yes | Session identifier from authentication. |
| `{bucket_key}` | object | Yes | One or more bucket objects, keyed by identifier (e.g., `shared.SERIAL`). |

#### Bucket object fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `object_key` | string | Yes | Bucket identifier, matches the parent key. |
| `base_object_revision` | integer | Yes | Revision this update is based on. |
| `if_object_revision` | integer | No | Conditional write guard. See below. |
| `{field}` | varies | Yes | One or more data fields to update (e.g., `target_temperature`). |

#### Revision fields in PUT requests

The device uses two different revision fields depending on the bucket type:

| Field | When used | Server behavior |
|-------|-----------|-----------------|
| `base_object_revision` | Most buckets | Informational only—no validation required |
| `if_object_revision` | Certain bucket types | Conditional write guard |

When the device sends `if_object_revision`, it expects the server to validate that this revision matches the current stored revision before accepting the write. The specific server response for validation failure is implementation-defined.

> **Note**: Unlike subscribe requests which use `object_revision`, PUT requests use `base_object_revision` or `if_object_revision` depending on bucket type.

#### Conflict detection

When accepting a PUT request:

1. Compare incoming `base_object_revision` against stored revision.
2. If they match, accept the update and increment revision.
3. If they don't match, return `200 OK` but include the current server state in the response so the device can reconcile.

The device handles conflicts by comparing timestamps. If the server's data is newer, the device accepts it. If the device's data is newer, it retries the PUT.

#### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

Return `200 OK` to acknowledge receipt. The response body is optional.

#### Errors

| Status | Meaning | Device behavior |
|--------|---------|-----------------|
| 200 | Update received | Continue normal operation |
| 400 | Malformed request | Retry up to 2 times, then give up |
| 401 | Unauthorized | Re-authenticate |
| 5xx | Server error | Retry with backoff |

#### Example: temperature change

```http
POST /abc123/put HTTP/1.1
Host: your-server.example.com:443
Content-Type: application/json
X-nl-protocol-version: 1

{
  "session": "sess_xyz789",
  "shared.09AB1234": {
    "object_key": "shared.09AB1234",
    "base_object_revision": 457,
    "target_temperature": 21.5,
    "target_temperature_type": "heat"
  }
}
```

#### Example: multiple buckets

The device can update multiple buckets in a single PUT:

```http
POST /abc123/put HTTP/1.1
Host: your-server.example.com:443
Content-Type: application/json
X-nl-protocol-version: 1

{
  "session": "sess_xyz789",
  "shared.09AB1234": {
    "object_key": "shared.09AB1234",
    "base_object_revision": 457,
    "target_temperature": 21.5
  },
  "device.09AB1234": {
    "object_key": "device.09AB1234",
    "base_object_revision": 122,
    "fan_timer_timeout": 900
  }
}
```

---

### POST /entry

Initial device registration endpoint. The device contacts this endpoint on first boot, after factory reset, or when re-registering with the server. The response provides the `transport_url` used for all subsequent communication.

#### Request

```http
POST /entry HTTP/1.1
Host: your-server.example.com
Content-Type: application/x-www-form-urlencoded
X-nl-device-id: 09AA01AB12345678

reset=FALSE&mac=18B430ABCDEF&model=Diamond-2.6&request_id=1&software_version=5.9.3-5&wireless_reg_domain=US&backplate_model=Backplate-2.1
```

#### Request headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-nl-device-id` | Yes | Device serial number. |
| `Content-Type` | Yes | Must be `application/x-www-form-urlencoded`. |

#### Request body (form-urlencoded)

| Field | Required | Description |
|-------|----------|-------------|
| `reset` | Yes | `TRUE` if device was factory reset, `FALSE` otherwise. |
| `mac` | Yes | Device WiFi MAC address (12 hex chars, no separators). |
| `model` | Yes | Device model string (e.g., `Diamond-2.6`, `Flintstone-4.0`). |
| `request_id` | Yes | Monotonically increasing request counter. |
| `software_version` | Yes | Current firmware version (e.g., `5.9.3-5`). |
| `wireless_reg_domain` | No | WiFi regulatory domain (e.g., `US`, `EU`). |
| `backplate_model` | No | Backplate model string (thermostat only). |

#### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "transport_url": "https://your-server.example.com:443"
}
```

#### Response headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-nl-set-client-credentials` | No | Provisions device credentials as `"userid password"`. |

#### Response body

| Field | Required | Description |
|-------|----------|-------------|
| `transport_url` | Yes | Base URL for all subsequent API calls (`subscribe`, `put`). **Must include explicit port.** |
| `passphrase_url` | No | Entry key fetch endpoint for device pairing. |
| `ping_url` | No | Connectivity check endpoint. |
| `weather_url` | No | Weather data endpoint. |
| `upload_url` | No | Device log upload endpoint. |

#### Device behavior after registration

1. Device stores `transport_url` for subsequent requests.
2. Device proceeds to `POST /{czid}/subscribe` using the `transport_url`.
3. Registration is cached; device re-registers periodically or after errors.

#### Errors

| Status | Meaning | Device behavior |
|--------|---------|-----------------|
| 200 | Success | Store transport_url, proceed to subscribe |
| 302 | Redirect | Follow redirect, retry registration |
| 401 | Unauthorized | Log error, retry with backoff |
| 5xx | Server error | Retry with backoff |

#### Example

```json
{
  "transport_url": "https://your-server.example.com:443"
}
```

---

## Authentication (provisional)

> **Note**: The device's authentication and pairing system was designed for Google's cloud infrastructure and companion mobile app. That architecture handles credential provisioning, account linking, and pairing through a coordinated flow between the phone, cloud, and device — none of which exists in a home server deployment. The approaches documented here are functional but may not cover all firmware edge cases. If you encounter unexpected behavior, [file an issue](https://github.com/cjserio/nest-thermostat-protocol-docs/issues).

Authentication is optional. The device supports HTTP Basic Authentication for subscribe and put requests, but you can also identify devices entirely through HTTP headers without provisioning credentials.

### Device identification

Even without credentials, the device identifies itself in every request through HTTP headers:

| Header | Format | Sent when | Found in |
|--------|--------|-----------|----------|
| `X-nl-client-id` | `d.{SERIAL}.{random}` | Device has no valid credential session | Subscribe and PUT requests |
| `X-nl-device-id` | `{SERIAL}` (bare serial) | Device has no valid credential session | Entry (frontdoor) requests |

Both headers identify the device by serial number. `X-nl-device-id` is the bare serial. `X-nl-client-id` requires parsing — extract the second dot-delimited segment to get the serial (e.g., `d.09AA01AB12345678.BC7C9039` → `09AA01AB12345678`).

### Credential types

The device has two credential modes:

| Type | Description |
|------|-------------|
| Default | Built-in fallback credentials the device uses before your server provisions new ones. |
| Assigned | Credentials your server provisions through the `X-nl-set-client-credentials` header. |

The device uses assigned credentials for HTTP Basic Authentication once provisioned, and falls back to default credentials after an authentication failure.

### Credential provisioning

To provision credentials, include the `X-nl-set-client-credentials` header in any 200 response:

```http
X-nl-set-client-credentials: userid password
```

The format is space-separated `userid` and `password`. The device stores these values and uses them for HTTP Basic Authentication on subsequent requests.

> **Warning**: Provisioning credentials through a 401 response can cause a credential loop on some firmware versions. The device receives the new credentials, then immediately falls back to default credentials before using them. This results in the device cycling between default and assigned credentials indefinitely. If you provision credentials, do so in 200 responses rather than 401 responses.

### Recommended approach for home servers

For home server deployments, the simplest and most reliable approach is to skip credential provisioning entirely:

1. Accept all subscribe and put requests regardless of credentials.
2. Identify devices by the `X-nl-client-id` or `X-nl-device-id` header.
3. Look up the device serial in your database.

This avoids credential management complexity and the credential loop described above.

| Endpoint | Authentication required |
|----------|------------------------|
| `POST /entry` | No |
| `POST /{czid}/subscribe` | No (identify by header) |
| `POST /{czid}/put` | No (identify by header) |
| `GET {passphrase_url}` | No |

### Entry key

During device pairing, users need an entry key displayed on the device screen. The device fetches this key from the `passphrase_url` endpoint specified in the `/entry` response.

The entry key is separate from HTTP authentication. It verifies physical access to the device during initial setup.

#### Request

```http
GET {passphrase_url} HTTP/1.1
X-NL-Device-ID: 09AA01AB12345678
```

#### Response

```json
{
  "value": "123ABCD",
  "expires": 1707148800000
}
```

| Field | Type | Description |
|-------|------|-------------|
| `value` | string | 7-character alphanumeric code displayed on the device screen. |
| `expires` | number | Expiration timestamp in milliseconds since Unix epoch. **Must be a JSON number, not a string.** |

> **Warning**: The `expires` field must be a JSON number (for example, `1707148800000`). If you send it as a string (for example, `"1707148800000"`), the device silently rejects the response and never displays the entry key. The device also requires the expiration to be at least 30 minutes in the future.

The device displays the entry key in `XXX-XXXX` format (for example, `123-ABCD`).

#### Implement the entry key endpoint

1. Generate a random 7-character alphanumeric code.
2. Associate the code with the device serial from the `X-NL-Device-ID` header.
3. Set an expiration time at least 30 minutes in the future (recommended: 1 hour).
4. Return the code in the JSON response. Ensure `expires` is a JSON number.
5. Validate the code when the user enters it in your application during pairing.

> **Note**: The device polls this endpoint repeatedly until pairing completes. Return the same unexpired key on each request rather than generating a new one, otherwise the key is invalidated before the user can enter it.

---

## Pairing

After the user enters the entry key in your application, the server must complete pairing on the device. This dismisses the setup screen and transitions the device to normal operation.

### Pairing flow

```
User                  Server                          Device
  |                      |                               |
  |                      |   [Device fetches entry key]  |
  |                      |<--- GET {passphrase_url} -----|
  |                      |--- {"value":"123ABCD"} ------>|
  |                      |                               |
  |   [User reads code   |                [Device shows  |
  |    from device]      |                 "123-ABCD"]   |
  |                      |                               |
  |-- Enter code ------->|                               |
  |                      |   [Server claims code,        |
  |                      |    creates ownership record]  |
  |                      |                               |
  |                      |--- Push user bucket --------->|  (triggers pairing completion)
  |                      |--- Push structure bucket ----->|  (establishes device-home link)
  |                      |                               |
  |                      |          [Pairing dialog      |
  |                      |           dismisses]          |
  |                      |                               |
  |<-- "Paired!" --------|                               |
```

### Complete pairing on the device

To dismiss the pairing screen, push two buckets to the device through the subscribe connection:

1. **User bucket** — This is the critical piece. When the device receives a user bucket with a `name` field, it records the value as its pairing token and triggers pairing completion internally. Without this bucket, the pairing screen remains visible.

2. **Structure bucket** — This establishes the association between the device and a home/structure. The device uses it for home-level settings like away mode.

Push both buckets together in a single subscribe response:

```json
{
  "objects": [
    {
      "object_revision": 1,
      "object_timestamp": 1707148800000,
      "object_key": "user.your_user_id",
      "value": {
        "name": "your_user_id"
      }
    },
    {
      "object_revision": 1,
      "object_timestamp": 1707148800000,
      "object_key": "structure.your_structure_id",
      "value": {
        "name": "Home",
        "devices": ["09AA01AB12345678"]
      }
    }
  ]
}
```

> **Important**: The user bucket's `name` field is what completes pairing. The structure bucket alone is not sufficient — the device checks for an existing pairing token before processing structure updates, and that token doesn't exist until the user bucket provides it.

### Maintain pairing state across reconnections

Include both the user bucket and structure bucket in subscribe responses for paired devices on every reconnection, not just at registration time. The device may reboot, lose state, or reconnect to a freshly started server. Consistently including these buckets ensures pairing state is always current.

When the device already has the latest version of a bucket (matching timestamp), it ignores the duplicate. There is no penalty for including them.

> **Note**: After a fresh server start or device reboot, the device may only send partial state updates. Reboot the device after starting a new server for the first time to force a full state sync. During a reboot, the device re-initializes all of its data fields and uploads its complete state.

---

## Connection lifecycle

### Normal operation

```
Device                              Server
  |                                   |
  |-- POST /subscribe --------------->|
  |                                   |
  |<-- 200 OK (chunked) --------------|
  |    X-nl-suspend-time-max: 600     |
  |                                   |
  |   [Device may sleep any time]     |
  |   [TCP connection stays open]     |
  |                                   |
  |        ... time passes ...        |
  |                                   |
  |<-- JSON body ---------------------|  Server pushes update
  |                                   |
  |   [Device wakes in ~100-500ms]    |
  |   [Processes update]              |
  |                                   |
  |-- POST /subscribe --------------->|  Device resubscribes
```

### Server push wake mechanism

The device can sleep while maintaining the TCP connection. The WiFi hardware:

1. Keeps the TCP socket open
2. Sends periodic TCP keep-alive probes
3. Monitors for incoming data
4. Wakes the main CPU when data arrives

**From your server's perspective**:
- The connection appears idle but remains open.
- Your TCP stack automatically responds to keep-alive probes.
- When you send data, the device wakes within ~100–500 ms.
- No special action required—just send your chunked body.

> **Important**: For WoWLAN to work, your server URL must include an explicit port. See [Appendix: URL port requirement](#appendix-url-port-requirement).

### Connection timing

After sending chunked headers, the device may sleep at any moment. The connection remains open and you can push data whenever ready.

| Event | Server Observes |
|-------|-----------------|
| Device sleeps | Nothing. Connection stays open. |
| You push data | Device wakes, processes data, resubscribes |
| Suspend time expires | Device sends RST, then reconnects |
| Network failure | TCP timeout, then device reconnects |

### Service tickle (administrative only)

Sending an empty body (0-byte chunk) forces the device to reconnect immediately.

```http
0\r\n
\r\n
```

**Valid use cases**:
- Server graceful shutdown
- Load balancer migration
- Force state refresh

**Don't use for**:
- "No updates available" (just hold the connection open)
- "Keep device awake" (device sleeps anyway after reconnect)
- Periodic heartbeats (TCP keep-alives handle this)

---

## Server implementation notes

### Session ID behavior

The `session` field in subscribe requests is a device-scoped identifier, not a per-request identifier. The device reuses the same session value for all subscribe requests during its operational lifetime.

**Important:** Don't use the session ID as a unique key for tracking subscriptions server-side. Instead, generate a unique identifier for each subscription request.

### Overlapping subscriptions

When a device wakes early (for example, due to user interaction), it may send a new subscribe request while the server is still holding the previous connection. Both connections are valid simultaneously until the older one times out.

To handle this correctly:

1. Track each subscription independently using a server-generated identifier.
2. When pushing data, send to all active subscriptions for the device.
3. When a connection times out, remove only that specific subscription.

```
Device                              Server
  │                                   │
  │── POST /subscribe ───────────────>│  Create subscription A
  │<── 200 OK (chunked) ──────────────│
  │                                   │
  │   [Device wakes early]            │
  │                                   │
  │── POST /subscribe ───────────────>│  Create subscription B
  │<── 200 OK (chunked) ──────────────│  (same session, different subscription)
  │                                   │
  │   [Subscription A times out]      │  Remove A only, B remains active
  │                                   │
  │<── Push data on B ────────────────│  Works correctly
```

### Batching multiple pushes

When your server has data to push on a subscribe connection, you can batch multiple pushes into a single connection instead of closing after each one. This avoids forcing the device through a full resubscribe cycle for every intermediate change — useful when a user makes rapid adjustments (e.g., clicking +/- repeatedly).

**How it works**:

1. Send the first chunk immediately when data is available.
2. Hold the connection open briefly (recommend 3 seconds, must be under 5 seconds) waiting for additional data.
3. If more data arrives within the batch window, send it as another chunk on the same connection.
4. Repeat until no new data arrives within the window.
5. Close the connection after the batch window expires with no new data.

Each chunk must be a complete `{"objects": [...]}` JSON document. The device parses each chunk independently. Its 5-second closing timer resets on every chunk received, so the connection remains valid as long as you keep sending data within that window.

**Tradeoff**: Single-item pushes now wait 3 seconds before closing, in case more data arrives. The device stays awake for those extra seconds. This is negligible for wall-powered devices and acceptable for battery-powered devices given the 600-second suspend cycle.

#### Batched flow vs single-push flow

```
Single-push (without batching):

User clicks +1°     +1°     +1°
    |                 |       |
Server               |       |
    |── chunk 1 ─────>|       |
    |── close ────────>|       |
    |                 |       |
    |   [device resubscribes] |
    |                 |       |
    |── chunk 2 ──────────────>|
    |── close ─────────────────>|
    |                 |       |
    |   [device resubscribes] |
    |                 |       |
    |── chunk 3 ───────────────────>
    |── close ──────────────────────>
    = 3 subscribe cycles


Batched (with 3s batch window):

User clicks +1°     +1°     +1°
    |                 |       |
Server               |       |
    |── chunk 1 ─────>|       |
    |   [wait 3s...]  |       |
    |── chunk 2 ──────────────>|
    |   [wait 3s...]  |       |
    |── chunk 3 ───────────────────>
    |   [wait 3s, no more data]
    |── close ──────────────────────>
    = 1 subscribe cycle
```

#### Relevant timing constraints

| Timer | Duration | Source |
|-------|----------|--------|
| Device closing window | 5 seconds | Resets on each chunk received |
| Recommended batch window | 3 seconds | Server-side, must be < 5s |
| Update pending | 3 seconds | Device-internal processing window |

See the [Timing reference](#timing-reference) section for the full timing table.

---

## Versioning and synchronization

Each bucket maintains three fields that control synchronization between device and server:

| Field | Type | Description |
|-------|------|-------------|
| `object_revision` | int32 | Monotonically increasing counter, incremented on each write |
| `object_timestamp` | int64 | Milliseconds since Unix epoch (sole authority for sync decisions) |
| `value` | object | The actual bucket data (e.g., `target_temperature`) |

### Sync decision rules

**Timestamp is the primary authority** for determining which data is newer. The device compares timestamps when deciding whether to accept server updates or keep local state.

| Condition | Result |
|-----------|--------|
| Server timestamp > Device timestamp | Accept server data |
| Server timestamp < Device timestamp | Keep local data (server data is stale) |
| Server timestamp = Device timestamp | Use revision as tiebreaker (see below) |
| Server timestamp = 0 | Sentinel: "no data exists" - device should send its state |

**Revision Tiebreaker**:
When timestamps are exactly equal, the device uses revision as a tiebreaker:

| Condition (when timestamps equal) | Result |
|-----------------------------------|--------|
| Server revision > Device revision | Accept server data |
| Server revision <= Device revision | Keep local data (stale) |

Note: This is a **strict** greater-than comparison. If both timestamp and revision are equal, the server data is considered stale and rejected.

### Zero timestamp sentinel

A timestamp of `0` is a special sentinel value meaning "no data exists on server." When the device receives `object_timestamp: 0`:

- The server is signaling it has no state for this bucket
- The device should upload its current local state via PUT
- This is commonly seen after device reset or server-side data deletion

### Revision vs. timestamp

**Revision** (`object_revision`):
- Incremented on every write operation
- Used **only** for conditional writes (`if_object_revision`)
- If `if_object_revision` is provided in a PUT request and doesn't match the server's current revision, the server should reject the write (the specific error response is implementation-defined)
- `base_object_revision` in PUT requests is informational only (no validation)

**Timestamp** (`object_timestamp`):
- 64-bit milliseconds since Unix epoch
- **Sole authority** for sync decisions
- Determines "which version is newer"
- Zero = sentinel for "no data"

### Summary table

| Field | Purpose | Used For |
|-------|---------|----------|
| `object_revision` | Write ordering | Conditional writes (reject on mismatch) |
| `object_timestamp` | Sync authority | Determining newer data |
| `if_object_revision` | Conditional write guard | PUT validation |
| `base_object_revision` | Informational | Debugging/logging (no validation) |

---

## Response headers

### X-nl-suspend-time-max

Sets the maximum time (in seconds) before the device will wake and reconnect.

#### Syntax

```http
X-nl-suspend-time-max: 600
```

#### Value

Integer representing seconds. Recommended range: 300-900.

#### Description

This header controls the **fallback wake timer**—the maximum time a device sleeps before reconnecting even if you don't push data. It's **not** your primary latency control.

**Key insight**: When you push data to a sleeping device, it wakes in approximately 100–500 ms regardless of this value. The suspend time only determines how long until the device reconnects if you push nothing.

| Value | Use Case |
|-------|----------|
| 300 | Unreliable network; faster recovery from connection drops |
| **600** | **Recommended**. Good balance of battery life and reliability. |
| 900 | Maximum battery life; highly reliable network |

#### Example

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 600
```

---

### X-nl-defer-device-window

Controls how long the device delays sending local changes (e.g., user turning the dial).

#### Syntax

```http
X-nl-defer-device-window: 30
```

#### Value

Integer representing seconds. Maximum: 3599 (values ≥3600 are rejected).

#### Description

When a user adjusts the thermostat dial, the device delays sending the update to batch rapid changes. The actual delay is randomized within the window you specify.

| Value | Effect |
|-------|--------|
| 0 | Disabled. Every change sends immediately. |
| 15-30 | **Recommended**. Captures dial adjustments without feeling laggy. |
| 60+ | Very conservative. May feel unresponsive. |

#### Example

User turns dial: 70°F → 71°F → 72°F → 73°F → 72°F (settles)

- **Without defer**: 5 PUT requests sent
- **With defer (30s)**: 1 PUT request sent (final value only)

#### When defer is automatically disabled

The device bypasses the defer delay in these situations:

| Condition | Reason |
|-----------|--------|
| Mobile app recently connected | Ensures app sees changes immediately |
| `X-nl-disable-defer-window` header set | Server explicitly requested immediate updates |
| Device clock invalid | Cannot calculate delay reliably |

When you push a temperature change from your server, set `X-nl-disable-defer-window` to ensure the device's acknowledgment arrives promptly.

---

### X-nl-disable-defer-window

Temporarily disables the defer delay after pushing an update.

#### Syntax

```http
X-nl-disable-defer-window: 60
```

#### Value

Integer representing seconds. Maximum: 3599 (values ≥3600 are rejected).

#### Description

When you push a temperature change to the device, you typically want immediate confirmation. This header suppresses the defer delay for the specified duration, ensuring the device sends its acknowledgment PUT immediately.

#### Example

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 600
X-nl-service-timestamp: 1707148800000
X-nl-disable-defer-window: 60

{"objects": [{"object_revision": 458, "object_timestamp": 1707148800000, "object_key": "shared.09AA01AB12345678", "value": {"target_temperature": 22.0, "target_change_pending": true}}]}
```

For the next 60 seconds, any local changes also send immediately.

---

### X-nl-service-timestamp

Server's current time for clock synchronization.

#### Syntax

```http
X-nl-service-timestamp: 1707148800000
```

#### Value

64-bit integer representing milliseconds since Unix epoch (January 1, 1970 00:00:00 UTC).

#### Description

This header provides the server's authoritative timestamp for clock synchronization. The device uses this value to:

1. **Sync its internal clock**: If the device detects significant clock skew, it can correct its local time
2. **Validate timestamp comparisons**: Ensures the device's timestamp comparisons against `object_timestamp` values are meaningful

**Clock Skew Correction**: The device automatically corrects its clock if the skew between device time and server time exceeds approximately 10 minutes. This prevents sync issues where stale data appears newer due to clock drift.

#### Example

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 600
X-nl-service-timestamp: 1707148800000

{"shared": {"object_revision": 458, "object_timestamp": 1707148800000, "object_key": "shared.SERIAL", "value": {"target_temperature": 72.0}}}
```

#### Recommendations

- Always include this header in subscribe responses.
- Use your server's current system time in milliseconds.
- Ensure your server's clock is synchronized with NTP.

---

## Timing reference

### Device-side timers

These timers are internal to the device and cannot be controlled by the server.

| Timer | Duration | Trigger | Server Impact |
|-------|----------|---------|---------------|
| Closing window | 5 seconds | After receiving body data | Device stays awake briefly to complete processing. Each chunk resets this timer. |
| Immediate timeout | 7 seconds | Non-chunked response | Applies only when server responds without `Transfer-Encoding: chunked`. |
| Idle timeout | 50 seconds | No activity, non-chunked mode | Not applicable when using chunked encoding. |
| Update pending | 3 seconds | After receiving data | When streaming multiple chunks, send them within 3s of each other. |

**Closing window behavior**: When body data arrives, the 5-second timer starts. Each additional chunk resets this timer. If you're streaming multiple updates, send them within 5 seconds of each other to keep the device awake.

**Immediate timeout (7 seconds)**: If the server responds without `Transfer-Encoding: chunked`, the device expects the complete response body within 7 seconds. Use chunked encoding for long-poll connections to avoid this timeout.

### Recommended server configuration

| Setting | Value | Rationale |
|---------|-------|-----------|
| `X-nl-suspend-time-max` | 600 | Balance of battery and reliability |
| Server idle timeout | 900+ seconds | Must exceed suspend time |
| Server TCP keep-alive | Disabled | Device handles keep-alives |
| Response encoding | Chunked | Required for server push |

### Concurrent PUT and subscribe

The device may send a PUT while your subscribe response is in-flight. The device handles this internally by deferring subscribe data processing until the PUT completes.

**What this means for your server**:

- Accept PUTs at any time, even while holding a subscribe connection open.
- After accepting a PUT, your next subscribe response should reflect the updated state.
- If the subscribe response contains data older than the PUT, the device ignores it.

**Timing**:

```
Device                              Server
  |                                   |
  |-- POST /subscribe --------------->|
  |<-- 200 OK (chunked) --------------|
  |                                   |
  |   [User adjusts thermostat]       |
  |                                   |
  |-- POST /put --------------------->|  Device sends change
  |                                   |
  |<-- {"shared": ...} ---------------|  Subscribe data arrives
  |   [Device defers this data]       |
  |                                   |
  |<-- 200 OK ------------------------|  PUT acknowledged
  |   [Device processes deferred data]|
  |   [Keeps data only if newer]      |
```

The device compares timestamps to decide whether to keep its PUT data or accept the subscribe data. Your server doesn't need to coordinate these—just ensure your data has accurate timestamps.

---

## Battery behavior

### Low battery effects

| Voltage | Effect |
|---------|--------|
| 3.8V+ | Normal operation |
| 3.7V | +25 seconds added to sleep duration |
| 3.65V | Low battery flag set |
| 3.6V | **WiFi disabled**. Device goes offline. |
| 3.5V | +225 seconds added to sleep duration |

### Offline recovery

When battery drops below 3.6V, the device goes completely offline. It reconnects only after voltage recovers above 3.8V.

**Recommendation**: Queue updates for offline devices and deliver them when the device reconnects.

---

## Bucket types

State is organized into named buckets. Each bucket has an `object_key` (e.g., `shared.09AB1234`), `object_revision`, and `object_timestamp`.

### shared bucket

Use this bucket to control temperature settings.

| Field | Type | Writable | Description |
|-------|------|----------|-------------|
| `target_temperature` | float | Yes | Target temperature (Celsius) |
| `target_temperature_high` | float | Yes | Upper bound for range mode (Celsius) |
| `target_temperature_low` | float | Yes | Lower bound for range mode (Celsius) |
| `target_temperature_type` | string | Yes | `heat`, `cool`, `range`, or `off` |
| `target_change_pending` | boolean | Yes | Signals a pending temperature change. See [Display wake behavior](#display-wake-behavior). |
| `fan_timer_timeout` | integer | Yes | Fan timer end time (Unix timestamp) |
| `fan_timer_duration` | integer | Yes | Fan timer duration (seconds) |

**Writable** fields can be pushed via subscribe. The device also sends these fields in PUT requests when changed locally.

#### Display wake behavior

When you push a temperature change to a sleeping device, the device wakes and applies the new setpoint. However, the physical display remains off unless you also set `target_change_pending: true`.

**Set `target_change_pending: true` when changing:**

- `target_temperature`
- `target_temperature_high`
- `target_temperature_low`

**Don't set `target_change_pending` when changing:**

- `target_temperature_type` (mode changes have separate display handling)
- `away` (away state changes trigger their own display updates)
- Fan settings
- Any field in the `device` or `structure` buckets

#### How it works

1. Your server pushes `target_temperature` and `target_change_pending: true` together.
2. The device wakes, applies the temperature, and lights the display.
3. The display shows an animation of the temperature changing.
4. The device sends a PUT with `target_change_pending: false` to acknowledge.
5. Your server accepts this acknowledgment to avoid sync loops.

#### Example

```json
{
  "objects": [
    {
      "object_revision": 458,
      "object_timestamp": 1707148800000,
      "object_key": "shared.09AA01AB12345678",
      "value": {
        "target_temperature": 21.5,
        "target_change_pending": true
      }
    }
  ]
}
```

#### Handling the acknowledgment

When the device sends `target_change_pending: false` in a PUT request, accept this value even if your server still has `true`. This prevents an update loop where the server keeps pushing `true` and the device keeps clearing it.

```
Server                              Device
  |                                   |
  |-- target_change_pending: true --->|  Push temperature change
  |                                   |
  |   [Display lights up]             |
  |   [Shows temperature animation]   |
  |                                   |
  |<-- target_change_pending: false --|  Device acknowledges
  |                                   |
  |   [Server accepts false]          |  Don't push true again
```

### device bucket

Use this bucket for device configuration and to read current sensor values.

| Field | Type | Writable | Description |
|-------|------|----------|-------------|
| `current_temperature` | float | No | Current measured temperature (Celsius) |
| `current_humidity` | float | No | Current measured humidity (percent) |
| `away_temperature_high` | float | Yes | Upper eco/away temperature (Celsius) |
| `away_temperature_low` | float | Yes | Lower eco/away temperature (Celsius) |
| `temperature_scale` | string | Yes | Display units: `F` or `C` |
| `can_heat` | boolean | No | Heating capability |
| `can_cool` | boolean | No | Cooling capability |
| `has_fan` | boolean | No | Fan control capability |
| `hvac_ac_state` | boolean | No | AC running |
| `hvac_heater_state` | boolean | No | Heater running |
| `hvac_fan_state` | boolean | No | Fan running |

### user bucket

Use this bucket to complete pairing. See [Pairing](#pairing) for details.

| Field | Type | Writable | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | User identifier. Triggers pairing completion on the device. |

### structure bucket

Use this bucket for home-level settings and device-structure association.

| Field | Type | Writable | Description |
|-------|------|----------|-------------|
| `manual_eco_all` | boolean | Yes | Eco mode. Set to `true` to activate eco mode. See [Home/Away mode](#homeaway-mode). |
| `manual_eco_timestamp` | integer | Yes | Unix timestamp (seconds) for eco mode change. Must be within 600s of device clock. |
| `away` | boolean | Yes | Legacy away field. Do **not** use for eco control — overridden by schedule preconditioning. |
| `name` | string | Yes | Structure name |
| `devices` | array | Yes | List of device serials belonging to this structure |
| `postal_code` | string | Yes | Postal/ZIP code |
| `time_zone` | string | Yes | Timezone identifier |

### Other buckets

| Bucket | Purpose |
|--------|---------|
| `schedule` | Heating/cooling schedules |
| `where` | Room/location assignments |
| `message` | In-app messages |

### device_alert_dialog bucket

Use this bucket for server-initiated dialog prompts displayed on the device.

| Field | Type | Description |
|-------|------|-------------|
| `dialog_id` | string | Dialog type identifier |
| `dialog_data` | string | Dialog payload data |
| `msg_id` | string | Message identifier |
| `active` | boolean | Whether the dialog is active |
| `choices` | string | User choice options |
| `callback_url` | string | Response callback URL |
| `timeout` | integer | Dialog timeout in seconds |

To display a dialog, push a `dialog_id` value. To dismiss the dialog, clear or empty `dialog_id`.

---

## Home/Away mode

Eco mode is controlled through the **structure bucket** using the `manual_eco_all` and `manual_eco_timestamp` fields. When the device receives `manual_eco_all` set to `true`, it switches to eco mode and uses eco temperatures. Setting it to `false` restores normal schedule operation.

### Set away mode

Push a structure bucket with `manual_eco_all` and `manual_eco_timestamp` in the subscribe response:

```json
{
  "objects": [
    {
      "object_revision": 5,
      "object_timestamp": 1707148800000,
      "object_key": "structure.your_structure_id",
      "value": {
        "manual_eco_all": true,
        "manual_eco_timestamp": 1707148800
      }
    }
  ]
}
```

The device applies eco temperatures automatically. It uses the `away_temperature_high` and `away_temperature_low` fields from the device bucket as eco setpoints.

### Set home mode

Push `manual_eco_all: false` the same way:

```json
{
  "objects": [
    {
      "object_revision": 6,
      "object_timestamp": 1707148801000,
      "object_key": "structure.your_structure_id",
      "value": {
        "manual_eco_all": false,
        "manual_eco_timestamp": 1707148801
      }
    }
  ]
}
```

The device restores its normal target temperature and schedule.

### Timestamp requirement

The `manual_eco_timestamp` field must be within **600 seconds** of the device's internal clock. If the timestamp is stale, the device silently ignores the eco mode change — no error is returned.

Use current Unix time in **seconds** (not milliseconds) when pushing eco mode changes. Ensure your server's clock is NTP-synced to avoid timestamp drift.

### Why not the `away` field?

The structure bucket also has an `away` field, but don't use it for eco mode control. The `away` field triggers a different internal eco mode that the device's schedule preconditioning overrides within seconds. The `manual_eco_all` field triggers a persistent eco mode that the device respects through schedule changes.

Do **not** push both `away` and `manual_eco_all` simultaneously — they conflict internally and produce unpredictable results.

### Deliver the structure bucket

The device doesn't explicitly subscribe to the structure bucket. Your server must include it as an additional object in the subscribe response — the same mechanism used for [pairing](#pairing).

Once the device receives a structure bucket, it remembers the structure key and includes it in subsequent subscribe requests. From that point on, the device expects structure updates through the normal subscribe flow.

For the initial delivery (before the device knows about the structure), include the structure bucket alongside any other updates you push.

### Choose the structure key

| Scenario | Structure key | Example |
|----------|---------------|---------|
| Device has an owner (claimed) | Derive from user/owner identity | `structure.user123` |
| Device has no owner (unclaimed) | Use `structure.default` | `structure.default` |

For unclaimed devices, use `structure.default` as the structure key. The device processes it the same way regardless of the key name — what matters is that the bucket type is `structure`.

### Device bucket away fields (read-only)

The device bucket contains several away-related fields. These reflect the device's **own sensors and state** — writing to them from the server has no effect on eco mode.

| Field | Type | Direction | Description |
|-------|------|-----------|-------------|
| `auto_away` | boolean | Device → Server | Occupancy sensor output. `true` when no presence detected. |
| `auto_away_enable` | boolean | Device → Server | Whether the occupancy sensor is active. |
| `auto_away_reset` | boolean | Device → Server | Reset flag for auto-away state. |
| `home_away_input` | boolean | Device → Server | Physical presence detection input. |
| `away_temperature_high` | float | Bidirectional | Upper eco temperature (Celsius). Device-writable. |
| `away_temperature_low` | float | Bidirectional | Lower eco temperature (Celsius). Device-writable. |

> **Warning**: Don't write `auto_away` or `away` to the device bucket expecting to control eco mode. The device ignores server-pushed values for these fields in the device bucket. Use the structure bucket's `manual_eco_all` field instead.

### Eco temperatures

When in eco mode, the device uses `away_temperature_high` and `away_temperature_low` from the device bucket as its setpoints. These are set by the device itself (based on user preferences) and can also be pushed from the server. They are separate from the normal `target_temperature` fields in the shared bucket.

### Exiting eco mode

To exit eco mode, push `manual_eco_all: false` with a fresh `manual_eco_timestamp`. The device returns to its normal schedule and target temperature.

The user can also exit eco mode by physically turning the dial on the thermostat. The device's occupancy sensor does **not** automatically exit manual eco mode — physical interaction or a server push is required.

---

## Error handling

### HTTP status codes

| Status | When to Send | Device Behavior |
|--------|--------------|-----------------|
| 200 | Success | Process response |
| 400 | Malformed request | Log error, retry up to 2 times (3 total attempts) |
| 401 | Authentication failed | Re-authenticate |
| 403 | Access denied | Log error, reset comms state |
| 404 | Unknown device/path | Log error, reset comms state |
| 5xx | Server error | Retry with exponential backoff |

### Connection errors

| Error | Device Behavior | Server Observes |
|-------|-----------------|-----------------|
| TCP timeout | Reconnect | Connection closes |
| TLS failure | Retry | Handshake fails |
| Keep-alive timeout | Reconnect | RST packet |
| Wake timer expiry | Abort and reconnect | RST packet |

**Note**: When the wake timer expires (`X-nl-suspend-time-max`), the device sends RST (not FIN). This is normal behavior, not an error.

**HTTP 400 Retry Behavior**: The device retries HTTP 400 errors up to 2 times (3 total attempts) before giving up.

> **TODO**: Document retry intervals and backoff strategy for 5xx errors from binary analysis.

---

## Implementation checklist

### Minimum requirements

- [ ] **[Include explicit port in server URL](#appendix-url-port-requirement)** (e.g., `:443`) — required for WoWLAN
- [ ] Implement `POST /entry` endpoint (returns `transport_url` with explicit port)
- [ ] Implement `POST /{czid}/subscribe` endpoint
- [ ] Return `Transfer-Encoding: chunked` on all subscribe responses
- [ ] Return `X-nl-suspend-time-max` header (recommend: 600)
- [ ] Keep connections open after sending headers
- [ ] Set server idle timeout higher than suspend time (e.g., 900s)
- [ ] Handle RST gracefully (device wake timer)
- [ ] Store revision (int32) and timestamp (int64) for each bucket
- [ ] Update both revision and timestamp atomically on every bucket write
- [ ] Use timestamp as primary sync authority (not revision)
- [ ] Treat timestamp=0 as "no data" sentinel
- [ ] Ensure `object_revision` and `object_timestamp` appear BEFORE `object_key` in JSON responses
- [ ] Include `X-nl-service-timestamp` header in responses (milliseconds since epoch)
- [ ] Implement entry key endpoint with `expires` as a JSON number (not string)
- [ ] Push user bucket (with `name` field) to complete pairing
- [ ] Push structure bucket to establish device-home association
- [ ] Include `manual_eco_timestamp` as Unix seconds (not milliseconds) when pushing eco mode changes
- [ ] Include user and structure buckets on every subscribe for paired devices

### Recommended

- [ ] Push updates immediately when available
- [ ] Batch rapid pushes on a single subscribe connection (hold ≤3s between chunks)
- [ ] Set `X-nl-defer-device-window: 15-30` for setpoint jitter
- [ ] Set `X-nl-disable-defer-window: 60` when pushing temperature changes
- [ ] Set `target_change_pending: true` when pushing temperature changes (wakes display)
- [ ] Accept `target_change_pending: false` from device without re-pushing `true`
- [ ] Queue updates for offline devices
- [ ] Log device connections for debugging
- [ ] Use `base_object_revision` from PUT requests for conflict detection
- [ ] Accept PUTs at any time, even while subscribe connections are open

### Avoid

- [ ] Closing connections after sending headers
- [ ] Using short `X-nl-suspend-time-max` for "responsiveness"
- [ ] Sending server-side TCP keep-alives
- [ ] Using service tickles for normal operation
- [ ] Setting aggressive idle timeouts
- [ ] Sending `object_key` before `object_revision`/`object_timestamp` in JSON responses
- [ ] Using revision for sync decisions (use timestamp instead)
- [ ] Sending entry key `expires` as a JSON string (must be a number)
- [ ] Provisioning credentials through 401 responses (can cause credential loop)
- [ ] Using the structure bucket `away` field for eco mode control (overridden by schedule preconditioning)

---

## Examples

> **IMPORTANT: Field Order Requirement**
>
> The device parser expects `object_revision` and `object_timestamp` to appear BEFORE `object_key` in JSON responses.
> Incorrect field ordering can cause parsing failures or sync issues.

### Basic subscribe response

No updates available—hold connection open:

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 600
X-nl-service-timestamp: 1707148800000

```
(Connection held open, no body sent yet)

### Push temperature update

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 600
X-nl-service-timestamp: 1707148800000
X-nl-disable-defer-window: 60

{"objects": [{"object_revision": 458, "object_timestamp": 1707148800000, "object_key": "shared.09AA01AB12345678", "value": {"target_temperature": 21.5, "target_change_pending": true}}]}
```

Note: `object_revision` and `object_timestamp` appear before `object_key` in the JSON. Data fields are nested in `value`. Include `target_change_pending: true` to wake the display.

### Subscribe response with full objects array

When multiple buckets have updates, use the `objects` array format:

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 600
X-nl-service-timestamp: 1707148800000

{
  "objects": [
    {
      "object_revision": 458,
      "object_timestamp": 1707148800000,
      "object_key": "shared.09AA01AB12345678",
      "value": {
        "target_temperature": 22.0,
        "target_change_pending": true
      }
    },
    {
      "object_revision": 123,
      "object_timestamp": 1707148799000,
      "object_key": "device.09AA01AB12345678",
      "value": {
        "current_temperature": 20.5
      }
    }
  ]
}
```

Note: Within each object, `object_revision` and `object_timestamp` MUST precede `object_key`. Include `target_change_pending: true` when pushing temperature changes.

### Graceful server shutdown (service tickle)

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked

0

```
(Empty body forces device to reconnect to different server)

---

## Appendix: URL port requirement

> **Always include an explicit port in your server URL.**

The device's URL parser fails to extract ports from URLs that omit them, breaking WoWLAN (WiFi wake-on-LAN) functionality.

### Problem

URLs without explicit ports cause silent parsing failures:

```
http://your-server.example.com/path    ← WRONG: no port
http://your-server.example.com:80/path ← CORRECT: explicit port
```

The device falls back to a stale cached port value, causing TCP keepalive offload to be configured incorrectly. Result: **server push cannot wake a sleeping device**.

### Symptoms

- Device works normally while awake
- Server push fails to wake sleeping device
- Device logs show packet size of 0 instead of ~40 bytes

### Solution

Always include the port, even for standard ports:

| Protocol | URL Format |
|----------|------------|
| HTTP | `http://example.com:80/path` |
| HTTPS | `https://example.com:443/path` |

### Verification

```bash
# Check that srcPort matches your server's port
grep "RegisterSniffer" /var/log/messages | tail -1

# Confirm keepalive configured (enable = 1, connected = 1)
grep "Configuring keep alive" /var/log/messages | tail -1
```

---

## Document status

| Section | Status |
|---------|--------|
| URL port requirement | Complete |
| Subscribe endpoint | Complete |
| PUT endpoint | Complete |
| Response headers | Complete |
| Connection lifecycle | Complete |
| Timing reference | Complete |
| Battery behavior | Complete |
| Versioning and synchronization | Complete |
| Entry endpoint | Complete |
| Authentication | Provisional |
| Pairing | Complete |
| Bucket schemas | Complete |
| Home/Away mode | Complete |
| Error handling | Partial |

---

## Changelog

| Revision | Date | Changes |
|----------|------|---------|
| 1.8 | 2026-02-07 | Added "Batching multiple pushes" section under Server implementation notes. Explains how to send multiple chunks on a single subscribe connection using the device's 5-second closing timer reset behavior. |
| 1.7 | 2026-02-07 | Corrected Home/Away mode section — use `manual_eco_all` + `manual_eco_timestamp` instead of `away` field. The `away` field triggers auto-eco which is overridden by schedule preconditioning. Added timestamp requirement (600s staleness window). Added guidance on field conflicts and exiting eco mode. |
| 1.6 | 2026-02-07 | Added Home/Away mode section: structure bucket controls away state, device bucket away fields are read-only, eco temperature behavior, structure key selection for claimed vs unclaimed devices. Enhanced structure bucket `away` field description. |
| 1.5 | 2026-02-07 | Documented `X-nl-client-id` header format (`d.{SERIAL}.{random}`). Added entry key polling note (server must return same unexpired key). Fixed broken anchor links for defer-device-window. Fixed `if_object_revision` contradiction (now consistently "implementation-defined"). Fixed subscribe example to use `objects` array format. |
| 1.4 | 2026-02-06 | Clarified 403/404 trigger comms reset (not just logging). |
| 1.3 | 2026-02-06 | Marked `X-nl-longest-wake` header as vestigial (server ignores, never resets). |
| 1.2 | 2026-02-05 | Added Authentication section (provisional) with device identification, credential types, credential loop warning, and recommended home server approach. Added Pairing section with user bucket mechanism. Fixed entry key `expires` type (must be JSON number, not string). Added user bucket to bucket types. Added device reboot note for full state sync. Expanded POST /entry response fields. Added device_alert_dialog bucket. |
| 1.1 | 2026-02-05 | Added `if_object_revision` conditional write documentation. Clarified closing timer reset behavior and 7-second timeout context. Added JSON library field ordering implementation note. |
| 1.0 | 2026-02-04 | Initial release. Subscribe, PUT, and entry endpoints. Response headers. Timing reference. Battery behavior. Bucket types. URL port requirement appendix. |
