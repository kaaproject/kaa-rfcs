---
name: Configuration Management Extension
shortname: 5/CMX
status: raw
editor: Dmitry Sergeev <dsergeev@cybervisiontech.com>
contributors: Andrew Kokhanovskyi <akokhanovskyi@cybervisiontech.com>
---

## Introduction

The Configuration Management Extension (CMX) protocol is an endpoint-aware [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

It is intended to manage endpoint configuration distribution.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## Requirements and constraints

- CMX protocol must support client-initiated configuration data pull.
Compliant endpoints should be able to receive the latest configuration by request.
- In case of the configuration pull no receipt confirmation is required from the client.
- CMX protocol must support server-initiated configuration data push.
Compliant endpoints should be able to receive the latest configuration data whenever it changes at the server or at the moment of connecting without explicitly requesting a configuration update.
- In case of the configuration push compliant clients MUST acknowledge the receipt.
- In cases when the configuration data changes more than once while the endpoint is not connected to the server, after reconnecting it SHOULD receive only the latest data.
- CMX protocol implementations MUST support JSON-structured configuration data transfer.
Support for any other formats is OPTIONAL.

## Design

### Configuration pull

#### Configuration pull request

Client MUST send requests with the following extension-specific [resource path](/0001-kaa-protocol/README.md#resource-path-format) part in order to receive the configuration:

  `<endpoint_token>/pull/<message_format>[/<config_format>]`

  where:
  - `endpoint_token` identifies the endpoint;
  - `message_format` specifies the format of the configuration pull messages' payload, e.g. `json`, `protobuf`, etc.;
  - `config_format` specifies the format of the configuration body, e.g. `json`, `avro`, etc.
  This resource path parameter is optional.
  In case it is not specified, the configuration body format is assumed to match the message format.
  It is RECOMMENDED to omit the `<config_format>` resource path suffix in case the desired configuration format matches the message format.

Example configuration pull request extension-specific resource path parts:
  - `<endpoint_token>/pull/json`
  - `<endpoint_token>/pull/json/json`
  - `<endpoint_token>/pull/json/avro`

This document only defines the `json` message and configuration formats.
More formats may be specified in separate RFCs.

When the `<message_format>` is set to `json`, the request payload MUST be a JSON-encoded object with the following [JSON schema](http://json-schema.org/) ([separate file](./config-pull-request.schema.json)):

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "5/CMX configuration pull request schema",

  "type": "object",
  "properties": {
    "id": {
      "type": [ "string", "number" ],
      "description": "ID of the message used to match server response to the request."
    },
    "configId": {
      "type": "string",
      "description": "Identifier of the currently applied configuration. Optional."
    }
  },
  "required": [ "id" ],
  "additionalProperties": false
}

```

Example 1:
```json
{
  "id": 42,
  "configId": "97016dbe8bb4adff8f754ecbf24612f2"
}
```

Example 2:
```json
{
  "id": 42
}
```

#### Configuration pull response

The server MUST respond to the configuration pull request by publishing the response message according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).
The extension-specific resource path part format is:

  `<endpoint_token>/pull/<message_format>[/<config_format>]/status`

  where `message_format` and `config_format` (if present) are copied from the request message.

When the `<message_format>` is set to `json`, the response payload MUST be a JSON-encoded object with the following JSON schema ([separate file](./config-pull-response.schema.json)):

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "5/CMX configuration pull response schema",

  "type": "object",
  "properties": {
    "id": {
      "type": [ "string", "number" ],
      "description": "ID of the message used to match server response to the request."
    },
    "configId": {
      "type": "string",
      "description": "Identifier of the current configuration."
    },
    "statusCode": {
      "type": "number",
      "description": "Status code based on HTTP status codes."
    },
    "reasonPhrase": {
      "type": "string",
      "description": "A human-readable string explaining the cause of an error (if any). In case the request was successful, it is `ok`."
    },
    "config": {
      "description": "Configuration body of an arbitrary type (as set by the request `config_format`)."
    }
  },
  "required": [ "id", "configId", "statusCode", "reasonPhrase" ],
  "additionalProperties": false
}
```

  where
  - `id` MUST match that in the request;
  - `configId` MUST be set to the ID of the most current endpoint configuration known to the server;
  - `config` MUST be present, unless `configId` matches in the request and the response.

Example:
```json
{
  "id": 42,
  "configId": "97016dbe8bb4adff8f754ecbf24612f2",
  "statusCode": 200,
  "reasonPhrase": "ok",
  "config": {
    "key": "value",
    "array" : [
        "value2"
    ]
  }
}
```

Example for the case when there is no new configuration data for the endpoint (`configId` matches the most up to date one):
```json
{
  "id": 42,
  "configId": "97016dbe8bb4adff8f754ecbf24612f2",
  "statusCode": 304,
  "reasonPhrase": "Not changed"
}
``` 

### Configuration push

#### Configuration push request

Server MUST send requests with the following extension-specific resource path part in order to push the configuration to the clients:

  `<endpoint_token>/push/<message_format>[/<config_format>]`

  where:
  - `endpoint_token` identifies the endpoint;
  - `message_format` specifies the format of the push configuration messages' payload, e.g. `json`, `protobuf`, etc.;
  - `config_format` specifies the format of the configuration body, e.g. `json`, `avro`, etc.
  This resource path parameter is optional.
  In case it is not specified, the configuration body format is assumed to match the message format.
  It is RECOMMENDED to omit the `<config_format>` resource path suffix in case the configuration format matches the message format.

Example configuration push request extension-specific resource path parts:
  - `<endpoint_token>/push/json`
  - `<endpoint_token>/push/json/json`
  - `<endpoint_token>/push/json/avro`

This document only defines the `json` message and configuration formats.
More formats may be specified in separate RFCs.

When the `<message_format>` is set to `json`, the request payload MUST be a JSON-encoded object with the following JSON schema ([separate file](./config-push-request.schema.json)):

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "5/CMX configuration push request schema",

  "type": "object",
  "properties": {
    "id": {
      "type": [ "string", "number" ],
      "description": "ID of the message used to match client response to the request."
    },
    "configId": {
      "type": "string",
      "description": "Identifier of the current configuration."
    },
    "config": {
      "description": "Configuration body of an arbitrary type (as defined by the request `config_format`)."
    }
  },
  "required": [ "id", "configId", "config" ],
  "additionalProperties": false
}
```

Example:
```json
{
  "id": 42,
  "configId": "97016dbe8bb4adff8f754ecbf24612f2",
  "config": [
    { "key": "value" },
    15,
    [ "an", "array", 13 ]
  ]
}
```

#### Configuration push response

The client MUST respond to the configuration push request by publishing the response message according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).
The extension-specific resource path part format is:

  `<endpoint_token>/push/<message_format>[/<config_format>]/status`

  where `message_format` and `config_format` (if present) are copied from the request message.

When the `<message_format>` is set to `json`, the response payload MUST be a JSON-encoded object with the following JSON schema ([separate file](./config-push-response.schema.json)):

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "5/CMX configuration push response schema",

  "type": "object",
  "properties": {
    "id": {
      "type": [ "string", "number" ],
      "description": "ID of the message used to match client response to the request."
    },
    "statusCode": {
      "type": "number",
      "description": "Status code based on HTTP status codes."
    },
    "reasonPhrase": {
      "type": "string",
      "description": "A human-readable string explaining the cause of an error (if any). In case the request was successful, it is `ok`."
    }
  },
  "required": [ "id", "statusCode", "reasonPhrase" ],
  "additionalProperties": false
}
```

Example:
```json
{
  "id": 42,
  "statusCode": 200,
  "reasonPhrase": "ok"
}
```
