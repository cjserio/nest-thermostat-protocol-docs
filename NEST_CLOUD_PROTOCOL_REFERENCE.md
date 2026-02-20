# Nest Cloud Protocol Reference

Protocol specification for implementing a server that communicates with Nest thermostat devices.

**Revision**: 2.7
**Last updated**: 2026-02-19
**Author**: Chris Serio

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
  - [Connection hold time](#connection-hold-time)
  - [Concurrent PUT and subscribe](#concurrent-put-and-subscribe)
- [Battery behavior](#battery-behavior)
- [Bucket types](#bucket-types)
  - [All bucket types](#all-bucket-types)
  - [device bucket](#device-bucket)
  - [shared bucket](#shared-bucket)
  - [structure bucket](#structure-bucket)
  - [user bucket](#user-bucket)
  - [schedule bucket](#schedule-bucket)
  - [device_alert_dialog bucket](#device_alert_dialog-bucket)
  - [hvac_partner bucket](#hvac_partner-bucket)
  - [topaz bucket](#topaz-bucket)
  - [kryptonite bucket](#kryptonite-bucket)
  - [Display wake behavior](#display-wake-behavior)
  - [Write protection](#write-protection)
  - [Schedule sync guards](#schedule-sync-guards)
- [Thermostat control](#thermostat-control)
  - [State model](#state-model)
  - [HVAC modes](#hvac-modes)
  - [Set the temperature](#set-the-temperature)
  - [Read device state](#read-device-state)
  - [Feature reference](#feature-reference)
  - [State interactions](#state-interactions)
- [Home/Away and eco mode](#homeaway-and-eco-mode)
  - [Enter eco mode](#enter-eco-mode)
  - [Exit eco mode](#exit-eco-mode)
  - [Eco temperatures](#eco-temperatures)
  - [The `away` field](#the-away-field)
  - [Read eco mode status](#read-eco-mode-status)
  - [Deliver the structure bucket](#deliver-the-structure-bucket)
  - [Choose the structure key](#choose-the-structure-key)
  - [Device bucket away fields](#device-bucket-away-fields)
  - [Exiting eco mode](#exiting-eco-mode)
- [Temperature schedules](#temperature-schedules)
  - [Schedule bucket](#schedule-bucket)
  - [Schedule JSON format](#schedule-json-format)
  - [Day indexing](#day-indexing)
  - [Setpoint fields](#setpoint-fields)
  - [Temperature encoding](#temperature-encoding)
  - [Schedule modes](#schedule-modes)
  - [Push a schedule to the device](#push-a-schedule-to-the-device)
  - [Read schedules from the device](#read-schedules-from-the-device)
  - [Switch schedule mode](#switch-schedule-mode)
  - [Continuation setpoints](#continuation-setpoints)
  - [Custom schedules](#custom-schedules)
  - [Schedule behavior details](#schedule-behavior-details)
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
| Keep-alive | WiFi hardware maintains connection during sleep |

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
X-nl-suspend-time-max: 300
X-nl-service-timestamp: 1707148800000

{"shared": {"object_revision": 457, "object_timestamp": 1707148800000, "object_key": "shared.09AA01AB12345678", "value": {"target_temperature": 22.00000}}}
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

Return `200 OK` to acknowledge receipt.

**The PUT response is a write receipt, not a data channel.** The server stores the device's fields, increments the revision, and returns `{object_revision, object_timestamp, object_key}`. No `value` field. Server-to-device data has exactly one path: subscribe.

The device has no per-key staleness protection. If you include a `value` field in the PUT response, the device applies every field in it as authoritative cloud data — the same code path as subscribe, no filtering, no comparison. Any field the device has updated locally since the PUT was sent (even milliseconds ago) will be silently overwritten. This includes conditional write (CAS) conflict responses: return the updated revision so the device can retry, but never include `value`.

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
  |    X-nl-suspend-time-max: 300     |
  |                                   |
  |   [Device may sleep any time]     |
  |   [TCP connection stays open]     |
  |                                   |
  |        ... time passes ...        |
  |                                   |
  |<-- JSON body ---------------------|  Server pushes update (optional)
  |                                   |
  |   [Device wakes in ~100-500ms]    |
  |   [Processes update]              |
  |                                   |

  ... OR if no data to push ...

  |                                   |
  |   [~290 seconds pass]            |
  |                                   |
  |<-- 0\r\n\r\n --------------------|  Server closes (hold time reached)
  |                                   |
  |   [Device wakes in ~100-500ms]    |
  |                                   |
  |-- POST /subscribe --------------->|  Device resubscribes
```

### Server push wake mechanism

The device can sleep while maintaining the TCP connection. The WiFi hardware:

1. Keeps the TCP socket open
2. Monitors for incoming data (WoWLAN)
3. Wakes the main CPU when data arrives

**From your server's perspective**:
- The connection appears idle but remains open.
- When you send data, the device wakes within ~100–500 ms.
- No special action required—just send your chunked body.
- Do **not** enable server-side `SO_KEEPALIVE`—the device does not respond to keep-alive probes while the CPU is asleep.

> **Caution**: If the connection remains idle for approximately 360 seconds, the device considers it dead and resubscribes on a new connection—without closing the old one. Your server will briefly see two active subscriptions for the same device. To avoid this, keep your connection hold time under 350 seconds. See [Connection hold time](#connection-hold-time).

> **Important**: For WoWLAN to work, your server URL must include an explicit port. See [Appendix: URL port requirement](#appendix-url-port-requirement).

### Connection timing

After sending chunked headers, the device may sleep at any moment. The connection remains open and you can push data whenever ready.

| Event | Server observes |
|-------|-----------------|
| Device sleeps | Nothing. Connection stays open. |
| You push data | Device wakes, processes data, resubscribes |
| You close the connection | Device wakes, resubscribes |
| Network failure | TCP timeout, then device reconnects |

**The server controls the subscribe cycle.** The device doesn't close the connection on its own during normal operation. To drive the reconnect cycle, close the connection by sending the final chunk terminator (`0\r\n\r\n`). The device wakes, processes the close, and resubscribes.

Your connection hold time (how long you keep the connection open before closing it) must be **shorter** than `X-nl-suspend-time-max`. See [Connection hold time](#connection-hold-time).

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
- Periodic heartbeats (not needed—the server drives the cycle by closing the connection)

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

**Tradeoff**: Single-item pushes now wait 3 seconds before closing, in case more data arrives. The device stays awake for those extra seconds. This is negligible for wall-powered devices and acceptable for battery-powered devices given the 300-second suspend cycle.

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
X-nl-suspend-time-max: 300
```

#### Value

Integer representing seconds. Recommended range: 250-350.

#### Description

This header controls the **fallback wake timer**—the maximum time a device sleeps before reconnecting even if you don't push data. It's **not** your primary latency control.

**Key insight**: When you push data to a sleeping device, it wakes in approximately 100–500 ms regardless of this value. The suspend time only determines how long until the device reconnects if you push nothing.

> **Critical**: Your server's connection hold time must be **shorter** than this value. The server closing the connection is the primary mechanism that drives the subscribe cycle. The device's wake timer is a safety net, not the driver. See [Connection hold time](#connection-hold-time).

| Value | Use case |
|-------|----------|
| 250 | Unreliable network; faster recovery from connection drops |
| **300** | **Recommended**. Good balance of battery life and reliability. |
| 350 | Maximum value. Must stay under ~360s idle connection timeout. |

#### Example

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 300
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
X-nl-suspend-time-max: 300
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
X-nl-suspend-time-max: 300
X-nl-service-timestamp: 1707148800000

{"shared": {"object_revision": 458, "object_timestamp": 1707148800000, "object_key": "shared.SERIAL", "value": {"target_temperature": 22.00000}}}
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

### Connection hold time

The connection hold time is how long your server keeps a subscribe connection open before closing it. This value **must be shorter** than `X-nl-suspend-time-max`.

The device maintains the TCP connection through CPU sleep using WiFi hardware offload. The device doesn't close the connection on its own during normal sleep cycles. Instead, the server drives the reconnect cycle by closing the connection (sending the final chunk terminator). When the server closes, the WiFi hardware wakes the CPU, and the device resubscribes.

Two constraints apply to the connection hold time:

1. **Must be shorter than `X-nl-suspend-time-max`**. Otherwise, the device's fallback wake timer fires first, causing unpredictable behavior.
2. **Must be shorter than approximately 350 seconds**. If the connection is idle for about 360 seconds during sleep, the device considers it dead, abandons it without closing, and resubscribes on a new connection. This creates overlapping subscriptions.

**Formula**: `connection_hold_time = X-nl-suspend-time-max - margin`

A margin of 10 seconds is sufficient. Keep `X-nl-suspend-time-max` at or below 350 seconds.

| `X-nl-suspend-time-max` | Connection hold time | Margin |
|--------------------------|---------------------|--------|
| **300** | **290** | **10** |
| 350 | 340 | 10 |

### Recommended server configuration

| Setting | Value | Rationale |
|---------|-------|-----------|
| `X-nl-suspend-time-max` | 300 | Must be less than 350 to stay under idle connection timeout |
| Connection hold time | 290 seconds | Must be shorter than suspend time (suspend - 10) |
| Server `SO_KEEPALIVE` | Disabled | Device does not respond to keep-alive probes while asleep |
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

State is organized into named data containers called buckets. Each bucket has an `object_key` (for example, `shared.09AB1234`), `object_revision`, and `object_timestamp`. For revision and timestamp semantics, see [Versioning and synchronization](#versioning-and-synchronization).

Bucket object keys follow the format `{type}.{identifier}`. The type prefix determines how the device processes the data. The identifier is usually the device serial number but varies by bucket type.

### All bucket types

The device recognizes 28 bucket types. Most server implementations only need to handle the essential ones.

| Bucket | Object key | Direction | Priority |
|--------|-----------|-----------|----------|
| `device` | `device.{serial}` | Bidirectional (restricted) | Essential |
| `shared` | `shared.{serial}` | Bidirectional | Essential |
| `structure` | `structure.{structureId}` | Server &rarr; device | Essential |
| `user` | `user.{userId}` | Server &rarr; device | Essential |
| `schedule` | `schedule.{serial}` | Bidirectional (guarded) | Essential |
| `where` | `where.{whereId}` | Bidirectional | Secondary |
| `message` | `message.{messageId}` | Bidirectional | Secondary |
| `link` | `link.{linkId}` | Server &rarr; device | Secondary |
| `custom_schedule` | `custom_schedule.{id}` | Bidirectional | Secondary |
| `device_alert_dialog` | `device_alert_dialog.{id}` | Server &rarr; device | Secondary |
| `hvac_partner` | `hvac_partner.{partnerId}` | Bidirectional | Specialized |
| `topaz` | `topaz.{topazId}` | Server &rarr; device | Specialized |
| `kryptonite` | `kryptonite.{sensorId}` | Bidirectional | Specialized |
| `servicegroup` | `servicegroup.{id}` | Server &rarr; device | Specialized |
| `occupancy` | `occupancy.{serial}` | Device &rarr; server | Specialized |
| `demand_response` | `demand_response.{id}` | Bidirectional | Specialized |
| `demand_response_event` | `demand_response_event.{eventId}` | Bidirectional | Specialized |
| `utility` | `utility.{id}` | Server &rarr; device | Specialized |
| `diamond_sensor_config` | `diamond_sensor_config.{id}` | Server &rarr; device | Specialized |
| `diamond_sensor_event` | `diamond_sensor_event.{id}` | Bidirectional | Specialized |
| `rate_plan` | `rate_plan.{id}` | Server &rarr; device | Specialized |
| `tou` | `tou.{id}` | Bidirectional | Specialized |
| `demand_charge` | `demand_charge.{id}` | Server &rarr; device | Specialized |
| `demand_charge_event` | `demand_charge_event.{eventId}` | Bidirectional | Specialized |
| `rcs_settings` | `rcs_settings.{id}` | Bidirectional | Specialized |
| `cloud_algo` | `cloud_algo.{id}` | Bidirectional | Specialized |
| `diagnostics` | `diagnostics.{id}` | Bidirectional | Specialized |
| `tuneups` | `tuneups.{id}` | Bidirectional | Specialized |

**Direction key:**

| Direction | Meaning |
|-----------|---------|
| Server &rarr; device | Server pushes data on subscribe. Device reads and applies it. Device never sends this bucket in PUT requests. |
| Device &rarr; server | Device sends data through PUT. Server stores it. |
| Bidirectional | Both sides read and write. Some bidirectional buckets have per-field restrictions (see [device bucket](#device-bucket)). |

### device bucket

The largest bucket. Contains device telemetry, configuration, hardware state, and sensor readings. The device sends ~198 fields on a full sync after boot.

**Object key**: `device.{serial}`
**Direction**: Bidirectional (with per-field restrictions)
**Revision type**: `base_object_revision` (unconditional)

#### Field access modes

Every field in the device bucket has one of three access modes that determine whether the server can write to it:

| Mode | Count | Server can write? | In PUT? | Description |
|------|-------|-------------------|---------|-------------|
| Device-only | ~113 | **No** — device rejects and overwrites. See [Write protection](#write-protection). | Yes | Hardware state, sensors, computed values |
| Special | ~29 | Varies | No | Custom processing paths (eco, HVAC capacities). Not in standard PUT. |
| Cloud-writable | ~97 | **Yes** | Yes | Configuration the server can push. |

#### Key read-only fields

These fields appear in PUT requests. Store them, but don't try to overwrite them — the device ignores or actively reverts server writes to these fields.

| Field | Type | Description |
|-------|------|-------------|
| `current_temperature` | float | Current measured temperature (Celsius) |
| `current_humidity` | integer | Current relative humidity (percent) |
| `backplate_temperature` | float | Backplate temperature (Celsius) |
| `battery_level` | float | Battery charge level |
| `has_fan` | boolean | Fan control available |
| `has_humidifier` | boolean | Humidifier detected |
| `has_dehumidifier` | boolean | Dehumidifier detected |
| `leaf` | boolean | Leaf icon displayed (energy-saving indicator) |
| `auto_away` | integer | Occupancy sensor state: `0` = home, `1` = away |
| `time_to_target` | integer | Estimated seconds to reach target temperature |
| `time_to_target_training` | string | Training status: `ready`, `training`, or `not_ready` |
| `error_code` | string | Active error code |
| `serial_number` | string | Device serial number |
| `software_version` | string | Firmware version |
| `model_version` | string | Hardware model |
| `local_ip` | string | Device IP on local network |
| `mac_address` | string | Wi-Fi MAC address |
| `is_online` | boolean | Device considers itself connected |

#### Cloud-writable fields

These fields accept server writes through subscribe responses. Push them inside the `value` object of a `device.*` bucket.

**Temperature settings:**

| Field | Type | Description |
|-------|------|-------------|
| `away_temperature_high` | float | Upper eco temperature (Celsius) |
| `away_temperature_high_enabled` | boolean | Enable upper eco temperature limit |
| `away_temperature_low` | float | Lower eco temperature (Celsius) |
| `away_temperature_low_enabled` | boolean | Enable lower eco temperature limit |
| `temperature_scale` | string | Display unit preference: `F` or `C` |
| `upper_safety_temp` | float | Upper safety limit (Celsius) |
| `upper_safety_temp_enabled` | boolean | Enable upper safety limit |
| `lower_safety_temp` | float | Lower safety limit (Celsius) |
| `lower_safety_temp_enabled` | boolean | Enable lower safety limit |

**Temperature lock:**

| Field | Type | Description |
|-------|------|-------------|
| `temperature_lock` | boolean | Enable temperature lock |
| `temperature_lock_high_temp` | float | Lock upper bound (Celsius) |
| `temperature_lock_low_temp` | float | Lock lower bound (Celsius) |
| `temperature_lock_pin_hash` | string | Lock PIN hash |

**Fan settings:**

| Field | Type | Description |
|-------|------|-------------|
| `fan_mode` | string | Fan operating mode |
| `fan_cooling_enabled` | boolean | Fan runs during cooling |
| `fan_duty_cycle` | integer | Fan duty cycle (percent) |
| `fan_duty_start_time` | integer | Fan schedule start (seconds from midnight) |
| `fan_duty_end_time` | integer | Fan schedule end (seconds from midnight) |
| `fan_heat_cool_speed` | string | Fan speed during HVAC operation |
| `fan_schedule_speed` | string | Fan speed during scheduled runs |
| `fan_timer_duration` | integer | Fan timer length (seconds) |
| `fan_timer_speed` | string | Fan timer speed |
| `fan_timer_timeout` | integer | Fan timer end time (Unix timestamp) |

**Eco/away:**

| Field | Type | Description |
|-------|------|-------------|
| `auto_away_enable` | boolean | Enable occupancy-based auto-away |
| `auto_away_reset` | boolean | Reset auto-away to home state |
| `home_away_input` | boolean | Enable home/away feature |

**Humidity control:**

| Field | Type | Description |
|-------|------|-------------|
| `auto_dehum_enabled` | boolean | Automatic dehumidification |
| `target_humidity` | float | Target humidity (percent) |
| `target_humidity_enabled` | boolean | Enable humidity targeting |
| `humidifier_type` | string | Humidifier type |
| `humidifier_fan_activation` | boolean | Fan activates with humidifier |
| `dehumidifier_type` | string | Dehumidifier type |
| `dehumidifier_fan_activation` | boolean | Fan activates with dehumidifier |
| `dehumidifier_orientation_selected` | string | Dehumidifier orientation |
| `humidity_control_lockout_enabled` | boolean | Enable humidity control lockout |
| `humidity_control_lockout_start_time` | integer | Lockout start (seconds from midnight) |
| `humidity_control_lockout_end_time` | integer | Lockout end (seconds from midnight) |

**Heat pump / dual fuel:**

| Field | Type | Description |
|-------|------|-------------|
| `dual_fuel_breakpoint` | float | Temperature below which aux heat activates (Celsius) |
| `dual_fuel_breakpoint_override` | string | Breakpoint override setting |
| `dual_fuel_selected` | boolean | Dual fuel mode active |
| `heat_pump_aux_threshold` | float | Aux heat lockout temperature (Celsius) |
| `heat_pump_aux_threshold_enabled` | boolean | Enable aux heat threshold |
| `heat_pump_comp_threshold` | float | Compressor lockout temperature (Celsius) |
| `heat_pump_comp_threshold_enabled` | boolean | Enable compressor threshold |
| `heatpump_savings` | string | Heat pump savings mode |

**Learning and scheduling:**

| Field | Type | Description |
|-------|------|-------------|
| `learning_mode` | boolean | Auto-schedule learning enabled |
| `schedule_learning_reset` | boolean | Reset learned schedule data |
| `preconditioning_enabled` | boolean | Start heating/cooling before scheduled setpoints |
| `max_nighttime_preconditioning_seconds` | integer | Max preconditioning duration at night |

**Hot water (EU/UK Heat Link systems):**

| Field | Type | Description |
|-------|------|-------------|
| `hot_water_away_enabled` | boolean | Hot water available during eco mode |
| `hot_water_boost_time_to_end` | integer | Hot water boost timer end (Unix timestamp) |
| `hot_water_mode` | string | Hot water operating mode |
| `hot_water_temperature` | float | Hot water target temperature (Celsius) |
| `heat_link_manual_mode` | boolean | Heat Link manual override |

**Device configuration:**

| Field | Type | Description |
|-------|------|-------------|
| `click_sound` | boolean | Audible click on dial turn |
| `farsight_screen` | string | Display content when approaching |
| `should_wake_on_approach` | boolean | Wake display on approach |
| `ob_orientation` | string | O/B wire orientation (heat pumps) |
| `ob_persistence` | string | O/B wire persistence |
| `radiant_control_enabled` | boolean | Radiant heating mode |
| `sunlight_correction_enabled` | boolean | Compensate for direct sunlight on sensor |
| `where_id` | string | Room assignment identifier |
| `pro_id` | string | Nest Pro installer identifier |

**Filter reminders:**

| Field | Type | Description |
|-------|------|-------------|
| `filter_changed_date` | integer | Last filter change (Unix timestamp) |
| `filter_changed_set_date` | integer | Date the change was recorded |
| `filter_reminder_enabled` | boolean | Enable filter change reminders |
| `filter_replacement_threshold_sec` | integer | Filter lifetime (seconds) |

**HVAC source/delivery** (wiring configuration):

`alt_heat_delivery`, `alt_heat_source`, `alt_heat_x2_delivery`, `alt_heat_x2_source`, `aux_heat_delivery`, `aux_heat_source`, `cooling_delivery`, `cooling_source`, `cooling_x2_delivery`, `cooling_x2_source`, `cooling_x3_delivery`, `cooling_x3_source`, `emer_heat_delivery`, `emer_heat_enable`, `emer_heat_source`, `forced_air`, `heater_delivery`, `heater_source`, `heat_x2_delivery`, `heat_x2_source`, `heat_x3_delivery`, `heat_x3_source`, `star_type`, `y2_type`

**Setup wizard state** (out-of-box):

`oob_interview_completed`, `oob_temp_completed`, `oob_test_completed`, `oob_startup_completed`, `oob_summary_completed`, `oob_where_completed`, `oob_wifi_completed`, `oob_wires_completed`

### shared bucket

Controls the active temperature setpoint and HVAC mode. This is the primary bucket for thermostat control.

**Object key**: `shared.{serial}`
**Direction**: Bidirectional
**Revision type**: `if_object_revision` (conditional write)

> **Important**: The `shared` bucket is the **only** bucket that uses conditional writes. When the device sends a PUT with `if_object_revision`, it expects the server to reject the write if the revision doesn't match. This prevents the device from overwriting a temperature change that the server pushed while the device was preparing its PUT. All other buckets use `base_object_revision` (unconditional). See [Versioning and synchronization](#versioning-and-synchronization).

#### Fields

| Field | Type | Server | Description |
|-------|------|--------|-------------|
| `target_temperature` | float | Read/write | Target temperature in Celsius |
| `target_temperature_high` | float | Read/write | Upper bound in range mode (Celsius) |
| `target_temperature_low` | float | Read/write | Lower bound in range mode (Celsius) |
| `target_temperature_type` | string | Read/write | HVAC mode: `heat`, `cool`, `range`, `emergency`, or `off`. See [HVAC modes](#hvac-modes). |
| `target_change_pending` | boolean | Read/write | Pending temperature change flag. See [Display wake behavior](#display-wake-behavior). |
| `touched_by` | object | Read | Metadata about the last temperature change source. See [Temperature change source tracking](#temperature-change-source-tracking). |
| `schedule_mode` | string | Read/write | Active schedule mode: `HEAT`, `COOL`, or `RANGE`. See [Temperature schedules](#temperature-schedules). |
| `can_heat` | boolean | Read | Device supports heating |
| `can_cool` | boolean | Read | Device supports cooling |
| `hvac_heater_state` | boolean | Read | Primary heater running |
| `hvac_heat_x2_state` | boolean | Read | Stage 2 heat running |
| `hvac_heat_x3_state` | boolean | Read | Stage 3 heat running |
| `hvac_aux_heater_state` | boolean | Read | Auxiliary heater running |
| `hvac_alt_heat_state` | boolean | Read | Alternate heat source running |
| `hvac_alt_heat_x2_state` | boolean | Read | Alternate heat stage 2 running |
| `hvac_emer_heat_state` | boolean | Read | Emergency heat running |
| `hvac_ac_state` | boolean | Read | Air conditioning running |
| `hvac_cool_x2_state` | boolean | Read | Stage 2 cooling running |
| `hvac_cool_x3_state` | boolean | Read | Stage 3 cooling running |
| `hvac_fan_state` | boolean | Read | Fan running |
| `name` | string | Read/write | Device display name |

#### target_temperature vs schedule setpoints

The `target_temperature` field in the shared bucket may be updated by schedule transitions, manual overrides (dial turns), or cloud pushes. The device evaluates its own schedule locally using internal timers. When a schedule transition fires, the device updates its internal setpoint and may update `target_temperature` in the shared bucket to reflect the new schedule value.

This has critical implications for server implementations:

- **Do not re-push a stale `target_temperature` to the device.** If your server pushes shared bucket data containing an old `target_temperature` value that differs from what the device currently has stored, the device treats it as a new cloud override and applies it — canceling the schedule-derived setpoint.
- **Pushing `target_temperature` creates a temporary hold.** The device treats any cloud-pushed `target_temperature` as a manual override that persists until the next schedule transition. This is the intended mechanism for remote temperature control.
- **The same-value guard prevents redundant overrides.** If the pushed `target_temperature` equals the value already stored in the shared bucket, the device ignores it (no event is fired). Problems only arise when the value differs.
- **During eco mode, `target_temperature` tracks the schedule.** The schedule continues running underneath eco mode. Schedule transitions update `target_temperature` even while eco is active. The eco temperature override happens at the HVAC control layer, not the schedule layer.

In practice, this means your server should only include `target_temperature` in a subscribe response when it has genuinely changed (for example, from a user action in your app). Simply echoing back the last known value on every subscribe can inadvertently override schedule transitions.

#### Schedule transition PUT ordering

When a schedule transition fires, the device sends HVAC state changes (e.g., `hvac_fan_state`, `hvac_heater_state`) and the new `target_temperature` in separate PUTs. The HVAC changes are sent first because they are synchronous and non-delayable. The temperature change is sent second because it goes through the defer delay mechanism (`X-nl-defer-device-window`).

This creates a window where the server has received updated HVAC states but still holds the **previous** `target_temperature`. If the server pushes the shared bucket to the device during this window (via subscribe or PUT response), the stale `target_temperature` overwrites the schedule-derived setpoint.

**Mitigation**: Set `X-nl-defer-device-window` to a non-zero value (e.g., 15–30 seconds). This causes the device to batch the temperature change into a single PUT along with any other pending changes, closing the window. Without this header, the device sends temperature changes immediately but in a separate PUT from the HVAC changes — maximizing the stale echo window.

#### Temperature precision

Temperature values are stored internally as fixed-point numbers. Changes smaller than ~0.01 C are treated as insignificant and ignored.

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

#### How display wake works

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

#### Temperature change source tracking

The device tags every temperature setpoint with metadata describing what caused the change. This is serialized as a `touched_by` object in the shared bucket:

```json
{
  "touched_by": {
    "touched_by": 1,
    "touched_at": 1707148800,
    "touched_tzo": -18000,
    "touched_user_id": ""
  }
}
```

| `touched_by` value | Source |
|---------------------|--------|
| `1` | Schedule transition |
| `2` | Local user interaction (dial turn) |
| `3` | Remote/cloud push |

When a user turns the dial, the device creates a "temperature hold" that persists until the next schedule transition. The device displays "Holding until ..." on screen. There is no persistent hold flag — the hold implicitly expires when the schedule timer fires the next different-temperature setpoint.

When pushing `target_temperature` from the cloud, set `touched_by.touched_by` to `3` and include a `touched_by.touched_user_id` if your server tracks user identity.

### structure bucket

Home-level settings that apply to all devices in a structure. The device never sends this bucket in PUT requests.

**Object key**: `structure.{structureId}`
**Direction**: Server &rarr; device only

For eco mode control and structure key selection, see [Home/Away and eco mode](#homeaway-and-eco-mode).

#### Fields

| Field | Type | Description |
|-------|------|-------------|
| `away` | boolean | Away state. Use `manual_eco_all` for direct eco control. See [Home/Away and eco mode](#homeaway-and-eco-mode). |
| `manual_eco_all` | boolean | Eco mode control. `true` activates eco mode. |
| `manual_eco_timestamp` | integer | Unix timestamp (seconds) of the eco change. **Must be within 600 seconds of device clock.** |
| `house_type` | string | House type classification |
| `renovation_date` | string | Year of last renovation |
| `num_thermostats` | string | Number of thermostats in the structure |
| `country_code` | string | ISO country code |
| `postal_code` | string | Postal/ZIP code |
| `location` | object | Location data (zipcode, country, latitude, longitude) |
| `time_zone` | object | Timezone information |
| `devices` | array | JSON array of device object keys (for example, `["device.09AA01AC12345678"]`) |
| `name` | string | Structure name |
| `dr_reminder_enabled` | boolean | Demand response reminders enabled |

### user bucket

Exists solely to trigger pairing completion on the device. The device ignores all fields except `name`. See [Pairing](#pairing) for the complete flow.

**Object key**: `user.{userId}`
**Direction**: Server &rarr; device only

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | User identifier. Triggers pairing completion on the device. |

### schedule bucket

Temperature schedules. See [Temperature schedules](#temperature-schedules) for the complete JSON format, day indexing, setpoint fields, and sync behavior.

**Object key**: `schedule.{serial}`
**Direction**: Bidirectional, subject to [sync guards](#schedule-sync-guards)
**Revision type**: `base_object_revision` (unconditional)

### where bucket

Room and location labels for devices. The `where_id` field in the device bucket references a `where` bucket entry.

**Object key**: `where.{whereId}`
**Direction**: Bidirectional
**Revision type**: `base_object_revision`

### message bucket

Cloud-to-device messages, including reboot commands and notifications.

**Object key**: `message.{messageId}`
**Direction**: Bidirectional
**Revision type**: `base_object_revision`

### link bucket

Associates a device with a structure. The device receives this on subscribe but doesn't PUT it.

**Object key**: `link.{linkId}`
**Direction**: Server &rarr; device only

### custom_schedule bucket

User-created custom temperature schedules. Same JSON format as the [schedule bucket](#schedule-bucket). The identifier is **server-assigned** — the device never creates schedule IDs on its own. See [Custom schedules](#custom-schedules).

**Object key**: `custom_schedule.{id}`
**Direction**: Bidirectional
**Revision type**: `base_object_revision`

### device_alert_dialog bucket

Server-initiated dialog prompts displayed on the device screen.

**Object key**: `device_alert_dialog.{id}`
**Direction**: Server &rarr; device only

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

### hvac_partner bucket

Data from Heat Link boilers, primarily EU/UK systems.

**Object key**: `hvac_partner.{partnerId}`
**Direction**: Bidirectional
**Revision type**: `base_object_revision`

| Field | Type | Description |
|-------|------|-------------|
| `boiler_setpoint` | float | Boiler target temperature (Celsius, range: -40 to 127) |
| `modulation` | float | Boiler modulation level |
| `gas_usage` | float | Gas usage metric |
| `water_temp` | float | Water temperature (Celsius) |
| `fault_code` | string | Active fault code |
| `flame_on` | boolean | Burner flame status |

### topaz bucket

Status data from Nest Protect (smoke and CO detector) devices. The thermostat uses Protect occupancy sensors for auto-away decisions.

**Object key**: `topaz.{topazId}`
**Direction**: Server &rarr; device only

The device subscribes to only 7 fields from this bucket. Other fields in the push are silently ignored.

| Field | Type | Description |
|-------|------|-------------|
| `co_status` | integer | Carbon monoxide alarm status |
| `smoke_status` | integer | Smoke alarm status |
| `line_power_present` | boolean | Mains power connected |
| `hushed_state` | boolean | Alarm silenced |
| `spoken_where_id` | string | Room assignment for voice announcements |
| `thread_mac_address` | string | Thread radio MAC address |
| `home_away_input` | boolean | Occupancy sensor home/away state |

### kryptonite bucket

Data from Nest Temperature Sensors (remote wireless sensors).

**Object key**: `kryptonite.{sensorId}`
**Direction**: Bidirectional

The device subscribes to only 2 fields from this bucket:

| Field | Type | Description |
|-------|------|-------------|
| `where_id` | string | Room assignment |
| `description` | string | Sensor description |

### Remaining bucket types

These buckets support energy utility programs, peripheral devices, and diagnostics. They follow the standard sync protocol but are rarely relevant for home server implementations. Store any data the device sends for these buckets, and serve it back on subscribe.

| Bucket | Description |
|--------|-------------|
| `servicegroup` | Service group configuration (server &rarr; device) |
| `occupancy` | Per-device occupancy data (device &rarr; server, rarely used) |
| `tuneups` | Seasonal Savings energy optimization events |
| `demand_response` | Demand response program enrollment |
| `demand_response_event` | Individual demand response event instances |
| `utility` | Utility provider information (server &rarr; device) |
| `diamond_sensor_config` | Temperature sensor configuration (server &rarr; device) |
| `diamond_sensor_event` | Temperature sensor event data |
| `rate_plan` | Utility rate plan information (server &rarr; device) |
| `tou` | Time-of-use electricity pricing |
| `demand_charge` | Demand charge configuration (server &rarr; device) |
| `demand_charge_event` | Demand charge event instances |
| `rcs_settings` | Remote comfort sensor settings |
| `cloud_algo` | Cloud algorithm parameters |
| `diagnostics` | Device diagnostics data |

### Temperature values

All temperature values across all buckets use **Celsius**. The `temperature_scale` field in the device bucket (`F` or `C`) is a display-only preference — it doesn't affect the data format.

The device serializes temperatures with up to 5 decimal places (for example, `21.00000`). Changes smaller than approximately 0.01 C are ignored.

### Write protection

The device protects ~113 device-only fields from server writes. If you push a value for one of these fields, the device compares it against its local value. If different, it marks the field dirty and re-sends its own value in the next PUT — actively overwriting your change. Accept these re-PUTs normally.

Twelve device-only fields have explicit consistency checking and logging:

`heat_link_connection`, `error_code`, `wiring_error`, `auto_dehum_state`, `away_temperature_low_adjusted`, `away_temperature_high_adjusted`, `dehumidifier_state`, `demand_charge_icon`, `fan_control_state`, `fan_cooling_state`, `humidifier_state`, `tou_icon`

Don't write to device-only fields. The device always wins.

### Safety fields

When any safety-related field changes, the device forces four additional fields into the PUT regardless of whether they changed: `battery_level`, `safety_temp_activating_hvac`, `safety_state`, and `safety_state_time`. Accept these extra fields in your normal merge process.

### Schedule sync guards

Three mechanisms protect schedule integrity. For details and examples, see [Schedule behavior details](#schedule-behavior-details).

1. **15-second debounce**: Multiple schedule pushes within 15 seconds — only the last one takes effect.
2. **Pending local change**: If the user is editing the schedule on the device, incoming server pushes are discarded.
3. **Timestamp rejection**: Schedules with older timestamps than the current schedule are silently rejected.

### PUT order

When the device sends a PUT with multiple buckets, they appear in this order:

`demand_response` &rarr; `demand_response_event` &rarr; `tuneups` &rarr; `structure` &rarr; `schedule` &rarr; `custom_schedule` &rarr; `message` &rarr; `shared` &rarr; `device` &rarr; `where` &rarr; `diamond_sensor_event` &rarr; `tou` &rarr; `hvac_partner` &rarr; `cloud_algo` &rarr; `demand_charge_event` &rarr; `rcs_settings` &rarr; `kryptonite` &rarr; `diagnostics`

The server doesn't need to process them in order. This is documented for debugging.

---

## Thermostat control

The previous sections cover the transport protocol and data model. This section explains how to use those primitives to control the thermostat — setting modes, changing temperatures, reading state, and configuring features.

For eco mode control, see [Home/Away and eco mode](#homeaway-and-eco-mode). For schedule management, see [Temperature schedules](#temperature-schedules).

### State model

The thermostat's state isn't a single "mode." It's the combination of four independent dimensions, each controlled through different bucket fields.

The following table summarizes the four dimensions.

| Dimension | What it controls | Bucket | Key field | Settable by server? |
|-----------|-----------------|--------|-----------|---------------------|
| HVAC mode | Heating, cooling, or both | Shared | `target_temperature_type` | Yes |
| Temperature setpoint | Target temperature | Shared | `target_temperature` (and variants) | Yes |
| Eco mode | Home/away energy state | Structure | `manual_eco_all` | Yes |
| HVAC operation | What hardware is running | Shared | `hvac_*_state` fields | No (read-only) |

These dimensions are independent — changing one doesn't reset the others:

- Setting `target_temperature_type` to `"off"` preserves the previous HVAC mode internally. Setting it back to `"heat"` resumes where it left off.
- Activating eco mode doesn't change the HVAC mode or schedule. When eco ends, the previous setpoints resume.
- The HVAC operation state is a consequence of the other dimensions — the server observes it but can't set it directly.

### HVAC modes

The HVAC mode determines whether the thermostat heats, cools, or does both. Set it by pushing `target_temperature_type` in the [shared bucket](#shared-bucket).

The following table lists the supported values.

| Value | Behavior | Temperature fields used | Wiring required |
|-------|----------|------------------------|-----------------|
| `"heat"` | Heating only | `target_temperature` | `can_heat` |
| `"cool"` | Cooling only | `target_temperature` | `can_cool` |
| `"range"` | Automatic heat and cool | `target_temperature_low`, `target_temperature_high` | `can_heat` and `can_cool` |
| `"off"` | All HVAC disabled | None | — |
| `"emergency"` | Emergency/auxiliary heat | `target_temperature` | `has_emer_heat` (device bucket) |

Values are case-insensitive. The device sends lowercase in PUT requests.

#### Validate mode against capabilities

Before setting a mode, check the device's wiring capabilities. The `can_heat` and `can_cool` fields are in the [shared bucket](#shared-bucket). The `has_emer_heat` field is in the [device bucket](#device-bucket).

If you push a mode the device's wiring can't support, the device silently falls back to a supported mode — preferring heat over cool. Check capabilities before offering modes in a UI to avoid confusing the user.

#### Emergency heat

Emergency heat bypasses the heat pump compressor and runs the auxiliary heater directly. This is expensive and intended for equipment failure or extreme cold.

When emergency heat activates, the device automatically:

- Saves and disables learning mode.
- Saves and disables auto-away.
- Blocks preconditioning entirely.
- Restores all saved settings when emergency heat is turned off.

Don't leave a device in emergency heat long-term. It significantly increases energy costs.

#### Example: set HVAC mode

```json
{
  "objects": [
    {
      "object_revision": 460,
      "object_timestamp": 1707148800000,
      "object_key": "shared.09AA01AB12345678",
      "value": {
        "target_temperature_type": "cool"
      }
    }
  ]
}
```

> **Note:** Remember that `object_revision` and `object_timestamp` must appear before `object_key` in the JSON. See [Response format](#response-body).

### Set the temperature

Which temperature fields to set depends on the active HVAC mode. For the complete field list, see [shared bucket](#shared-bucket).

#### Single-setpoint modes

In `heat`, `cool`, or `emergency` mode, set `target_temperature`:

```json
{
  "objects": [
    {
      "object_revision": 461,
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

#### Dual-setpoint mode

In `range` mode, set both `target_temperature_low` (heating threshold) and `target_temperature_high` (cooling threshold):

```json
{
  "objects": [
    {
      "object_revision": 461,
      "object_timestamp": 1707148800000,
      "object_key": "shared.09AA01AB12345678",
      "value": {
        "target_temperature_low": 18.0,
        "target_temperature_high": 24.0,
        "target_change_pending": true
      }
    }
  ]
}
```

The device heats when the current temperature drops below `target_temperature_low` and cools when it rises above `target_temperature_high`.

#### Temperature encoding and limits

All temperatures across all buckets are **Celsius floats** — for example, `21.5`. The device converts for display based on the `temperature_scale` field in the device bucket. See [Temperature values](#temperature-values).

The following table summarizes the valid range.

| Property | Value |
|----------|-------|
| Unit | Celsius |
| Format | Float (for example, `21.5`) |
| Recommended range | 4.5 °C – 32.0 °C (40 °F – 90 °F) |
| Device-accepted range | ~2.0 °C – 58.0 °C |
| Precision | ~0.01 °C minimum meaningful increment |

#### Display wake

Include `target_change_pending: true` whenever you push a temperature change. This wakes the physical display and plays the temperature animation. See [Display wake behavior](#display-wake-behavior) for the acknowledgment flow.

### Read device state

The device reports its state through read-only fields in PUT requests. Store these values and use them to drive your UI and automation logic.

#### Current conditions

| Field | Bucket | Type | Description |
|-------|--------|------|-------------|
| `current_temperature` | Device | float | Indoor temperature (°C) |
| `current_humidity` | Device | integer | Indoor relative humidity (%) |
| `backplate_temperature` | Device | float | Backplate temperature (°C) |
| `battery_level` | Device | float | Battery charge level (0.0–1.0) |

#### Equipment capabilities

Check these fields before offering controls in a UI.

| Field | Bucket | Type | Description |
|-------|--------|------|-------------|
| `can_heat` | Shared | boolean | Heating wiring detected |
| `can_cool` | Shared | boolean | Cooling wiring detected |
| `has_emer_heat` | Device | boolean | Emergency/auxiliary heat wiring detected |
| `has_fan` | Device | boolean | Fan control wiring detected |
| `has_humidifier` | Device | boolean | Humidifier wiring detected |
| `has_dehumidifier` | Device | boolean | Dehumidifier wiring detected |
| `has_hot_water_control` | Device | boolean | Hot water system detected (UK/EU) |

#### HVAC operation

The [shared bucket](#shared-bucket) contains individual boolean fields for each wire relay. Use them to determine what equipment is currently running.

The following table maps equipment categories to their state fields.

| Currently... | Check these shared bucket fields |
|-------------|----------------------------------|
| Heating | `hvac_heater_state`, `hvac_heat_x2_state`, `hvac_heat_x3_state` |
| Cooling | `hvac_ac_state`, `hvac_cool_x2_state`, `hvac_cool_x3_state` |
| Emergency heating | `hvac_emer_heat_state` |
| Auxiliary/alternate heat | `hvac_aux_heater_state`, `hvac_alt_heat_state`, `hvac_alt_heat_x2_state` |
| Fan running | `hvac_fan_state` |

If any field in a category is `true`, that equipment type is active. Multiple fields can be `true` simultaneously — for example, stage 1 and stage 2 heating.

#### Eco state

The device reports its eco mode through the `eco_mode` field in the device bucket. This is a read-only JSON string:

```json
{"mode":"manual-eco","touched_by":3,"mode_update_timestamp":1738800000,"touched_user_id":"userId"}
```

The `mode` value indicates the current eco state. For the full list of values and how to control eco mode, see [Home/Away and eco mode](#homeaway-and-eco-mode).

#### Time to target

| Field | Bucket | Type | Description |
|-------|--------|------|-------------|
| `time_to_target` | Device | integer | Estimated seconds until the target temperature is reached. `0` when at target. |
| `time_to_target_training` | Device | string | `"ready"` when estimates are calibrated, `"training"` while calibrating, `"not_ready"` when no estimates are available. |

### Feature reference

The device supports independent features that overlay on the core state model. Each feature can be enabled or disabled through fields in the [device bucket](#device-bucket). This section describes what each feature does and its key fields. For the complete field list, see the [cloud-writable fields](#cloud-writable-fields) tables.

#### Fan control

Controls the HVAC fan independently of heating and cooling.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `fan_mode` | Bidirectional | string | `"off"`, `"auto"`, or `"duty-cycle"` |
| `fan_timer_duration` | Bidirectional | integer | Fan timer length in seconds |
| `fan_timer_timeout` | Bidirectional | integer | Fan timer end time (Unix timestamp). Set to `0` to stop the timer. |
| `fan_duty_cycle` | Bidirectional | integer | Minutes per hour the fan runs in duty-cycle mode |
| `fan_cooling_enabled` | Server → device | boolean | Enable Airwave — runs the fan with residual cold from the AC coil to save compressor energy |
| `fan_cooling_state` | Device → server | string | Current Airwave state |

To start a fan timer, set both `fan_timer_duration` and `fan_timer_timeout`. Check `has_fan` before showing fan controls.

#### Safety temperature

Forces heating or cooling when the indoor temperature crosses a threshold, **regardless of all other state** — including system off, eco mode, and the active schedule. Safety temperatures are the highest-priority override.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `upper_safety_temp` | Bidirectional | float | Cool activates above this temperature (°C) |
| `upper_safety_temp_enabled` | Bidirectional | boolean | Enable the high threshold |
| `lower_safety_temp` | Bidirectional | float | Heat activates below this temperature (°C) |
| `lower_safety_temp_enabled` | Bidirectional | boolean | Enable the low threshold |
| `safety_state` | Device → server | string | `"safe"`, `"below"`, or `"above"` |
| `safety_temp_activating_hvac` | Device → server | boolean | Safety override is currently running HVAC |

When any safety field changes, the device forces four additional fields into its next PUT regardless of whether they changed: `battery_level`, `safety_temp_activating_hvac`, `safety_state`, and `safety_state_time`. See [Safety fields](#safety-fields).

#### Temperature lock

Locks the physical dial to prevent unauthorized temperature changes. The device requires a PIN to unlock.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `temperature_lock` | Server → device | boolean | Enable the lock |
| `temperature_lock_pin_hash` | Bidirectional | string | PIN hash for unlocking |
| `temperature_lock_low_temp` | Bidirectional | float | Minimum allowed temperature when locked (°C) |
| `temperature_lock_high_temp` | Bidirectional | float | Maximum allowed temperature when locked (°C) |

When locked, the user can still adjust the temperature on the dial but only within the bounds you set.

#### Preconditioning

Starts heating or cooling early so the home reaches the target temperature at the scheduled transition time. The server can enable or disable it but can't trigger it directly — the device calculates when to start based on thermal history.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `preconditioning_enabled` | Server → device | boolean | Enable early-start scheduling |
| `preconditioning_ready` | Device → server | boolean | Feature is ready (has enough thermal data) |
| `eta_preconditioning_active` | Device → server | boolean | Currently preconditioning |
| `max_nighttime_preconditioning_seconds` | Bidirectional | integer | Maximum preconditioning duration at night (seconds) |

> **Note:** Preconditioning is automatically blocked during server-initiated eco mode (`manual_eco_all`). See [Home/Away and eco mode](#homeaway-and-eco-mode).

#### Learning mode

The auto-schedule feature learns from the user's manual temperature adjustments and modifies the schedule over time.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `learning_mode` | Bidirectional | boolean | Auto-schedule learning enabled |
| `schedule_learning_reset` | Server → device | boolean | Reset all learned schedule data |

Learning pauses automatically during eco mode and emergency heat.

> **Note:** When learning is enabled, the device may modify a schedule you pushed. If you need the schedule to remain exactly as pushed, disable learning first.

#### Humidity control

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `target_humidity` | Bidirectional | float | Target humidity percentage |
| `target_humidity_enabled` | Bidirectional | boolean | Enable humidity targeting |
| `current_humidity` | Device → server | integer | Current humidity reading (%) |
| `auto_dehum_enabled` | Server → device | boolean | Enable automatic dehumidification |
| `humidifier_type` | Bidirectional | string | Humidifier type |
| `dehumidifier_type` | Bidirectional | string | Dehumidifier type |

Check `has_humidifier` and `has_dehumidifier` before showing humidity controls.

#### Sunblock

Compensates for direct sunlight on the temperature sensor by adjusting the reading.

| Field | Direction | Description |
|-------|-----------|-------------|
| `sunlight_correction_enabled` | Bidirectional | Enable sunlight compensation |
| `sunlight_correction_active` | Device → server | Sunlight currently detected on sensor |
| `sunlight_correction_ready` | Device → server | Feature has enough data to operate |

#### Heat pump balance

Controls the tradeoff between energy savings and comfort for heat pump systems.

| Field | Direction | Values |
|-------|-----------|--------|
| `heatpump_savings` | Bidirectional | `"max-savings"`, `"balanced"`, or `"max-comfort"` |

#### Radiant heat

Optimizes control for radiant/underfloor heating systems by accounting for their slower thermal response time.

| Field | Direction | Type |
|-------|-----------|------|
| `radiant_control_enabled` | Bidirectional | boolean |

#### Hot water (UK/EU)

For Heat Link systems with domestic hot water control.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `hot_water_mode` | Bidirectional | string | `"schedule"` or `"off"` |
| `hot_water_boost_time_to_end` | Bidirectional | integer | Boost timer end (Unix timestamp) |
| `hot_water_active` | Device → server | boolean | Hot water currently heating |
| `hot_water_away_enabled` | Bidirectional | boolean | Allow hot water during eco mode |

Check `has_hot_water_control` before showing hot water controls.

#### Smoke and CO safety shutoff

Integrates with Nest Protect devices to shut off HVAC when smoke or CO is detected.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `smoke_safety_shutoff_enabled` | Bidirectional | boolean | Enable smoke-triggered HVAC shutoff |
| `safety_shutoff_enabled` | Bidirectional | boolean | Enable CO-triggered HVAC shutoff |
| `hvac_smoke_safety_shutoff_active` | Device → server | boolean | HVAC currently shut off due to smoke |
| `hvac_safety_shutoff_active` | Device → server | boolean | HVAC currently shut off due to CO |

#### Compressor lockout

Prevents the compressor from short-cycling by enforcing a minimum off time between cycles.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `compressor_lockout_enabled` | Bidirectional | boolean | Enable compressor lockout |
| `compressor_lockout_timeout` | Bidirectional | integer | Minimum off time in seconds |

#### Display and sound

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `farsight_screen` | Bidirectional | string | What the display shows on standby |
| `should_wake_on_approach` | Bidirectional | boolean | Wake the display when someone approaches |
| `click_sound` | Bidirectional | boolean | Audible click on dial turn |
| `temperature_scale` | Bidirectional | string | Display unit: `"F"` or `"C"`. Display-only — all data is always Celsius. |

#### Leaf thresholds

The green leaf icon appears when the current setpoint is energy-efficient. The server can adjust the thresholds.

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `leaf_threshold_heat` | Bidirectional | float | Leaf appears below this heating setpoint (°C) |
| `leaf_threshold_cool` | Bidirectional | float | Leaf appears above this cooling setpoint (°C) |
| `leaf_schedule_delta` | Bidirectional | float | Offset from schedule setpoint used to compute leaf thresholds (°C). The device's learning algorithm adjusts this over time. |
| `leaf_away_low` | Bidirectional | float | Eco heating threshold for leaf (°C) |
| `leaf_away_high` | Bidirectional | float | Eco cooling threshold for leaf (°C) |
| `leaf` | Device → server | boolean | Whether the leaf icon is currently displayed |

#### Filter reminder

| Field | Direction | Type | Description |
|-------|-----------|------|-------------|
| `filter_reminder_enabled` | Server → device | boolean | Enable filter change reminders |
| `filter_changed_date` | Bidirectional | integer | Last filter change (Unix timestamp) |
| `filter_replacement_needed` | Device → server | boolean | Filter needs replacement |
| `filter_runtime_sec` | Device → server | integer | Total filter runtime in seconds |

### State interactions

The four state dimensions operate independently, but some combinations produce specific behaviors. The following table describes how common server actions affect the device.

| Server action | HVAC mode | Temperature source | Eco mode | HVAC runs? |
|--------------|-----------|-------------------|----------|------------|
| Set `target_temperature_type` to `"heat"` | → Heat | Schedule or manual setpoint | Unchanged | If below setpoint |
| Set `target_temperature_type` to `"off"` | → Off | — | Unchanged | No |
| Set `manual_eco_all` to `true` | Unchanged | → Eco temperatures | → Manual-eco | Only outside eco band |
| Set `manual_eco_all` to `false` | Unchanged | → Schedule setpoint (fresh lookup) | → Schedule | If below/above setpoint |
| Push a new schedule | Unchanged | Updated for future transitions | Unchanged | Recalculated |
| Safety threshold crossed | Overridden | Overridden | Overridden | Forced on |
| Emergency mode set | → Emergency | Manual setpoint | Unchanged | Emergency heat |
| User turns dial | Unchanged | → Manual setpoint | Eco reverts to schedule | Recalculated |

#### Eco mode and feature interactions

Several features change behavior during eco mode.

| Feature | Normal operation | During eco mode |
|---------|-----------------|-----------------|
| Preconditioning | Runs normally | **Blocked** (manual-eco only) |
| Schedule following | Active | Schedule timer continues running internally; HVAC uses eco temperatures instead of schedule setpoints |
| Learning | Active | Paused |
| Safety temperature | Active | Active — overrides eco |
| Fan timer | Active | Active |

---

## Home/Away and eco mode

Eco mode is an energy-saving state where the device relaxes its temperature targets. Instead of following the active schedule, the device uses wider temperature bounds — only running the HVAC if the indoor temperature crosses a high or low threshold. When eco mode ends, the device resumes the current schedule setpoint.

Your server controls eco mode through the **structure bucket**. The device also has a built-in occupancy sensor that can trigger eco independently, but server-initiated eco takes priority.

### Enter eco mode

To activate eco mode, set `manual_eco_all` to `true` in the structure bucket. Eco mode takes effect as soon as the device processes the update.

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

Include a `manual_eco_timestamp` set to the current Unix time in **seconds** (not milliseconds). The device rejects the change if this timestamp differs from its own clock by more than **600 seconds**. Keep your server clock NTP-synced.

### Exit eco mode

To deactivate eco mode, send `manual_eco_all: false` and `away: false` in the structure bucket, and write `eco.mode: "schedule"` to the device bucket:

```json
{
  "objects": [
    {
      "object_revision": 6,
      "object_timestamp": 1707148801000,
      "object_key": "structure.your_structure_id",
      "value": {
        "manual_eco_all": false,
        "manual_eco_timestamp": 1707148801,
        "away": false
      }
    },
    {
      "object_revision": 7,
      "object_timestamp": 1707148801000,
      "object_key": "device.your_serial_number",
      "value": {
        "eco": {
          "mode": "schedule",
          "touched_by": 3,
          "mode_update_timestamp": 1707148801
        }
      }
    }
  ]
}
```

> **Caution**: Always include `away: false` when exiting eco mode. The `manual_eco_all` field is subject to the same 600-second timestamp validation on exit as on entry — if the device's clock has drifted, it silently drops the change and eco mode persists. The `away` field has no timestamp validation, so it always takes effect and serves as a reliable fallback.
>
> **Caution**: Always include the device bucket `eco.mode: "schedule"` write when exiting eco mode. If your server pushes other device bucket fields in the same subscribe response (which is typical), the device processes the entire merged object. Any stale `eco.mode: "manual-eco"` left over from a previous write will cause the device to re-enter eco mode immediately after the structure bucket exits it. Writing `eco.mode: "schedule"` explicitly overwrites any stale value.

After exiting eco mode, the device performs a fresh schedule lookup. Any previous manual temperature override (from a dial turn or server push) is not restored — the device uses whatever the schedule says at that moment.

The user can also exit eco mode by physically turning the thermostat dial. The occupancy sensor does not automatically exit server-initiated eco — only a physical interaction or a server push clears it.

### Eco temperatures

During eco mode, the device uses `away_temperature_high` and `away_temperature_low` from the device bucket as its setpoints. These define the bounds: the device heats if the temperature drops below `away_temperature_low`, and cools if it rises above `away_temperature_high`. Both fields are cloud-writable.

These are separate from the `target_temperature` fields in the shared bucket, which continue to track the active schedule during eco mode.

### The `away` field

The structure bucket also has an `away` field that provides a second way to trigger eco mode. When the device receives `away: true`, it starts a timer. If the timer expires without being cancelled, the device enters eco mode on its own.

The key differences from `manual_eco_all`:

| | `manual_eco_all` | `away` |
|-|-------------------|--------|
| Activation | Immediate | Delayed (device timer) |
| Timestamp validation | Yes (600-second window) | `away: true` validated, `away: false` is not |
| Schedule preconditioning | Blocked during eco | Can end eco early |

The `away` field is designed for occupancy-based automation. If your server tracks whether anyone is home, set `away: true` when the home is unoccupied and `away: false` when someone returns. The built-in delay prevents false triggers from brief absences.

For direct eco control — for example, a user tapping an "Eco" button in your app — use `manual_eco_all`. The immediate activation provides a better user experience, and eco mode won't be interrupted by the device's schedule.

### Read eco mode status

The device reports its current eco state through the `eco_mode` field in the [device bucket](#device-bucket):

| Value | Meaning |
|-------|---------|
| `"schedule"` | Not in eco mode. Following the schedule normally. |
| `"manual-eco"` | Server-initiated eco (set through `manual_eco_all`). |
| `"auto-eco"` | Device-initiated eco (occupancy sensor or `away` timer). |

### Deliver the structure bucket

The device doesn't subscribe to the structure bucket by default. Your server must include it as an additional object in the subscribe response — the same mechanism used for [pairing](#pairing).

Once the device receives a structure bucket, it remembers the structure key and includes it in subsequent subscribe requests. From that point on, the device expects structure updates through the normal subscribe flow.

For the initial delivery (before the device knows about the structure), include the structure bucket alongside any other updates you push.

### Choose the structure key

| Scenario | Structure key | Example |
|----------|---------------|---------|
| Device has an owner (claimed) | Derive from user or owner identity | `structure.user123` |
| Device has no owner (unclaimed) | Use `structure.default` | `structure.default` |

The device processes the bucket the same way regardless of the key name — what matters is that the bucket type is `structure`.

### Device bucket away fields

The device bucket contains several away-related fields. The `auto_away` field is read-only (device to server). The others are cloud-writable but don't control eco mode — use the structure bucket fields instead.

| Field | Type | Direction | Description |
|-------|------|-----------|-------------|
| `auto_away` | integer | Device &rarr; server | Occupancy sensor output. `0` = home, `1` = away. |
| `auto_away_enable` | boolean | Bidirectional | Whether the occupancy sensor is active. |
| `auto_away_reset` | boolean | Bidirectional | Reset occupancy learning data (sensor history, not eco state). |
| `home_away_input` | boolean | Bidirectional | Enable home/away feature globally. |
| `away_temperature_high` | float | Bidirectional | Upper eco temperature (Celsius). |
| `away_temperature_low` | float | Bidirectional | Lower eco temperature (Celsius). |

> **Caution**: Don't write `auto_away` to the device bucket to control eco mode — it's a read-only sensor output. For the complete list of device bucket fields and their access modes, see [device bucket](#device-bucket).

---

## Temperature schedules

The device maintains a weekly temperature schedule that determines its target temperature throughout the day. Your server can push complete schedules to the device and receive schedule updates when the user modifies the schedule locally (for example, by turning the dial).

### Schedule bucket

The schedule bucket uses the key `schedule.<serial>`, where `<serial>` is the device's serial number. The device subscribes to this bucket automatically.

| Property | Value |
|----------|-------|
| Bucket key | `schedule.<serial>` |
| Direction | Bidirectional |
| Subscribe | Device subscribes to this bucket |
| PUT | Device sends schedule changes via PUT |

> **Note**: Unlike the structure bucket, the device explicitly subscribes to the schedule bucket. You don't need to inject it — it appears in the device's subscribe request body alongside `device` and `shared`.

### Schedule JSON format

The schedule value is a single JSON object containing the full weekly schedule. Version is always `2`.

```json
{
  "ver": 2,
  "name": "Default",
  "schedule_mode": "HEAT",
  "days": {
    "0": {
      "0": {
        "type": "HEAT",
        "time": 25200,
        "entry_type": "setpoint",
        "temp": 21.00000
      },
      "1": {
        "type": "HEAT",
        "time": 32400,
        "entry_type": "setpoint",
        "temp": 18.50000
      }
    },
    "1": {},
    "2": {},
    "3": {},
    "4": {},
    "5": {},
    "6": {}
  }
}
```

#### Top-level fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ver` | integer | Yes | Schema version. Always `2`. |
| `name` | string | Yes | Schedule name (for example, `"Default"`). |
| `schedule_mode` | string | Yes | Active mode. One of: `HEAT`, `COOL`, `RANGE`. See [Schedule modes](#schedule-modes). |
| `days` | object | Yes | Container for daily setpoints, keyed `"0"` through `"6"`. |

> **Note**: The device parser accepts both `schedule_mode` and `mode` as the key name. Use `schedule_mode` for consistency.

### Day indexing

Days are indexed as integer strings starting from Monday:

| Key | Day |
|-----|-----|
| `"0"` | Monday |
| `"1"` | Tuesday |
| `"2"` | Wednesday |
| `"3"` | Thursday |
| `"4"` | Friday |
| `"5"` | Saturday |
| `"6"` | Sunday |

Within each day, setpoints are also keyed by sequential integer strings: `"0"`, `"1"`, `"2"`, and so on.

> **Warning**: The day-of-week mapping starts with Monday, not Sunday. This differs from many programming language conventions. Double-check your day mapping when generating schedules.

### Setpoint fields

Each setpoint object within a day has the following fields:

| Field | Type | Modes | Description |
|-------|------|-------|-------------|
| `type` | string | All | Setpoint type: `HEAT`, `COOL`, or `RANGE`. |
| `time` | integer | All | Seconds from midnight (0–86399). |
| `entry_type` | string | All | `"setpoint"` for user-defined points, `"continuation"` for implied fill. See [Continuation setpoints](#continuation-setpoints). |
| `temp` | float | HEAT, COOL | Target temperature in Celsius. |
| `temp-min` | float | RANGE | Lower bound temperature in Celsius. |
| `temp-max` | float | RANGE | Upper bound temperature in Celsius. |

The `type` field in each setpoint must match the schedule's `schedule_mode`. A `HEAT` schedule contains only `HEAT` setpoints, and a `RANGE` schedule contains only `RANGE` setpoints.

#### HEAT or COOL setpoint example

```json
{
  "type": "HEAT",
  "time": 25200,
  "entry_type": "setpoint",
  "temp": 21.00000
}
```

The `time` value `25200` represents 7:00 AM (7 hours * 3600 seconds/hour).

#### RANGE setpoint example

```json
{
  "type": "RANGE",
  "time": 25200,
  "entry_type": "setpoint",
  "temp-min": 18.00000,
  "temp-max": 24.00000
}
```

In RANGE mode, use `temp-min` and `temp-max` instead of `temp`. The device heats if the temperature drops below `temp-min` and cools if it rises above `temp-max`.

### Temperature encoding

Temperatures in the schedule are always Celsius, regardless of the device's display preference (`temperature_scale` in the device bucket). The device converts to Fahrenheit internally for display only.

| Property | Value |
|----------|-------|
| Unit | Celsius (always) |
| Format | Floating-point with up to 5 decimal places |
| Precision | ~0.25°C minimum meaningful increment |

Write temperature values as floating-point numbers. The device stores them as fixed-point integers internally, so sub-0.25°C precision is lost.

### Schedule modes

The device supports several schedule modes, though only three are common for residential thermostats:

| Mode | Description | Setpoint fields |
|------|-------------|----------------|
| `HEAT` | Heating only | `temp` |
| `COOL` | Cooling only | `temp` |
| `RANGE` | Dual heat and cool | `temp-min`, `temp-max` |

The device also recognizes `EMERGENCY`, `HUMIDIFY`, `DEHUMIDIFY`, and `HOTWATER`, but these are uncommon and hardware-dependent. If you push a mode the device's hardware can't support (for example, `COOL` on a heat-only system), the device falls back to a supported mode.

The active schedule mode is stored in the shared bucket as `schedule_mode`. See [Switch schedule mode](#switch-schedule-mode).

### Push a schedule to the device

To push a schedule, include the schedule bucket in a subscribe response. The value is the complete schedule JSON object:

```json
{
  "objects": [
    {
      "object_revision": 100,
      "object_timestamp": 1707148800000,
      "object_key": "schedule.09AA01AB12345678",
      "value": {
        "ver": 2,
        "name": "My Schedule",
        "schedule_mode": "HEAT",
        "days": {
          "0": {
            "0": {"type": "HEAT", "time": 21600, "entry_type": "setpoint", "temp": 20.00000},
            "1": {"type": "HEAT", "time": 28800, "entry_type": "setpoint", "temp": 21.50000},
            "2": {"type": "HEAT", "time": 64800, "entry_type": "setpoint", "temp": 19.00000}
          },
          "1": {
            "0": {"type": "HEAT", "time": 21600, "entry_type": "setpoint", "temp": 20.00000},
            "1": {"type": "HEAT", "time": 28800, "entry_type": "setpoint", "temp": 21.50000},
            "2": {"type": "HEAT", "time": 64800, "entry_type": "setpoint", "temp": 19.00000}
          },
          "2": {},
          "3": {},
          "4": {},
          "5": {},
          "6": {}
        }
      }
    }
  ]
}
```

> **Important**: Always push the **complete** schedule, not individual setpoints. The device replaces the entire schedule on each push. There is no mechanism to add or remove individual setpoints.

> **Note**: Remember that `object_revision` and `object_timestamp` must appear before `object_key` in the JSON. See [Response format](#response-body).

### Read schedules from the device

The device sends schedule updates to your server through PUT requests. The schedule data appears under the `schedule.<serial>` key:

```http
POST /abc123/put HTTP/1.1
Content-Type: application/json

{
  "session": "sess_xyz789",
  "schedule.09AA01AB12345678": {
    "object_key": "schedule.09AA01AB12345678",
    "base_object_revision": 99,
    "ver": 2,
    "name": "Default",
    "schedule_mode": "HEAT",
    "days": {
      "0": {
        "0": {"type": "HEAT", "time": 25200, "entry_type": "setpoint", "temp": 21.00000}
      }
    }
  }
}
```

Store the schedule value in its entirety. When the device subscribes, compare revisions and timestamps as you would for any other bucket, and return the schedule if the server's version is newer.

### Switch schedule mode

The active schedule mode is controlled by the `schedule_mode` field in the **shared** bucket, not the schedule bucket. To switch between heating and cooling schedules, push a shared bucket update:

```json
{
  "objects": [
    {
      "object_revision": 459,
      "object_timestamp": 1707148800000,
      "object_key": "shared.09AA01AB12345678",
      "value": {
        "schedule_mode": "COOL"
      }
    }
  ]
}
```

The device validates the mode against its hardware capabilities:

| Mode | Requires |
|------|----------|
| `HEAT` | Heat capability (`can_heat: true` in shared bucket) |
| `COOL` | Cool capability (`can_cool: true` in shared bucket) |
| `RANGE` | Both heat and cool capability |

If the device can't support the requested mode, it falls back to a mode its hardware supports.

> **Note**: The `schedule_mode` field appears in two places: inside the schedule JSON (determines which setpoint types the schedule contains) and in the shared bucket (determines which schedule mode is active). Keep them in sync — if the shared bucket says `COOL` but you push a schedule with `schedule_mode: "HEAT"`, the device ignores the schedule.

### Continuation setpoints

Setpoints with `entry_type: "continuation"` are filler entries that carry forward the previous setpoint's temperature. The device generates these automatically when processing a schedule.

When pushing a schedule, you only need to include `"setpoint"` entries. The device fills in continuation entries for any gaps. If you include continuation entries in your push, the device accepts them, but it overwrites them during processing anyway.

When reading schedules from the device (via PUT), the device may include continuation entries. Store them as-is, but understand that only `"setpoint"` entries represent user-configured temperatures.

### Custom schedules

The device supports additional schedules beyond the default built-in schedule. Custom schedules use a separate bucket key format:

```
custom_schedule.<schedule_id>
```

The `<schedule_id>` is **server-generated**. The device never creates schedule IDs on its own — it expects the server to assign them. The JSON format inside a custom schedule bucket is identical to the standard schedule format.

Custom schedules appear in the device bucket as a comma-separated list of bucket keys. If your server doesn't need multiple schedules, you can ignore custom schedules entirely and work only with the default `schedule.<serial>` bucket.

### Schedule behavior details

#### 15-second debounce

The device debounces incoming schedule pushes with a 15-second sliding window. If you push multiple schedule updates in rapid succession, only the last one takes effect — after 15 seconds of quiet.

Push the schedule once and don't retry within the debounce window. There is no acknowledgment — the device applies the schedule silently.

> **Note**: During connection recovery (for example, after the device reconnects), the debounce is bypassed entirely. Schedules pushed during recovery take effect immediately.

#### Timestamp rejection

If the device's current schedule has a newer timestamp than the one you push, the device silently rejects the push. Always use a current `object_timestamp` when pushing schedules. Don't reuse old timestamps from previous responses.

#### Pending local changes

If the user has modified the schedule locally (for example, by adjusting the schedule through the device's dial or a connected app) and the change hasn't been sent to the server yet, the device discards incoming cloud schedules entirely. The device's local version takes priority.

This means a schedule push can silently fail if the user is actively editing the schedule. There is no error response — the device simply ignores the push and uploads its local version in the next PUT.

#### Maximum setpoints

Each day supports a maximum of 96 setpoints. In practice, schedules rarely have more than 10–12 setpoints per day.

#### Auto-schedule learning

The device has a built-in learning system that automatically modifies the schedule based on user behavior (dial turns, temperature holds). Cloud-pushed schedule changes are treated as user actions by the learning system, so the device may subsequently adjust the schedule you pushed.

To prevent the device from modifying the pushed schedule, disable learning by pushing `learning_mode: false` in the device bucket before pushing the schedule.

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

| Error | Device behavior | Server observes |
|-------|-----------------|-----------------|
| TCP timeout | Reconnect | Connection closes |
| TLS failure | Retry | Handshake fails |
| Keep-alive timeout | Reconnect on new connection | Old connection goes stale (no FIN/RST) |
| Wake timer expiry | Reconnect | Varies (RST or FIN) |

**Note**: The wake timer (`X-nl-suspend-time-max`) is a safety net. During normal operation, the server closes the connection before the timer fires. If the timer does fire (for example, if the server holds the connection too long), the device reconnects. To avoid this scenario, set your connection hold time shorter than `X-nl-suspend-time-max`. See [Connection hold time](#connection-hold-time).

**HTTP 400 Retry Behavior**: The device retries HTTP 400 errors up to 2 times (3 total attempts) before giving up.

> **TODO**: Document retry intervals and backoff strategy for 5xx errors from binary analysis.

---

## Implementation checklist

### Minimum requirements

- [ ] **[Include explicit port in server URL](#appendix-url-port-requirement)** (e.g., `:443`) — required for WoWLAN
- [ ] Implement `POST /entry` endpoint (returns `transport_url` with explicit port)
- [ ] Implement `POST /{czid}/subscribe` endpoint
- [ ] Return `Transfer-Encoding: chunked` on all subscribe responses
- [ ] Return `X-nl-suspend-time-max` header (recommend: 300)
- [ ] Keep connections open after sending headers
- [ ] Set connection hold time shorter than `X-nl-suspend-time-max` (e.g., suspend - 10s)
- [ ] Close idle connections by sending the final chunk terminator (`0\r\n\r\n`)
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
- [ ] Store schedule data from device PUTs and return it on subscribe when newer
- [ ] Push complete schedules (all days, all setpoints), not partial updates
- [ ] Use Celsius for all schedule temperatures (device converts for display)
- [ ] Use day indexing 0=Monday through 6=Sunday in schedule JSON
- [ ] Validate HVAC mode against device capabilities (`can_heat`, `can_cool`, `has_emer_heat`) before pushing
- [ ] Use the correct temperature fields for the active mode (`target_temperature` for single-setpoint, `target_temperature_low`/`target_temperature_high` for range)
- [ ] Store all device-reported read-only fields from PUT requests (sensor data, HVAC states, capabilities)

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
- [ ] Wait at least 15 seconds after pushing a schedule before pushing another (debounce)
- [ ] Serve schedule bucket on subscribe when device requests it
- [ ] Check `has_fan`, `has_humidifier`, `has_dehumidifier`, `has_hot_water_control` before showing feature controls
- [ ] Report `safety_state` and `hvac_*_state` fields in your UI so users can see what's running
- [ ] Store `time_to_target` and `time_to_target_training` for estimated arrival time in UI
- [ ] Expose safety temperature, temperature lock, and learning mode as user-configurable settings

### Avoid

- [ ] Closing connections after sending headers (before hold time expires)
- [ ] Using short `X-nl-suspend-time-max` for "responsiveness"
- [ ] Holding connections longer than `X-nl-suspend-time-max`
- [ ] Sending server-side TCP keep-alives (device cannot respond while asleep)
- [ ] Using service tickles for normal operation
- [ ] Sending `object_key` before `object_revision`/`object_timestamp` in JSON responses
- [ ] Using revision for sync decisions (use timestamp instead)
- [ ] Sending entry key `expires` as a JSON string (must be a number)
- [ ] Provisioning credentials through 401 responses (can cause credential loop)
- [ ] Using the structure bucket `away` field for eco mode control (overridden by schedule preconditioning)
- [ ] Pushing partial schedules (individual setpoints) — always push the complete schedule
- [ ] Using Fahrenheit in schedule temperature values — always use Celsius
- [ ] Pushing schedules rapidly without waiting for the 15-second debounce window
- [ ] Assuming Sunday=0 for schedule days — 0 is Monday
- [ ] Pushing `target_temperature_type` values without checking `can_heat`/`can_cool` (device falls back silently)
- [ ] Writing to device-only fields like `current_temperature` or `hvac_*_state` (device overwrites your values)
- [ ] Leaving a device in emergency heat long-term (high energy cost, disables learning and auto-away)

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
X-nl-suspend-time-max: 300
X-nl-service-timestamp: 1707148800000

```
(Connection held open, no body sent yet)

### Push temperature update

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
X-nl-suspend-time-max: 300
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
X-nl-suspend-time-max: 300
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
| Bucket types | Complete |
| Home/Away and eco mode | Complete |
| Thermostat control | Complete |
| Temperature schedules | Complete |
| Error handling | Partial |

---

## Changelog

| Revision | Date | Changes |
|----------|------|---------|
| 2.7 | 2026-02-19 | Rewrote Home/Away and eco mode section. Corrected eco exit guidance: `manual_eco_all: false` can be silently dropped due to 600-second timestamp validation (not "re-entered" as previously stated). Always send `away: false` alongside for reliable exit. Restructured section around developer tasks (enter, exit, read status) instead of internal firmware paths. Added comparison table for `manual_eco_all` vs `away`. Added eco mode status reading section. Removed internal terminology ("manual eco"/"auto eco" as mode names). |
| 2.6 | 2026-02-12 | Fixed suspend-time-max contradiction: all examples and recommendations now use 300 (was 600 in most places, contradicting the ≤350 constraint from idle connection timeout). Fixed connection errors table: keep-alive timeout causes stale connection (no FIN/RST), not RST packet. Fixed "no server-side mechanism to disable learning" — `learning_mode` is cloud-writable (mode 2). Fixed `current_humidity` type from `float` to `integer`. Fixed `can_heat`/`can_cool` bucket references — fields are shared bucket only (removed incorrect device bucket listing, corrected schedule modes section). |
| 2.5 | 2026-02-12 | Corrected subscribe connection lifecycle: server controls the reconnect cycle, not the device wake timer. Added "Connection hold time" section — server must close the connection before `X-nl-suspend-time-max`. Updated connection timing table, normal operation diagram, recommended server configuration, error handling, and implementation checklist. The previous guidance that server hold time should exceed suspend time was incorrect. |
| 2.4 | 2026-02-11 | Rewrote PUT response section: the PUT response is a write receipt, not a data channel — return `{object_revision, object_timestamp, object_key}` only, never include `value`. The device applies any `value` as authoritative cloud data with no per-key staleness check, silently overwriting local state. This applies to CAS conflict responses too. Added "Schedule transition PUT ordering" section explaining HVAC-first/temperature-second deterministic ordering and stale echo window. |
| 2.3 | 2026-02-09 | Eco mode and schedule interaction clarifications: schedule timer continues running during eco (not "suspended"), eco exit performs fresh schedule lookup (manual overrides not restored), `target_temperature` tracks schedule setpoints during eco mode. Added `touched_by` field to shared bucket fields table and new "Temperature change source tracking" section documenting the temperature hold mechanism. Corrected state interaction matrix eco exit row. |
| 2.2 | 2026-02-09 | Fixed `target_temperature` example values from Fahrenheit (72.0) to Celsius (22.00000). Added critical "target_temperature vs schedule setpoints" section to shared bucket documentation explaining that `target_temperature` is a user/cloud override (not the schedule-derived setpoint), the device evaluates schedules locally, and re-pushing stale values overrides schedule transitions. |
| 2.1 | 2026-02-08 | Added Thermostat control section: four-axis state model, HVAC modes (heat/cool/range/off/emergency) with validation against wiring capabilities, temperature setpoint control (single and dual setpoint with examples), device state reading guide (current conditions, equipment capabilities, HVAC operation mapping, eco state, time-to-target), comprehensive feature reference (fan, safety temperature, temperature lock, preconditioning, learning, humidity, sunblock, heat pump balance, radiant heat, hot water, smoke/CO safety shutoff, compressor lockout, display/sound, leaf thresholds, filter reminder), and state interaction matrix. Added `emergency` to shared bucket `target_temperature_type` values. Updated implementation checklist with mode validation, feature capability checks, and new avoid items. |
| 2.0 | 2026-02-08 | Expanded Bucket types section into comprehensive reference: all 28 bucket types with object key formats, sync directions, and priority classification. Added device bucket field access modes (device-only, special, cloud-writable) with complete field lists by category (~97 writable, ~113 read-only). Added shared bucket conditional write semantics and corrected field list (moved `fan_timer_timeout` and `fan_timer_duration` to device bucket). Added structure bucket complete field list. Added hvac_partner, topaz, and kryptonite field tables with subscribe filters. Added write protection, safety fields, and schedule sync guard documentation. Fixed Home/Away section direction errors (`auto_away_enable`, `auto_away_reset`, `home_away_input` are bidirectional, not device-only). |
| 1.9 | 2026-02-08 | Added Temperature schedules section: schedule bucket format, complete JSON schema (version 2), day indexing (0=Monday), setpoint fields, temperature encoding (always Celsius), schedule modes, push/read flows, mode switching via shared bucket, continuation setpoints, custom schedules, debounce behavior (15-second sliding window), timestamp rejection, pending local change handling, auto-schedule learning caveat. Updated implementation checklist with schedule-related items. |
| 1.8 | 2026-02-07 | Added "Batching multiple pushes" section under Server implementation notes. Explains how to send multiple chunks on a single subscribe connection using the device's 5-second closing timer reset behavior. |
| 1.7 | 2026-02-07 | Corrected Home/Away mode section — use `manual_eco_all` + `manual_eco_timestamp` instead of `away` field. The `away` field triggers auto-eco which is overridden by schedule preconditioning. Added timestamp requirement (600s staleness window). Added guidance on field conflicts and exiting eco mode. |
| 1.6 | 2026-02-07 | Added Home/Away mode section: structure bucket controls away state, device bucket away fields are read-only, eco temperature behavior, structure key selection for claimed vs unclaimed devices. Enhanced structure bucket `away` field description. |
| 1.5 | 2026-02-07 | Documented `X-nl-client-id` header format (`d.{SERIAL}.{random}`). Added entry key polling note (server must return same unexpired key). Fixed broken anchor links for defer-device-window. Fixed `if_object_revision` contradiction (now consistently "implementation-defined"). Fixed subscribe example to use `objects` array format. |
| 1.4 | 2026-02-06 | Clarified 403/404 trigger comms reset (not just logging). |
| 1.3 | 2026-02-06 | Marked `X-nl-longest-wake` header as vestigial (server ignores, never resets). |
| 1.2 | 2026-02-05 | Added Authentication section (provisional) with device identification, credential types, credential loop warning, and recommended home server approach. Added Pairing section with user bucket mechanism. Fixed entry key `expires` type (must be JSON number, not string). Added user bucket to bucket types. Added device reboot note for full state sync. Expanded POST /entry response fields. Added device_alert_dialog bucket. |
| 1.1 | 2026-02-05 | Added `if_object_revision` conditional write documentation. Clarified closing timer reset behavior and 7-second timeout context. Added JSON library field ordering implementation note. |
| 1.0 | 2026-02-04 | Initial release. Subscribe, PUT, and entry endpoints. Response headers. Timing reference. Battery behavior. Bucket types. URL port requirement appendix. |
