# Server Subscribe Flow Design

**Author:** Chris Serio (cjserio)
**Date:** February 2026

This document describes how to implement a server that properly pairs with the Nest client firmware's subscribe/long-poll flow. The goal is to maximize device sleep time while ensuring timely data delivery.

---

## 1. Overview / TL;DR

### When you receive a subscribe request:

```
1. Parse the request body (get client's bucket revisions)
2. Compare each bucket's revision against your server state
3. Decision point:

   IF you have newer data (server_revision > client_revision):
       -> Send response immediately (headers + JSON body + close)

   IF you have NO newer data (revisions match):
       -> Don't respond yet
       -> Hold the connection open (no headers, no body, nothing)
       -> Wait...
```

### While holding:

```
WAIT FOR ONE OF:

   A) New data arrives (user action, MQTT push, another device, etc.)
       -> Send response (headers + JSON body + close)

   B) Timeout approaching (~80% of your X-nl-suspend-time-max)
       -> Send tickle (headers + empty body + close)

   C) Connection drops (client disconnected)
       -> Clean up, do nothing
```

### The key insight:

You send NOTHING until you have something to say. The connection just sits there open. The client is asleep. When you finally respond (data or tickle), you send headers + body + close all at once.

---

## 2. Request Format (What the Device Sends)

### Endpoint
```
POST /{version}/subscribe HTTP/1.1
```

### Headers
```http
Content-Type: application/json
Authorization: Basic {base64(user:password)}
X-nl-protocol-version: 1
X-nl-longest-wake: {seconds}          # Device's avg awake time (analytics only)
X-nl-device-swversion: {firmware}
X-nl-client-id: {device_id}           # Only if no session yet
```

### Body
```json
{
    "chunked": true,
    "session": "{session_id}",
    "objects": [
        {
            "object_key": "device.{device_id}",
            "object_revision": 12345,
            "object_timestamp": 1700000000000
        },
        {
            "object_key": "shared.{structure_id}",
            "object_revision": 6789,
            "object_timestamp": 1700000000001
        }
    ]
}
```

**Key points:**
- `"chunked": true` - Device is requesting long-poll mode
- `object_revision` - Device's current revision for each bucket
- `object_timestamp` - Last update timestamp for each bucket

---

## 3. Response Format (What the Server Sends)

### Response Headers
```http
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
X-nl-suspend-time-max: {seconds}
```

**Critical header: `X-nl-suspend-time-max`**
- This tells the device how long it can sleep while waiting
- Device sets a wake timer at `request_time + X-nl-suspend-time-max`
- Typical value: 60-300 seconds depending on your latency requirements
- Lower = faster updates, more power usage
- Higher = slower updates, better battery life

### Response Body (When Updates Exist)
```json
{
    "objects": [
        {
            "object_key": "device.{device_id}",
            "object_revision": 12346,
            "object_timestamp": 1700000001000,
            "value": {
                "current_temperature": 21.5,
                "target_temperature": 22.0,
                "hvac_mode": "heat"
            }
        }
    ]
}
```

**CRITICAL: `object_key` MUST appear before `value`** - the client uses simple string parsing with `strstr()` to find `"object_key"` for bucket routing before parsing the value.

### Response Body (Tickle - No Updates)
```
(empty body or Content-Length: 0)
```

A "tickle" is an empty response that:
- Confirms the server is alive
- Resets the connection timeout
- Allows the device to continue sleeping
- Does NOT wake the device for processing

---

## 4. Server Implementation Guide

### Pseudocode

```python
def handle_subscribe(request, connection):
    client_buckets = parse_subscribe_request(request)

    # Check for immediate updates
    updates = get_pending_updates(client_buckets)

    if updates:
        # Have data - respond immediately
        send_response(connection, updates)
        return

    # No data - hold connection
    timeout = time.now() + (X_NL_SUSPEND_TIME_MAX * 0.8)

    while time.now() < timeout:
        # Check for new data (poll DB, check queue, etc.)
        updates = check_for_new_updates(client_buckets)

        if updates:
            send_response(connection, updates)
            return

        if connection.is_closed():
            return  # Client disconnected

        time.sleep(0.1)  # Small poll interval

    # Timeout - send tickle
    send_tickle(connection)


def send_response(connection, updates):
    connection.send("HTTP/1.1 200 OK\r\n")
    connection.send("Content-Type: application/json\r\n")
    connection.send("Transfer-Encoding: chunked\r\n")
    connection.send("X-nl-suspend-time-max: 120\r\n")
    connection.send("\r\n")

    body = json.dumps({"objects": updates})
    connection.send_chunk(body)
    connection.send_final_chunk()
    connection.close()


def send_tickle(connection):
    connection.send("HTTP/1.1 200 OK\r\n")
    connection.send("Content-Type: application/json\r\n")
    connection.send("Content-Length: 0\r\n")
    connection.send("X-nl-suspend-time-max: 120\r\n")
    connection.send("\r\n")
    connection.close()
```

### Decision Logic Summary

| Condition | Action |
|-----------|--------|
| Bucket revision changed | Send update immediately |
| ~80% of timeout elapsed, no changes | Send tickle |
| Connection error | Close, device will reconnect |
| Device sends new subscribe | Process new request |

### State Machine Diagram

```
                     +----------------+
                     |    WAITING     |<--------------------------+
                     |  FOR DEVICE    |                           |
                     +-------+--------+                           |
                             |                                    |
                             | Device sends POST /subscribe       |
                             v                                    |
                     +-------+--------+                           |
                     |     CHECK      |                           |
                     |    BUCKETS     |                           |
                     +-------+--------+                           |
                             |                                    |
            +----------------+----------------+                    |
            |                                 |                    |
            v                                 v                    |
    +-------+--------+               +--------+-------+           |
    |  HAS UPDATES   |               |  NO UPDATES    |           |
    | (server_rev >  |               | (revisions     |           |
    |  client_rev)   |               |    match)      |           |
    +-------+--------+               +--------+-------+           |
            |                                 |                    |
            |                                 v                    |
            |                        +--------+-------+           |
            |                        |      HOLD      |<------+   |
            |                        |   CONNECTION   |       |   |
            |                        +--------+-------+       |   |
            |                                 |               |   |
            |              +------------------+--------+      |   |
            |              |                           |      |   |
            |              v                           v      |   |
            |      +-------+------+          +---------+----+ |   |
            |      |   TIMEOUT    |          |  NEW DATA    | |   |
            |      |  APPROACHING |          |  AVAILABLE   | |   |
            |      | (80% of max) |          +------+-------+ |   |
            |      +------+-------+                 |         |   |
            |             |                        |         |   |
            |             v                        |         |   |
            |      +------+-------+                |         |   |
            |      |    SEND      |----------------+         |   |
            |      |   TICKLE     |                          |   |
            |      | (empty body) |                          |   |
            |      +------+-------+                          |   |
            |             |                                  |   |
            |<------------+----------------------------------+   |
            |                                                    |
            v                                                    |
    +-------+--------+                                           |
    |     SEND       |                                           |
    |    UPDATES     |                                           |
    |   (chunked)    |                                           |
    +-------+--------+                                           |
            |                                                    |
            v                                                    |
    +-------+--------+                                           |
    |     CLOSE      |                                           |
    |   CONNECTION   |-------------------------------------------+
    +----------------+
```

---

## 5. Timing and Timeouts

### Timeline with data:
```
T=0:     Device sends POST /subscribe with its revisions
T=0:     Server checks - no new data - holds connection (sends nothing)
T=0:     Device goes to sleep (connection open, wake timer set)
         ...server holds...
         ...device sleeps...
T=45:    User changes temperature via phone app
T=45:    Server sees change, sends response, closes connection
T=45:    Device wakes, processes update, goes back to sleep
T=45:    Device sends new POST /subscribe
         ...repeat...
```

### Timeline with tickle (no data):
```
T=0:     Device sends POST /subscribe
T=0:     Server checks - no new data - holds connection
T=96:    Server approaching timeout (80% of 120s) - sends tickle, closes
T=96:    Device wakes briefly, sees tickle, goes back to sleep
T=96:    Device sends new POST /subscribe
         ...repeat...
```

### Server-Side Timers

| Timer | Recommended Value | Purpose |
|-------|-------------------|---------|
| X-nl-suspend-time-max | 60-300 seconds | Max time device will sleep |
| Tickle threshold | 80% of max | When to send keep-alive |
| Connection timeout | max + 30 seconds | When to give up on connection |

### Client-Side Timers (for reference)

| Timer | Value | Purpose |
|-------|-------|---------|
| Closing timer | 5 seconds | After data arrives, prevents sleep briefly |
| Chunked override | ~14 seconds | Max connection duration before forced sleep |
| Poll interval | 2 seconds | How often client checks state machine |

---

## 6. Chunked Encoding Details

**CRITICAL: The client does NOT support streaming multiple updates on a single connection.**

Despite using HTTP chunked transfer encoding, the client treats each connection as delivering exactly ONE response.

### What "Chunked" Actually Means Here

The "chunked" in the client's vocabulary means **long-poll mode**, NOT streaming multiple responses:

1. Client sends `"chunked": true` in subscribe request
2. Server responds with `Transfer-Encoding: chunked`
3. Server can **hold the connection open** waiting for changes
4. When there's something to send, server sends **one complete JSON response**
5. Server **closes the connection**
6. Client processes the response, then sends a new subscribe

**This is NOT streaming.** Chunked encoding is used for:
- Holding the connection open (long-polling)
- Not having to pre-calculate Content-Length
- Allowing the server to send headers immediately while data accumulates

### The 5-Second Data Timeout

The client has a critical timer that affects server behavior:

```
When body data arrives:
  -> startClosingTimer() called
  -> 5-second timer starts (or resets if already running)

If no data arrives for 5 seconds:
  -> Timer fires
  -> Connection is ABORTED by client
  -> Device goes to sleep
```

**Server implication:** If you're sending chunked data and pause for >5 seconds between chunks, the client will abort. Don't send data slowly over many seconds - batch it and send quickly.

### Correct: How to Send Updates

```python
def send_subscribe_update(connection, update_data):
    # Send all headers
    connection.send_headers({
        'HTTP/1.1': '200 OK',
        'Content-Type': 'application/json',
        'Transfer-Encoding': 'chunked',
        'X-nl-suspend-time-max': '120'
    })

    # Send complete JSON body as chunked
    json_body = json.dumps(update_data)
    connection.send_chunk(json_body)

    # Send final chunk and close
    connection.send_final_chunk()  # Sends "0\r\n\r\n"
    connection.close()
```

### Correct: How to Send Tickle

```python
def send_tickle(connection):
    # Option 1: Chunked encoding with empty body
    connection.send_headers({
        'HTTP/1.1': '200 OK',
        'Content-Type': 'application/json',
        'Transfer-Encoding': 'chunked',
        'X-nl-suspend-time-max': '120'
    })
    connection.send_final_chunk()  # Just "0\r\n\r\n"
    connection.close()

    # Option 2: Content-Length: 0
    connection.send_headers({
        'HTTP/1.1': '200 OK',
        'Content-Type': 'application/json',
        'Content-Length': '0',
        'X-nl-suspend-time-max': '120'
    })
    connection.close()
```

### WRONG: What NOT to Do

```python
# DON'T DO THIS - Client will abort after 5 seconds of no data
def stream_updates_wrong(connection, updates):
    connection.send_headers({'Transfer-Encoding': 'chunked'})

    for update in updates:
        connection.send_chunk(json.dumps(update))
        time.sleep(10)  # BAD! Client aborts after 5s silence

    connection.close()

# DON'T DO THIS - Client can't parse multiple JSON documents
def multi_json_wrong(connection):
    connection.send_headers({'Transfer-Encoding': 'chunked'})
    connection.send_chunk('{"objects":[{...}]}')  # First update
    time.sleep(1)
    connection.send_chunk('{"objects":[{...}]}')  # Second update - NOT PARSED!
    connection.close()
```

### How the Client Processes Data

```
HTTP library receives chunks
    |
    v
Accumulates all chunk data into buffer
    |
    v
Connection closes (final chunk received)
    |
    v
HandleDone() called with COMPLETE buffer
    |
    v
ProcessResponse() parses COMPLETE JSON
    |
    v
nlJSONParser_ParseDocument(complete_json_string)
```

The client's JSON parser (`nlJSONParser_ParseDocument`) expects a single, complete JSON document. It does NOT:
- Stream-parse JSON
- Handle multiple JSON documents concatenated
- Parse partial JSON

### Handling Rapid Successive Updates

If multiple changes happen quickly while holding a connection:

```python
def handle_subscribe_with_batching(connection, client_revisions):
    pending_updates = []
    timeout = calculate_timeout()  # 80% of X-nl-suspend-time-max

    while not timeout_reached():
        # Check for new updates
        new_updates = check_for_updates(client_revisions)
        pending_updates.extend(new_updates)

        # Small delay to batch rapid updates
        time.sleep(0.1)

        # If we have updates, send them all at once
        if pending_updates:
            send_subscribe_update(connection, {
                'objects': pending_updates
            })
            return

    # Timeout - send tickle
    send_tickle(connection)
```

### The JSON Field Order Requirement

**CRITICAL:** The `object_key` field MUST appear before `value` in each object:

```json
// CORRECT - object_key before value
{
    "objects": [{
        "object_key": "device.xyz",
        "object_revision": 12346,
        "object_timestamp": 1700000000000,
        "value": { "temperature": 21.5 }
    }]
}

// WRONG - value before object_key (parser may fail)
{
    "objects": [{
        "value": { "temperature": 21.5 },
        "object_key": "device.xyz",
        "object_revision": 12346
    }]
}
```

---

## 7. PUT Handling (Device Sending Data to Server)

### Endpoint
```
POST /{version}/put HTTP/1.1
```

### Request Body
```json
{
    "objects": [
        {
            "object_key": "device.{device_id}",
            "base_object_revision": 12345,
            "value": {
                "target_temperature": 23.0
            }
        }
    ]
}
```

**Note:** PUT uses `base_object_revision` for optimistic concurrency. Server should:
1. Check if `base_object_revision` matches current server revision
2. If match: apply update, increment revision, return 200
3. If mismatch: return 409 Conflict (or apply anyway depending on your policy)

### Server Response
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "objects": [
        {
            "object_key": "device.{device_id}",
            "object_revision": 12346
        }
    ]
}
```

---

## 8. Multi-Device Considerations

### Per-Device State

Each connected device needs:
```
{
    device_id: string,
    connection: HttpConnection,
    subscribed_buckets: Map<bucket_key, client_revision>,
    last_activity: timestamp,
    session_id: string
}
```

### Broadcasting Updates

When a bucket changes (e.g., user changes temperature):

```
1. Update bucket in database with new revision
2. For each connected device subscribed to this bucket:
   a. If device's revision < new revision:
      - Push update on their connection
3. For devices NOT currently connected:
   - Update will be sent on their next subscribe
```

### Handling Multiple Devices for Same Structure

Devices in the same home (structure) share buckets like `shared.{structure_id}`. When one device changes something:

```
Device A changes target_temperature
  -> Server updates shared.{structure_id}, revision++
  -> Server pushes to Device B (if connected and subscribed)
  -> Server pushes to Device C (if connected and subscribed)
  -> Device A gets update on next subscribe (or via PUT response)
```

---

## 9. Error Handling

### HTTP Status Codes to Return

| Status | When | Client Behavior |
|--------|------|-----------------|
| 200 | Success | Process response |
| 301/302 | Redirect | Follow redirect URL |
| 400 | Bad request | Retry up to 3 times |
| 401 | Auth failed | Try default creds, then reset |
| 403 | Forbidden | Reset comms |
| 404 | Not found | Reset comms |
| 420 | Rate limited | Retry with backoff |
| 5xx | Server error | Reset comms |

---

## Summary

The client is built to sleep. Your job as the server is to let it sleep as much as possible while still delivering updates promptly when they occur.

1. **Accept subscribe** - Parse bucket list and revisions from device
2. **Compare revisions** - Check each bucket against server state
3. **Push immediately** - If any bucket has newer data, send it now
4. **Hold if no updates** - Keep connection open while device sleeps (send nothing!)
5. **Push when data arrives** - External changes trigger immediate push
6. **Tickle before timeout** - Send empty response at ~80% of max time
7. **Respect the sleep goal** - The protocol is designed for sleeping devices; don't fight it
