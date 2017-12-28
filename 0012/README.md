---
name: Command Invocation Protocol
shortname: 12/CIP
status: raw
editor: Andrew Pasika <apasika@kaaiot.io>
---

<!-- toc -->


## Introduction

Command Invocation Protocol (CIP) is designed to communicate endpoint command data between Kaa services.

CIP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Command invocation caller (caller)**: any service that sends endpoint command data to other service(s).
- **Command invocation agent (agent)**: any service that receives endpoint command data from other service(s).


## Design

### Command invocation request

*Command invocation request* is a [targeted message](/0003/README.md#targeted-messaging) published by command invocation caller that wants to invoke command on specific endpoint.

The caller MUST send command invocation request messages using the following NATS subject:
```
kaa.v1.service.{provider-service-instance-name}.cip.command-request
```

The caller MUST include NATS `replyTo` field to handle the [command invocation result response](#command-invocation-result).
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{consumer-service-replica-id}.cip.command-result
```

For more information, see [3/Messaging IPC](/0003/README.md).


*Command invocation request* message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0012-command-invocation-request.avsc](./0012-command-invocation-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.cip.gen.v1",
    "type":"record",
    "name":"CommandInvocationRequest",
    "doc":"Command invocation request message.",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID that is used for message processing tracking across services."
        },
        {
            "name":"timestamp",
            "type":"long",
            "doc":"Message creation UNIX timestamp in milliseconds."
        },
        {
            "name":"timeout",
            "type":"long",
            "default":0,
            "doc":"Amount of milliseconds (since the timestamp) until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Identifier of the endpoint the command is invoked against."
        },
        {
            "name":"commandType",
            "type":"string",
            "doc":"Command type."
        },
        {
            "name":"commandId",
            "type":"int",
            "doc":"Unique command ID that is used for command correlation. (endpointId, commandType, commandId) tuple uniquely identifies a command instance."
        },
        {
            "name":"payload",
            "type":[
              "null",
              "bytes"
            ],
            "default":null,
            "doc":"The command payload to be interpreted by endpoint."
        }
    ]
}
```

Example:

```json
{
   "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
   "timestamp":1514372799674,
   "timeout":0,
   "endpointId":"b197e391-1d13-403b-83f5-87bdd44888cf",
   "commandType":"measurement",
   "commandId":284,
   "payload":{
      "bytes":"{\"temperature\":true,\"humidity\":false}"
   }
}
```

### Command invocation result

*Command invocation result* message MUST be sent by invocation agent in response to a [Command invocation request](#command-invocation-request).
Agent MUST publish command invocation result message to the subject provided in the NATS `replyTo` field of the request.

*Command invocation result* message payload MUST be an Avro-encoded object with the following schema ([0012-command-invocation-result.avsc](./0012-command-invocation-result.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.cip.gen.v1",
    "type":"record",
    "name":"CommandInvocationResult",
    "doc":"Command invocation result message.",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID that is used for message processing tracking across services."
        },
        {
            "name":"timestamp",
            "type":"long",
            "doc":"Message creation UNIX timestamp in milliseconds."
        },
        {
            "name":"timeout",
            "type":"long",
            "default":0,
            "doc":"Amount of milliseconds (since the timestamp) until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Application version of the endpoint that responded."
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Identified of the endpoint the command was invoked against."
        },
        {
            "name":"commandType",
            "type":"string",
            "doc":"Command type."
        },
        {
            "name":"commandId",
            "type":"int",
            "doc":"Unique command ID that is used for command correlation."
        },
        {
            "name":"statusCode",
            "type":"int",
            "doc":"Command execution status code. Based on HTTP status codes."
        },
        {
            "name":"reasonPhrase",
            "type":[
              "null",
              "string"
            ],
            "default":null,
            "doc":"Human-readable reason phrase."
        },
        {
            "name":"payload",
            "type":[
              "null",
              "bytes"
            ],
            "default":null,
            "doc":"A command result payload to be interpreted by the command caller."
        }
    ]
}
```

Example:

```json
{
   "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
   "timestamp":1514381235893,
   "timeout":0,
   "appVersionName":"smartSensorV1",
   "endpointId":"b197e391-1d13-403b-83f5-87bdd44888cf",
   "commandType":"measurement",
   "commandId":284,
   "statusCode":200,
   "reasonPhrase":{
      "string":"OK"
   },
   "payload":{
      "bytes":"{\"temperature\":24}"
   }
}
```
