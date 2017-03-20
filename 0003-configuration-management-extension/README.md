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
##### Use case 1
The server should be able to listen for configuration request at the following resource path:
```
<endpoint_token>/config/request/json
```

The payload should be a JSON-encoded object with the following fields:
- `id` (required) - id of the message. Should be either string or number. Used in delivery confirmation process.
- `currentConfigVersion` - current endpoint configuration version. Should be of integer type. Used by server to determine is it needed to send response with configuration, server can not send configuration record if latest available configuration version equals to the one sent by endpoint. If this field was excluded from the message, then server will send configuration anyway.

Example:
```json
{
  "id": 42,
  "currentConfigVersion": 1,
  "entries": [
    { "key": "value" },
    15,
    [ "an", "array", 13 ]
  ]
}
```

A server response is a JSON record with the following fields:
- `id` a copy of the `id` field from the corresponding request.
- `configVersion` (required) - version of configuration that is included into the message. If there's no new configuration versions and config body isn't included into message, then value of this field will equal to the one provided by endpoint.
- `status` a human-readable string explaining the cause of an error (if any). In case that request was sucessful, it is `"ok"`.
- `entries` - an array of configuration entries. Each one of the entries can be of any JSON type.

Example:
```json
{
  "id": 42,
  "configVersion": 2,
  "status": "ok",
  "entries": [
    { "key": "value" },
    15,
    [ "an", "array", 13 ]
  ]
}
```

Example for case when there's no new configuration version for endpoint:
```json
{
  "id": 42,
  "configVersion": 1,
  "status": "ok"
}
``` 

##### Use case 2
The server should be able to publish endpoint configuration updates at the following resource path:
```
<endpoint_token>/config/distribution/json
```


The payload is a JSON record with the following fields:
- `id` a copy of the `id` field from the corresponding request.
- `configVersion` (required) - version of configuration that is included into the message.
- `status` a human-readable string explaining the cause of an error (if any). In case that request was sucessful, it is `"ok"`.
- `entries` (required) - an array of configuration entries. Each one of the entries can be of any JSON type.

Example:
```json
{
  "id": 42,
  "configVersion": 2,
  "status": "ok",
  "entries": [
    { "key": "value" },
    15,
    [ "an", "array", 13 ]
  ]
}
```

_Note: we might have used MQTT packet id, but in that case we lose ability to work via gateways as a gateway may change MQTT packet id._

A delivery confirmation response is a JSON record with the following fields:
- `id` a copy of the `id` field from the corresponding request.
- `status` a human-readable string explaining the cause of an error (if any). In case processing was sucessful, it is `"ok"`.

## Glossary

- _Endpoint (short: EP)_ — end device that produces data. The user is interested in differentiating all endpoints. Endpoint can be virtual.
- _EP lifecycle event_ — message inside server infrastructure that notifies services that subscribed to corresponding topic about endpoint state changes (e.g.: "connected", "disconnected", etc)
