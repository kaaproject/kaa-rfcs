---
name: Configuration Management Protocol
shortname: 7/CMP
status: draft
editor: Alexey Shmalko <ashmalko@kaaiot.io>
contributors: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


# Introduction

The Configuration Management Protocol (CMP) is a [Kaa Protocol](/0001/README.md) extension.

CMP is intended to manage endpoint configuration distribution.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).


# Requirements and constraints

- Server should know if configuration was applied by endpoint. <!-- TODO: why? add more reasoning -->

  Possible solutions:
  - Client sends a reply upon receiving new configuration from the server.
    This means the server initiates a request.
    That does not work well for HTTP, CoAP, and other protocols that only support client-initiated requests, and is not recommended by 1/KP.
  - Introduce `/applied` resource, so client sends an additional request once configuration is applied.

- Endpoint might only require a specific part of the configuration, not all of it.

  Solutions:
  - Allow subscribing to a part of configuration, e.g., using [JSONPath](http://goessner.net/articles/JsonPath/) or JSON Pointer defined in [RFC 6901](https://tools.ietf.org/html/rfc6901).

- Difference between subsequent configurations might be small, so sending the whole configuration is inefficient.

  Solutions:
  - Send only updates, e.g., using JSON Patch format defined in [RFC 6902](https://tools.ietf.org/html/rfc6902).

- Endpoint can actively reject configuration.

  Examples are:
  - Endpoint is unable to process configuration.
  - Configuration is ill-formatted, or pre-conditions are not met.

  Solutions:
  - Send back to server a status result of the attempt to apply configuration (success or failure).


# Use cases

## UC1: Configuration request

Configuration delivery by request.
Endpoint should be able to receive its latest configuration from server by request.


## UC2: Configuration push

Configuration delivery is initiated by server.
Endpoint should receive latest configuration when it connects to server, if this configuration is not applied yet.


## UC3: Named configurations

Endpoint requires multiple independent configurations for different subsystems.
For example, an IoT device may have separate configurations for:
- Network settings (WiFi credentials, proxy settings)
- Display settings (brightness, themes)
- Sensor calibration parameters
- Business logic rules

Each named configuration can be managed, updated, and applied independently without affecting other configurations.


# Design

## Configuration identifier

Configuration identifier is an opaque string that uniquely identifies the configuration.
Client SHOULD NOT assume the nature of identifiers.


## Absent configuration

If an endpoint does not have a configuration assigned, its configuration is `null`, and the corresponding configuration identifier is an empty string (`""`).


## Request/response

<!-- TODO: move observe extension to 1/KP -->
7/CMP follows client-initiated request/response pattern defined in [1/KP](/0001/README.md#requestresponse-pattern) with observe extension.
For MQTT, this is achieved by publishing multiple responses to the response topic; for CoAP, it is achieved with the `Observe` option defined in [RFC 7641](https://tools.ietf.org/html/rfc7641).


## Configuration resource

Configuration resource is used by endpoints to subscribe to and receive the latest configuration from server.

The configuration resource is a request/response resource with an observe extension which resides at the following extension-specific resource path:
```
/<endpoint_token>/config/json[/<config_name>]
```

where `<config_name>` is an optional configuration name.

When `<config_name>` is omitted, the endpoint accesses the **default configuration**.
This ensures full backward compatibility with existing endpoints that use the path `/<endpoint_token>/config/json`.

When `<config_name>` is provided, the endpoint accesses a **named configuration**.
Named configurations allow endpoints to manage multiple independent configurations simultaneously.

Configuration name MUST be a non-empty alphanumeric string (matching `^[a-zA-Z0-9_-]+$`) that uniquely identifies the configuration within the endpoint.

Examples of valid resource paths:
- `/<endpoint_token>/config/json` — default configuration (backward compatible)
- `/<endpoint_token>/config/json/network` — named configuration "network"
- `/<endpoint_token>/config/json/display_settings` — named configuration "display_settings"


### Configuration resource request

The client SHOULD send configuration resource requests to the configuration resource path.

The request payload MUST be a UTF-8 encoded JSON object with the following [JSON schema](http://json-schema.org/) ([0007-config-request.schema.json](./0007-config-request.schema.json)):

```json
{
    "$schema":"http://json-schema.org/schema#",
    "title":"7/CMP configuration request schema",

    "type":"object",
    "properties":{
        "configId":{
            "type":"string",
            "description":"Identifier of the currently applied configuration"
        },
        "observe":{
            "type":"boolean",
            "description":"Whether the endpoint is interested in observing its configuration"
        }
    },
    "additionalProperties":false
}
```

If `configId` field is missing, server MUST respond with the current configuration for the given endpoint.

If `configId` field is present, server MUST send new configuration only if it differs from the identifier of the configuration currently assigned to that endpoint.

If `observe` field is present and is `true`, server MUST send new configuration to the endpoint whenever configuration changes.

If `observe` field is present and is `false`, server MUST NOT send new configurations to the endpoint without a further explicit request.

Server MAY send configurations to the endpoint if no request has yet been made or if `observe` was not present in a request.

If the current configuration matches one specified by `configId` in the request, the server MUST return response with both `configId` and `config` absent.

Below are request payload examples.
- Get current configuration only:
  ```json
  {
    "observe":false
  }
  ```
- Endpoint's current configuration ID is `97016dbe8bb4adff8f754ecbf24612f2`:
  ```json
  {
    "configId": "97016dbe8bb4adff8f754ecbf24612f2"
  }
  ```
- Subscribe to all future configuration updates:
  ```json
  {
    "observe": true
  }
  ```


### Configuration resource response

The server response payload MUST be a UTF-8 encoded object with the following JSON Schema ([0007-config-response.schema.json](./0007-config-response.schema.json)):

```json
{
    "$schema":"http://json-schema.org/schema#",
    "title":"7/CMP config response schema",

    "type":"object",
    "properties":{
        "configId":{
            "type":"string",
            "description":"Identifier of the current configuration"
        },
        "config":{
            "description":"Configuration body of an arbitrary type"
        }
    },
    "additionalProperties":false
}
```

`configId` and `config` MUST come in a pair.
If `configId` and `config` are absent in the response, that means configuration has not changed.

If endpoint successfully applies provided configuration, it SHOULD notify the server through the `/applied` resource.

Below are response payload examples.
- Configuration hasn't changed:
  ```json
  {
  }
  ```

- New configuration:
  ```json
  {
      "configId":"97016dbe8bb4adff8f754ecbf24612f2",
      "config":{
          "mode":"AP",
          "ssid":"Smart Teapot",
          "password":"acupofteaplease",
          "security":"WPA2_PSK"
      }
  }
  ```


## Applied configuration resource

Applied resource is used by endpoints to notify server of successfully applied configuration.

The applied resource is a request/response resource with the following resource path:
```
/<endpoint_token>/applied/json[/<config_name>]
```

where `<config_name>` is an optional configuration name that MUST match the configuration name used in the corresponding [configuration resource](#configuration-resource) request.

When `<config_name>` is omitted, the endpoint reports the applied status for the **default configuration**.
This ensures full backward compatibility with existing endpoints that use the path `/<endpoint_token>/applied/json`.

When `<config_name>` is provided, the endpoint reports the applied status for the specified **named configuration**.


### Applied configuration request

The request payload MUST be a UTF-8 encoded JSON object with the following JSON Schema ([0007-applied-request.schema.json](./0007-applied-request.schema.json)):

```json
{
    "$schema":"http://json-schema.org/schema#",
    "title":"7/CMX applied configuration request schema",

    "type":"object",
    "properties":{
        "configId":{
            "type":"string",
            "description":"Identifier of the applied configuration"
        },
        "statusCode":{
            "type":"number",
            "description":"Status code based on HTTP status codes",
            "default":200
        },
        "reasonPhrase":{
            "type":"string",
            "description":"Human-readable string explaining the cause of an error (if any)"
        }
    },
    "required":[
        "configId"
    ],
    "additionalProperties":false
}
```

A request means the endpoint has received the provided configuration and tried to apply it.

`statusCode` represents a result of applying the configuration (success or failure).
By convention, HTTP status codes SHOULD be used.
To indicate a successfully applied configuration, endpoints MUST use 2xx status codes.
Any other status codes MUST be treated by server as a failure.
If the result is a failure, `reasonPhrase` SHOULD include a reason for the failure.

Examples below.
- Configuration successfully applied:
  ```json
  {
      "configId":"97016dbe8bb4adff8f754ecbf24612f2"
  }
  ```

- Configuration rejected:
  ```json
  {
      "configId": "97016dbe8bb4adff8f754ecbf24612f2",
      "statusCode": 400,
      "reasonPhrase": "WPA2 is not supported"
  }
  ```


### Response

The response payload MUST be empty.


### Implicit configuration application

Server MAY mark configuration as applied if its identifier is specified in `configId` in [configuration resource request](#configuration-resource-request).


## Named configurations examples

This section provides complete examples of using named configurations.


### Example: Subscribing to multiple named configurations

An endpoint subscribes to both "network" and "display" configurations:

**Request to `/<endpoint_token>/config/json/network`:**
```json
{
    "observe": true
}
```

**Response:**
```json
{
    "configId": "net-a1b2c3d4",
    "config": {
        "wifi": {
            "ssid": "OfficeNetwork",
            "security": "WPA2"
        },
        "proxy": {
            "enabled": false
        }
    }
}
```

**Request to `/<endpoint_token>/config/json/display`:**
```json
{
    "observe": true
}
```

**Response:**
```json
{
    "configId": "disp-e5f6g7h8",
    "config": {
        "brightness": 80,
        "theme": "dark",
        "timeout": 300
    }
}
```


### Example: Reporting applied status for named configuration

After successfully applying the "network" configuration:

**Request to `/<endpoint_token>/applied/json/network`:**
```json
{
    "configId": "net-a1b2c3d4"
}
```


### Example: Backward compatibility

Existing endpoints continue to work without changes by using the default configuration (no `<config_name>` in the path):

**Request to `/<endpoint_token>/config/json`:**
```json
{
    "observe": true
}
```

**Response:**
```json
{
    "configId": "default-x9y8z7",
    "config": {
        "mode": "AP",
        "ssid": "Smart Teapot",
        "password": "acupofteaplease"
    }
}
```

**Applied notification to `/<endpoint_token>/applied/json`:**
```json
{
    "configId": "default-x9y8z7"
}
```


# Open questions

## Merge resources

The request to `/config` resource already includes most fields for `/applied` requests, so it might be possible to merge them in a single resource.


## Error handling

What errors are possible here? Should endpoint care?


## Subscribing to a part of configuration

This is still not addressed in this document.


## Only send configuration difference

Minimizing network traffic by only sending a configuration difference is not addressed.

On the other hand, we're currently sending JSONs, so there are other more efficient ways to minimize traffic by changing the format (e.g., use BSON, CBOR, MessagePack).
