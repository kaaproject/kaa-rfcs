---
name: Configuration Management Extension
shortname: 7/CMX
status: raw
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
---

# Introduction

The Configuration Management Extension is a [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

It is intended to manage endpoint configuration distribution.

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

# Requirements and constraints
## Problems and possible solutions

1. _The server should know if a configuration has been applied by the endpoint._ <!-- TODO: why? add more reasoning -->

   Possible solutions:
   - The client sends a reply to the new configuration provided by the server.

     This means the server is initiating a request. That does not work well for HTTP, CoAP, and other protocols that only support client-initiated requests, and is not recommended by 1/KP.

   - Introduce `/applied` resource, so the client sends an additional request once configuration is applied.

2. _An endpoint might be interested in a part of the general configuration only._

    Possible solutions:
    - Allow subscribing to a part of configuration. (e.g., using [JSONPath](http://goessner.net/articles/JsonPath/), or JSON Pointer defined in [RFC 6901](https://tools.ietf.org/html/rfc6901).)

3. _Difference between subsequent configurations might be small, so sending the whole configuration is inefficient._

    Possible solutions:
    - Send only updates. (e.g., using JSON Patch format, defined in [RFC 6902](https://tools.ietf.org/html/rfc6902).)

4. _Endpoint can actively reject a configuration._

   Examples are:
   - the endpoint is unable to process the configuration.
   - the configuration is ill-formated, or pre-conditions are not met.

   Possible solutions:
   - Send the status result of the configuration application (either success or failure) back to the server.

# Use cases

## UC1: Configuration request
Configuration delivery by request. The endpoint should be able to receive latest configuration from CMX by request.

## UC2: Configuration push
Configuration delivery is initiated by the server. The endpoint should receive latest configuration when it connects to the server if this configuration wasn't applied yet.

# Design
The extension follows client-initiated request/response pattern described in 1/KP, which nicely maps to both MQTT and CoAP.

There are two resources provided by the extension.

- `/<endpoint_token>/config/json` is used for subscribing and receiving a config.
- `/<endpoint_token>/applied/json` is used for notifiying server of the currently active configuration.

## Configuration identifier
Configuration identifier is an opaque string, which uniquely identifies the configuration. The client SHOULD NOT assume the nature of identifiers.

## Absent configuration
If an endpoint does not have a configuration assigned, its configuration is `null`, and the corresponding configuration identifier is an empty string (`""`).

## Request/response
The extension uses request/response pattern described in [1/KP](/0001-kaa-protocol/README.md) with observe extension. For MQTT, it is achieved by publishing multiple responses to the response topic; for CoAP, it is achieved with the `Observe` option defined in [RFC 7641](https://tools.ietf.org/html/rfc7641).

## Config resource
Config resource is used for distributing the latest configuration to endpoints.

The config resource is a request/response resource with an observe extension which resides at the following resource path:
```
/<endpoint_token>/config
```

### Request
Request payload MUST be a UTF-8 encoded object with the JSON Scheme as defined in [config-request.schema.json](./config-request.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "7/CMX config request schema",

  "type": "object",
  "properties": {
    "configId": {
      "type": "string",
      "description": "Identifier of the currently applied configuration."
    },
    "observe": {
      "type": "boolean",
      "description": "Whether the endpoint is interested in observing its configuration."
    }
  },
  "additionalProperties": false
}
```

If `configId` field is missing, the server MUST respond with the current configuration for the given endpoint.

If `configId` field is present, the server MUST send new configuration only if `configId` differs from the identifier of the currently assigned cofiguration for the endpoint.

If `observe` field is present and is `true`, the server MUST send new configuration to the endpoint whenever configuration changes.

If `observe` field is present and is `false`, the server MUST NOT send new configurations to the endpoint without a further explicit request.

The server MAY send configurations to the endpoint when no request were done before, or `observe` was not present in a request.

If the current config matches one specified by `configId` in the request, the server MUST return response with both `configId` and `config` absent.

Examples:
- get current configuration only
  ```json
  {
    "observe": false
  }
  ```
- endpoint's current config ID is `97016dbe8bb4adff8f754ecbf24612f2`.
  ```json
  {
    "configId": "97016dbe8bb4adff8f754ecbf24612f2"
  }
  ```
- subscribe to all configuration updates
  ```json
  {
    "observe": true
  }
  ```

### Response
Response payload MUST be a UTF-8 encoded object with the JSON Scheme as defined in [config-response.schema.json](./config-response.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "7/CMX config response schema",

  "type": "object",
  "properties": {
    "configId": {
      "type": "string",
      "description": "Identifier of the current configuration."
    },
    "config": {
      "description": "Configuration body of an arbitrary type."
    }
  },
  "additionalProperties": false
}
```

`configId` and `config` MUST come in a pair. If `configId` and `config` are absent in the response, that means configuration hasn't changed.

If an endpoint successfully applies provided configuration, it SHOULD notify the server via `/applied` resource.

Examples:
- configuration hasn't changed
  ```json
  {
  }
  ```

- new configuration
  ```json
  {
    "configId": "97016dbe8bb4adff8f754ecbf24612f2",
    "config": {
      "mode": "AP",
      "ssid": "Smart Teapot",
      "password": "acupofteaplease",
      "security": "WPA2_PSK"
    }
  }
  ```

## Applied resource
Applied resource is used by endpoints to notify the server of successfully applied configuration.

The applied resource is a request/response resource with the following resource path:
```
/<endpoint_token>/applied
```

### Request

The request is a UTF-8 encoded JSON object with the JSON Schema as defined in [applied-request.schema.json](./applied-request.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "7/CMX applied request schema",

  "type": "object",
  "properties": {
    "configId": {
      "type": "string",
      "description": "Identifier of the applied configuration."
    },
    "statusCode": {
      "type": "number",
      "description": "Status code based on HTTP status codes.",
      "default": 200
    },
    "reasonPhrase": {
      "type": "string",
      "description": "A human-readable string explaining the cause of an error (if any)."
    }
  },
  "required": [ "configId" ],
  "additionalProperties": false
}
```

A request means the endpoint has received the given configuration and tried to apply it.

`statusCode` represents a result of appliyng the configuration (either success or failure). If result is a failure, `reasonPhrase` SHOULD include a reason of the failure.

Examples:
- configuration successfully applied:
  ```json
  {
    "configId": "97016dbe8bb4adff8f754ecbf24612f2"
  }
  ```

- configuration is rejected
  ```json
  {
    "configId": "97016dbe8bb4adff8f754ecbf24612f2",
    "statusCode": 400,
    "reasonPhrase": "WPA2 is not supported"
  }
  ```

### Response

The response payload MUST be empty.

# Open questions
## Merge resources
The request to `/config` resource already includes most fields for `/applied` requests, so it might be possible to merge them in a single resource.

## Error handling
What errors are possible here? Should endpoint care?

## Subscribing to a part of configuration
This is still not addressed in this document.

## Only send configuration difference
Minimizing network traffic by only sending a configuration difference is not addressed.

On the other hand, we're sending JSONs, so there are other more efficient ways to minimize traffic by changing the format (e.g., use BSON, CBOR, MessagePack).
