# Hazomatic Sensor Protocol (Current Implementation - 16 Feb 2026)

This document describes the current on-device protocol expected by the Hazomatic Sensor app, based on the existing Swift implementation. It is not an idealized spec; it reflects what the app actually uses today.

## Overview

A sensor is expected to expose:

- HTTP endpoints on port 80 for basic JSON reads and identification.
- WebSocket endpoint on port 81 for streaming JSON readings and command control.

The app treats each sensor as a device with a unique ID, a location, and a network address. The core data model is `SPS30Output`.

## Discovery (mDNS / Bonjour)

The app browses the local network for HTTP services and filters by instance name.

- Service type: `_http._tcp` in domain `local.`
- Instance name must start with `hazomatic` (case-insensitive).
- The app constructs the host as `<instanceName>.<domain>` (typically `hazomatic42.local`).
- The service is resolved to an IPv4 address and port using `NetService`.

If discovery succeeds, the app creates a `Sensor` with:

- `mDNSName`: the `<instanceName>.<domain>` host string
- `ipAddress`: the resolved IPv4 address
- `id`: the uuid from the sps30 or other sensor - this should not change, and must persist through sensor power cycles.

The advertised port is used only for tracking the discovered HTTP URL; the app still targets port 80 for HTTP and port 81 for WebSocket connections.

Note: The app triggers the local network permission prompt by opening a TCP connection to `hazomatic.local:8000` during discovery.

## Network Endpoints

### HTTP (port 80)

Base URL pattern (used for direct reads and ID lookups):

- `http://<ip>:80/read`
- `http://<ip>:80/selfclean`
- `http://<ip>:80/ip`

#### GET /read

Returns a single JSON reading payload matching `SPS30Output`.

- Status code must be 2xx on success.
- Response body must be valid JSON.

#### GET /selfclean

Triggers the sensor self-cleaning routine.

- Status code must be 2xx on success.
- Response body is currently ignored by the app.

#### GET /ip

Returns a JSON object with the sensor network identity.

Expected JSON shape (from `SensorID`):

```json
{
    "ip": "192.168.1.134",
    "host": "hazomatic42",
    "uuid": "SENSOR-UUID-STRING"
}
```

- Status code must be 2xx on success.
- The app extracts `uuid` and uses it as the sensor ID.

### WebSocket (port 81)

WebSocket URL pattern:

- `ws://<ip>:81/`

The WebSocket is used for:

- Receiving streaming sensor readings.
- Sending JSON command messages to the sensor.
- Keeping a heartbeat (the app sends WebSocket ping frames every 8 seconds).

The app considers the sensor online while the WebSocket is connected and ping/pong is healthy.

## Connection Lifecycle

1. Optional discovery via mDNS (see above).
2. Optional sensor ID lookup via `GET /ip` to retrieve a stable `uuid`.
3. WebSocket connection to `ws://<ip>:81/`.
4. App sends a JSON `{\"cmd\":\"ping\"}` and begins WebSocket ping frames.
5. Sensor pushes JSON readings over the socket or responds to `{\"cmd\":\"read\"}`.

## WebSocket Commands (App -> Sensor)

Commands are sent as JSON objects with a `cmd` field.

### Request a reading

```json
{ "cmd": "read" }
```

The sensor should respond by sending a reading JSON payload (see `SPS30Output`) over the WebSocket.

### Ping

```json
{ "cmd": "ping" }
```

This is an application-level ping (not the WebSocket ping frame). The app currently does not parse a response. You may ignore it or return a simple acknowledgment.

### Self-clean

```json
{ "cmd": "selfclean" }
```

Triggers sensor self-cleaning. The app does not parse the response; you may send a confirmation string or ignore.

## Reading Payload: SPS30Output

The app expects JSON matching this structure. This is used for:

- WebSocket streaming reads.
- HTTP `/read` responses.

### Expected JSON keys

```json
{
    "version": "1.0",
    "serial": "SENSOR-UUID-STRING",
    "pm1.0": 1.23,
    "pm2.5": 4.56,
    "pm4.0": 7.89,
    "pm10.0": 12.34,
    "nc0.5": 100.0,
    "nc1.0": 80.0,
    "nc2.5": 60.0,
    "nc4.0": 40.0,
    "nc10.0": 20.0,
    "typical_particle_size": 0.6
}
```

### Field notes

- `serial` is used as the sensor ID in the app.
- `pm1.0`, `pm2.5`, `pm4.0`, `pm10.0` map to mass concentration values.
- `nc0.5`, `nc1.0`, `nc2.5`, `nc4.0`, `nc10.0` map to number concentration values.
- `typical_particle_size` is a floating-point value.
- All numeric values are decoded as `Double`.

## Error Handling Expectations

The app treats the following as errors:

- Non-2xx HTTP status codes.
- Invalid or non-JSON responses.
- JSON that does not match the `SPS30Output` schema.

For WebSocket, messages that do not decode to `SPS30Output` are ignored, but their raw text may be logged for debugging.

## Timing and Retries

- WebSocket connect request timeout: 8 seconds.
- WebSocket heartbeat: ping frames every 8 seconds.
- Sensor ID lookup (`/ip`): up to 3 attempts with exponential backoff (1s, 2s, 4s; capped at 8s).
- HTTP requests use a 6-second timeout where explicitly set.

## Port Summary

- HTTP: port 80
- WebSocket: port 81

## Minimal Implementation Checklist

- HTTP GET `/read` returns valid `SPS30Output` JSON.
- HTTP GET `/ip` returns valid `SensorID` JSON with `uuid`.
- HTTP GET `/selfclean` returns 2xx.
- WebSocket endpoint accepts JSON commands and emits `SPS30Output` JSON.
- WebSocket responds to ping frames (standard WebSocket pong).
