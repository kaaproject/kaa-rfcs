---
name: Configuration Management Extension
shortname: 3/CMX
status: raw
editor: Dmitry Sergeev <dsergeev@cybervisiontech.com>
---

## Introduction

The Configuration Management Extension is a [Kaa Protocol](/0002-kaa-protocol/README.md) extension.

It is intended to manage endpoint configuration distribution.

## Requirements and constraints
### Problems and possible solutions

1. _Multiple configurations available for client._ It is possible that there will be more than one available configurations for endpoint (e.g.: endpoint was offline while new configurations were added).
   Solution:
   - Only latest configuration should be sent to an endpoint to reduce amount of traffic.
_Note:_ Approach for delivering firmware updates over the air should be different.

2. _The server should know if a configuration has been delivered._ 

   Solutions:
   - Send PUBLISH message from client to "/status" topic. See below sections for details.

## Use cases

### UC1
Configuration delivery by request. The endpoint should be able to receive latest configuration from CMX by request.

### UC2
Configuration delivery as a reaction on EP lifecycle event. The endpoint should receive latest configuration when it connects to a server if this configuration hadn't applied yet.

## Design

### Request/response
The extension uses request/response pattern. A response from client is sent to confirm delivery.

For MQTT, responses MUST be published at `<request_path>/status`. Each response MUST be published with the same QoS as the corresponding request.

### Formats
#### Schemeless JSON
The server should support arbitrary JSON records as data points at the following resource path:
```
<endpoint_token>/json
```

The payload should be a JSON-encoded object with the following fields:
- `id` - id of the message. Should be either string or number. Used in delivery confirmation process.
- `entries` (required) - an array of data entries. Each one of the entries can be of any JSON type.

Example:
```json
{
  "id": 42,
  "entries": [
    { "key": "value" },
    15,
    [ "an", "array", 13 ]
  ]
}
```

```json
{
  "entries": [ 15 ]
}
```

_Note: we might have used MQTT packet id, but in that case we lose ability to work via gateways as a gateway may change MQTT packet id._

A delivery confirmation response is a JSON record with the following fields:
- `id` a copy of the `id` field from the corresponding request.
- `status` a human-readable string explaining the cause of an error (if any). In case processing was sucessful, it is `"ok"`.
