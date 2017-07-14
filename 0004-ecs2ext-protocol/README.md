---
name: ECS to extensions protocol
shortname: 4/ECS2EXT
status: draft
editor: Dmitry Sergeev <dsergeev@cybervisiontech.com>
contributors: Andrew Kokhanovskyi <akokhanovskyi@cybervisiontech.com>
---

## Introduction
ECS2EXT is the messaging protocol for communicating between Endpoint Communication Service implementations and the various extension services.
It is based on the [targeted messaging IPC design](/0003-messaging-ipc/README.md#targeted-messaging).

## Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## Requirements and constraints
1. The protocol must support transporting client-originated data to extensions and vice versa.
2. Some extensions may provide multiple message handling functions and payload formats.
The protocol must support specification of the handling function and payload format to use for processing transported data.
3. Some extensions require reporting processing status back to clients with or without any payload.
4. Some extensions require clients to report processing status with or without any payload.

## Design

### Client data transfer to extensions

ClientData messages are used by ECS implementations for transferring endpoint-originated data to extension services.
Non-affinity session messages MUST be published to a [service instance-wide subject](/0003-messaging-ipc/README.md#service-instance-wide-subjects):

  `kaa.v1.service.<service-instance-name>.ecs2ext.ClientData`

  where `service-instance-name` is the target extension service instance name.

In case of affinity sessions ClientData message MUST be published to the [service replica-specific subject](/0003-messaging-ipc/README.md#service-replica-specific-subjects) defined by the `replyTo` subject in the previously received message that set up the session affinity.

ECS implementations SHOULD also set `replyTo` subject when sending ClientData messages according to the [session affinity](/0003-messaging-ipc/README.md#session-affinity) rules.

The NATS message payload is an Avro object with the following schema ([file](./ecs2ext-client-data.avsc)):

```json
{
  "type": "record",
  "name": "ClientData",
  "namespace": "org.kaaproject.ipc.ecs2ext.gen.v1",
  "doc": "ClientData message carries client-originated messages from ECS to extensions",
  "fields": [
    {
      "name": "requestId",
      "type": "int",
      "doc": "Request ID primarily used for tracking the message from client side."
    },
    {
      "name": "correlationId",
      "type": "string",
      "doc": "Message tracing ID primarily used for tracking the message processing across services."
    },
    {
      "name": "timestamp",
      "type": "long",
      "doc": "Unix time in milliseconds when the message was created."
    },
    {
      "name": "timeout",
      "type": "long",
      "default": -1,
      "doc": "The amount of milliseconds since timestamp when the message expires at the originating entity."
    },
    {
      "name": "appVersionName",
      "type": [
        "string",
        "null"
      ],
      "default": "",
      "doc": "Unique name of the application version that the endpoint identified by the endpointID belongs to. Null for endpoint-unaware extensions."
    },
    {
      "name": "endpointId",
      "type": [
        "string",
        "null"
      ],
      "default": "",
      "doc": "Unique endpoint ID. Null for endpoint-unaware extensions."
    },
    {
      "name": "path",
      "type": "string",
      "doc": "Action path field that is used to determine the message handling function and the payload format."
    },
    {
      "name": "payload",
      "type": [
        "bytes",
        "null"
      ],
      "doc": "Serialized message content. Can be null in status-only messages."
    }
  ]
}
```
### Extension data transfer to clients

ExtensionData messages are used by extension services for transferring data to ECS services.
Non-affinity session messages MUST be published to a [service instance-wide subject](/0003-messaging-ipc/README.md#service-instance-wide-subjects):

  `kaa.v1.service.<service-instance-name>.ecs2ext.ExtensionData`

  where `service-instance-name` is the target ECS service instance name.

In case of affinity sessions ExtensionData message MUST be published to the [service replica-specific subject](/0003-messaging-ipc/README.md#service-replica-specific-subjects) defined by the `replyTo` subject in the previously received message that set up the session affinity.

Extensions MAY also set `replyTo` subject when sending ExtensionData messages according to the [session affinity](/0003-messaging-ipc/README.md#session-affinity) rules.

The NATS message payload is an Avro object with the following schema ([file](./ecs2ext-extension-data.avsc)):

```json
{
  "type": "record",
  "name": "ExtensionData",
  "namespace": "org.kaaproject.ipc.ecs2ext.gen.v1",
  "doc": "ExtensionData message carries extension-originated messages destined for clients to ECS",
  "fields": [
    {
      "name": "requestId",
      "type": "int",
      "doc": "Request ID primarily used for tracking the message from client side."
    },
    {
      "name": "correlationId",
      "type": "string",
      "doc": "Message tracing ID primarily used for tracking the message processing across services."
    },
    {
      "name": "timestamp",
      "type": "long",
      "doc": "Unix time in milliseconds when the message was created."
    },
    {
      "name": "timeout",
      "type": "long",
      "default": -1,
      "doc": "The amount of milliseconds since timestamp when the message expires at the originating entity."
    },
    {
      "name": "appVersionName",
      "type": [
        "string",
        "null"
      ],
      "default": "",
      "doc": "Unique name of the application version that the endpoint identified by the endpointID belongs to. Null for endpoint-unaware extensions."
    },
    {
      "name": "extensionInstanceName",
      "type": [
        "string",
        "null"
      ],
      "default": "",
      "doc": "Unique name of the extension instance that originated the message."
    },
    {
      "name": "endpointId",
      "type": [
        "string",
        "null"
      ],
      "default": "",
      "doc": "Unique endpoint ID. Null for endpoint-unaware extensions."
    },
    {
      "name": "path",
      "type": [
        "string",
        "null"
      ],
      "doc": "Action path field that is used to determine the message handling function and the payload format. Null if extension responding with error status code with no need to publish message to client"
    },
    {
      "name": "payload",
      "type": [
        "bytes",
        "null"
      ],
      "doc": "Serialized message content. Can be null in status-only messages."
    },
    {
      "name": "statusCode",
      "type": [
        "int",
        "null"
      ],
      "default": 200,
      "doc": "Message processing status code. Can be null in requests."
    },
    {
      "name": "reasonPhrase",
      "type": [
        "null",
        "string"
      ],
      "default": "OK",
      "doc": "Is intended to give a short textual description of the Status-Code."
    }
  ]
}
```

Example of an ExtensionData message:
```json
{
  "requestId": 42,
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "extensionInstanceName" : "humidity-sensor-cmx-1",
  "appVersionName" : "humidity-sensor-v3",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "path" : "/push/json",
  "payload" : {
    "bytes" : "ewogICJzYW1wbGluZyIgOiAyMDAKfQ=="
  },
  "status" : 200,
  "reasonePhrase": "OK"
}
```
## Glossary

- _ECS2EXT_ -- name of the protocol used in communication between Endpoint Communication Service and extension service.

- _ECS_ -- short name for Endpoint Communication Service
