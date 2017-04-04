---
name: Configuration Management Extension
shortname: 7/CMX
status: raw
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
---

## Introduction

The Configuration Management Extension is a [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

It is intended to manage endpoint configuration distribution.

## Requirements and constraints
### Problems and possible solutions

1. _The server should know if a configuration has been applied by the endpoint._

   Possible solutions:
   - The client sends a publish message in a response to the new configuration provided by the server.

     This means the server is initiating a request. That does not work well for HTTP, CoAP, and other protocols that only support client-initiated requests.

   - Introduce `/applied` resource, so the client sends an additional request once configuration is applied.

2. _An endpoint might be interested in a part of the general configuration only._

    Possible solutions:
    - Allow subscribing to a part of configuration. (e.g., using [JSONPath](http://goessner.net/articles/JsonPath/), or JSON Pointer defined in [RFC 6901](https://tools.ietf.org/html/rfc6901).)

3. _Difference between subsequent configurations might be small, so sending the whole configuration is inefficient._

    Possible solutions:
    - Send only updates. (e.g., using JSON Patch format, defined in [RFC 6902](https://tools.ietf.org/html/rfc6902).)

## Use cases

### UC1: Configuration request
Configuration delivery by request. The endpoint should be able to receive latest configuration from CMX by request.

### UC2: Configuration push
Configuration delivery is initiated by the server. The endpoint should receive latest configuration when it connects to the server if this configuration wasn't applied yet.

## Design
The extension follows client-initiated request/response pattern described in 1/KP, which nicely maps to both MQTT and CoAP.

There are two resources provided by the extension.

- `/<endpoint_token>/config/json` is used for subscribing and receiving a config.
- `/<endpoint_token>/applied/json` is used for notifiying server of the currently active configuration.

### Configuration identifier
Configuration identifier is an opaque string, which uniquely identifies the configuration. The client SHOULD NOT assume the nature of identifiers.

### Absent configuration
If an endpoint does not have a configuration assigned, its configuration is `null`, and the corresponding configuration identifier is an empty string (`""`).

### Request/response
The extension uses request/response pattern described in [1/KP](/0001-kaa-protocol/README.md) with observe extension. For MQTT, it is achieved by publishing multiple responses to the response path; for CoAP, it is achieved with the `Observe` option defined in [RFC 7641](https://tools.ietf.org/html/rfc7641).

### Config resource
`/<endpoint_token>/config/json` resource is used for distributing the latest endpoint configuration.

A request is a JSON object with the following fields:
- `id` (optional) - an opaque string used to match responses with requests. If absent, the client is not interested in matching responses with the requests.
- `configId` (optional) - the identifier of the currently applied configuration. If not present, the absent configuration identifier (`""`) is assumed.
- `observe` (optional) - identifies if the client is interested in observing the configuration.

A response is a new configuration along with needed meta information. It is a JSON object with the following fields:
- `id` (optional) - identifier of the corresponding request. MUST be the same as in the corresponding request.
- `configId` (optional) - identifier of the `config`.
- `config` (optional) - an arbitrary JSON type, representing the latest configuration available for the endpoint.

`configId` and `config` MUST come in a pair. If `configId` and `config` are absent in the response, that means configuration hasn't changed.

#### Semantics

If `configId` field is missing, the server MUST respond with the latest configuration for the given endpoint, or no configuration if none was assigned.

If `configId` field is present, the server MUST send new configuration only if `configId` differs from the currently assigned configuration identifier for the endpoint.

If `observe` field is present and is `true`, the server MUST send new configuration to the endpoint whenever configuration changes.

If `observe` field is present and is `false`, the server MUST NOT send new configurations to the endpoint without a further explicit request.

The server MAY send configurations to the endpoint when no request were done before, or `observe` was not present in a request.

If the current config matches one specified by `configId` in the request, the server MUST return response with both `configId` and `config` absent.

If an endpoint successfully applies provided configuration, it SHOULD notify the server via `/applied` resource.

### Applied resource
`/<endpoint_token>/applied/json` resource is used by the endpoint to notify the server of successfully applied configuration.

The request is a JSON object with the following fields:
- `configId` (required) - id of applied configuration.

The response is empty.

## Open questions
### Merge resources
The request to `/config` resource already includes all fields for `/applied` requests, so it might be possible to merge them in a single resource.

### Error handling
Whaw errors are possible here? Should endpoint care?

### Configuration rejection
Should it be possible for the endpoint to actively reject a configuration?

Examples are:
- the endpoint is unable to process the configuration.
- the configuration is ill-formated, or pre-conditions are not met.

### Subscribing to a part of configuration
This is still not addressed in this document.

### Only send configuration difference
Minimizing network traffic by only sending a configuration difference is not addressed.

On the other hand, we're sending JSONs, so there are other more efficient ways to minimize traffic by changing the format (e.g., use BSON, CBOR, MessagePack).
