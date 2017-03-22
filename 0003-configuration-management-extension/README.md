---
name: Configuration Management Extension
shortname: 3/CMX
status: raw
editor: Dmitry Sergeev <dsergeev@cybervisiontech.com>
---

## Introduction

The Configuration Management Extension is a [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

It is intended to manage endpoint configuration distribution.

## Requirements and constraints
### Problems and possible solutions

1. _Multiple configurations available for client._ It is possible that there will be more than one available configurations for endpoint (e.g., endpoint was offline while new configurations were added).
   
   Solution:
   - Only latest configuration should be sent to an endpoint to reduce amount of traffic.

2. _The server should know if a configuration has been delivered._ 

   Solutions:
   - Send PUBLISH message from client to "/status" topic. See below sections for details.

## Use cases

### UC1
Configuration delivery by request. The endpoint should be able to receive latest configuration from CMX by request.

### UC2
Configuration delivery that is initiated by the server. The endpoint should receive latest configuration when it connects to a server if this configuration hadn't applied yet.

## Design

### Request/response
The extension uses request/response pattern. A response from client is sent to confirm delivery.

For MQTT, responses MUST be published at `<request_path>/status`. Each response MUST be published with the same QoS as the corresponding request.

### Formats
#### Schemeless JSON
##### Use case 1
The server should be able to listen for configuration request at the following resource path:
```
<endpoint_token>/pull/json
```

The payload should be a JSON-encoded object with the following fields:
- `id` (required) - id of the message. Should be either string or number. Used in delivery confirmation process.
- `configVersion` (optional) - current endpoint configuration version. Should be of integer type.
- `requiredConfigVersion` (optional) - endpoint configuration version that is requested by endpoint. This provides ability to re-send configuration for endpoints that cannot persist the configuration and provides mechanism to request previous configuration versions (roll-back).
If there's no `requiredConfigVersion` field in message, then server should use `configVersion` to determine is it needed to send response with configuration, server must not send configuration record if latest available configuration version equals to the one sent by endpoint. If `configVersion` field was excluded from the message, then server will send configuration anyway

Example:
```json
{
  "id": 42,
  "configVersion": 1,
  "requiredConfigVersion": 2
}
```
Example 2:
```json
{
  "id": 42,
  "configVersion": 3,
  "requiredConfigVersion": 2
}
```
Example 3:
```json
{
  "id": 42,
  "configVersion": 1,
}
```
Example 4:
```json
{
  "id": 42
}
```

A server response is a JSON record with the following fields:
- `id` (required) a copy of the `id` field from the corresponding request.
- `configVersion` (required) - version of configuration that is included into the message. If there's no new configuration versions and config body isn't included into message, then value of this field will equal to the one provided by endpoint.
- `status` (required) a human-readable string explaining the cause of an error (if any). In case that request was sucessful, it is `"ok"`.
- `config` (optional) - configuration body of an arbitrary JSON type.
Destination topic is 
```
<endpoint_token>/pull/json/status
```

Example:
```json
{
  "id": 42,
  "configVersion": 2,
  "status": "ok",
  "config": {
    "key": "value",
    "array" : [
        "value2"
    ]
  }
}
```

Example for the case when there's no new configuration version for endpoint:
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
<endpoint_token>/push/json
```


The payload is a JSON record with the following fields:
- `id` (required) id of the message. Should be either string or number. Used in delivery confirmation process.
- `configVersion` (required) - version of configuration that is included into the message.
- `config` (required) - configuration body of an arbitrary JSON type.

Example:
```json
{
  "id": 42,
  "configVersion": 2,
  "config": [
    { "key": "value" },
    15,
    [ "an", "array", 13 ]
  ]
}
```

_Note: we might have used MQTT packet id, but in that case we lose ability to work via gateways as a gateway may change MQTT packet id._

A delivery confirmation response is a JSON record with the following fields:
- `id` (required) a copy of the `id` field from the corresponding request.
- `status` (required) a human-readable string explaining the cause of an error (if any). In case processing was sucessful, it is `"ok"`.
The destination topic is 
```
<endpoint_token>/push/json/status
```
Example:
```json
{
  "id": 42,
  "status": "ok"
}
```