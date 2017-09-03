---
name: Command Execution Protocol
shortname: 11/CEP
status: raw
editor: Alexey Shmalko <ashmalko@kaaiot.io>
---

<!-- toc -->

# Introduction
Command Execution Protocol is an endpoint-aware 1/KP extension. It is designed to allow command execution on endpoints.

This document describes how commands are delivered to endpoints and how the server is notified of command execution results. This document does not cover who and how initiates command, and how results are interpreted.

# Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

- **Command** is an action requested to be executed on a specific endpoint that may produce a result or fail.
- **Command Invocation** is an act of a command entering the system and being scheduled for execution.
- **Command Execution** is an act of transfering a command to endpoint for actual execution and waiting for execution results.

# Requirements and constraints
- Command execution is asynchronous in nature and may take time. Furthermore, the endpoint may reboot or client may reconnect as a part of the command execution sequence.
- An endpoint may support multiple command types, thus it must tell them apart.
- Command may success or fail. The caller must know outcomes of the command or the reason of a failure.
- Commands are often not idempotent. That is, executing a single command twice may result in a different (possibly undesired) outcome.
- An endpoint may be offline at the time of command invocation. The caller must be informed that the command execution did not succeed.

# Use cases
## UC1: Remote control
An endpoint operates a device. Command Execution Protocol may be used to transfer commands to the endpoint and instruct it to change device behavior.

## UC2: Expose local resources
An endpoint represents a sensor that does not measure the value periodically. Command Execution Protocol may be used to issue a measurement command to the endpoint and transfer measurement results back to the caller.

## UC3: OTA update
> Note: while OTA updates require more support and probably a separate protocol, a basic version can be implemented using command execution.

An endpoint may receive a command to update its firmware. When update is complete, the endpoint sends an update result to the server.

# Design
The Command Execution Protocol extension consists of two resource sets with the corresponding resource paths:
- `/<endpoint_token>/command/<command_type>` - command resource
- `/<endpoint_token>/result/<command_type>` - result resource

## Command resource
Command resource is used to distribute commands to the endpoints.

The command resource is a request/response resource with an observe extension which resides at the following resource path:
```
/<endpoint_token>/command/<command_type>
```
where `<command_type>` is a type of the command the endpoint wants to serve.

For MQTT, observe is achieved by publishing multiple responses to the response topic; for CoAP, it is achieved with the `Observe` option defined in [RFC 7641](https://tools.ietf.org/html/rfc7641).

Command type MUST be a non-empty alpha-numeric string identifying the command type to the endpoint.

### Command request
The endpoint command requests MUST be a UTF-8 encoded JSON object with the JSON Schema defined in [0011-command-request.schema.json](./0011-command-request.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "11/CEP command request schema",

  "type": "object",
  "properties": {
    "observe": {
      "type": "boolean",
      "description": "Whether to send upcoming command for this endpoint."
    }
  },
  "additionalProperties": false
}
```

If `observe` field is present and is `true`, the server MUST send new commands to the endpoint whenever they arrive.

If `observe` field is present and is `false`, the server MUST NOT send new commands to the endpoint whenever they arrive.

The server MAY send commands to the endpoint when no request were done before, or `observe` was not present in a request.

The server MAY support subscribing to all command types by accepting requests at `/<endpoint_token>/command` resource path. In that case, command representation MUST include `type` property that is a string identifying the command type.

### Command response
The server MUST respond with all outstanding commands of the specified command type.

The response MUST be a UTF-8 encoded array with the schema defined in [0011-command-response.schema.json](./0011-command-response.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "11/CEP command response schema",

  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "id": {
        "type": "integer",
        "description": "ID of the command."
      },
      "payload": {
        "description": "The command payload to be interpreted by the endpoint."
      }
    },
    "required": [
      "id"
    ],
    "additionalProperties": false
  }
}
```

Each element of the array represents an outstanding command that hasn't been completed or expired yet.

`id` property is used to match command results to commands and MUST be unique within the command type and an endpoint.

When there is no outstanding commands, the server MUST return an empty array.

Upon reception of a command, the endpoint SHOULD "execute" the command in a way that is command-specific and is out of scope of this document. Upon a command execution success or failure, the endpoint MUST push the command result using the Result resource.

## Result resource
The result resource is used to notify the server of the command execution status (success/failure) and results.

Result resource is a request/response resource with the following resource path:
```
/<endpoint_token>/result/<command_type>
```
where `<command_type>` is a type of the command.

### Result request
A result request is a UTF-8 encoded JSON array with the JSON Schema defined in the [0011-result-request.schema.json](./0011-result-request.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "11/CEP result request schema",

  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "id": {
        "type": "integer",
        "description": "ID of the command."
      },
      "statusCode": {
        "type": "integer",
        "description": "Status code of the command execution. Based on HTTP status codes."
      },
      "reasonPhrase": {
        "type": "string",
        "description": "Intended to give a short textual description of the status code."
      },
      "payload": {
        "description": "A command result payload to be interpreted by the caller."
      }
    },
    "required": [
      "id",
      "statusCode"
    ],
    "additionalProperties": false
  }
}
```

A request represents execution results of one or more commands.

Upon reception of a request, the server MUST remove the corresponding command from the list of outstanding commands for the given endpoint.

### Result response
A result response is an error response as defined per [1/KP](/0001/README.md#error-response-format).
