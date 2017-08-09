---
name: ECS to extensions protocol
shortname: 4/ECS2EXT
status: draft
editor: Dmitry Sergeev <dsergeev@kaaiot.io>
contributors: Andrew Kokhanovskyi <akokhanovskyi@kaaiot.io>, Alexey Shmalko <ashmalko@kaaiot.io>
---

<!-- toc -->

# Introduction
ECS2EXT protocol is designed to allow generic implementations of client communication services which are unaware of specific 1/KP extensions.
The protocol is used for communication between such generic endpoint communication service and various extensions services over NATS.

It is based on the [targeted messaging IPC design](/0003-messaging-ipc/README.md#targeted-messaging).

# Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **ECS**: short name for Endpoint Communication Service.

# Requirements and constraints
1. The protocol must support encapsulating 1/KP in a way so that client communication service doesn't have to know of all protocol extensions.

# Design

## Client data transfer to extensions

ClientData messages are used by ECS implementations for transferring endpoint-originated data to extension services.
Non-affinity session messages MUST be published to a [service instance-wide subject](/0003-messaging-ipc/README.md#service-instance-wide-subjects):

  `kaa.v1.service.<service-instance-name>.ecs2ext.ClientData`

  where `<service-instance-name>` is the target extension service instance name.

In case of affinity sessions, ClientData message MUST be published to the [service replica-specific subject](/0003-messaging-ipc/README.md#service-replica-specific-subjects) defined by the `replyTo` subject in the previously received message that set up the session affinity.

ECS implementations SHOULD set `replyTo` subject when sending ClientData messages according to the [session affinity](/0003-messaging-ipc/README.md#session-affinity) rules.

The NATS message payload is an Avro object with the following schema ([file](./ecs2ext-client-data.avsc)):

```json
{
  "type": "record",
  "name": "ClientData",
  "namespace": "org.kaaproject.ipc.ecs2ext.gen.v1",
  "doc": "Client-originated messages from ECS to extensions",
  "fields": [
    {
      "name": "correlationId",
      "type": "string",
      "doc": "Message tracing ID used for tracking the message processing across services."
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
      "type": "string",
      "doc": "Application version name the request is made to."
    },
    {
      "name": "endpointId",
      "type": [
        "string",
        "null"
      ],
      "doc": "Endpoint ID. Null for endpoint-unaware extensions."
    },
    {
      "name": "resourcePath",
      "type": "string",
      "doc": "Resource path that is used to determine the message handling function and the payload format."
    },
    {
      "name": "requestId",
      "type": [
        "int",
        "null"
      ],
      "doc": "Request ID used by endpoint to match responses."
    },
    {
      "name": "payload",
      "type": "bytes",
      "doc": "Serialized message content."
    }
  ]
}
```

Example of a ClientData message:
```json
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "humidity-sensor-v3",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "resourcePath" : "/push/json",
  "requestId" : {
    "int" : 42
  },
  "payload" : "ewogICJpZCI6IDQyLAogICJlbnRyaWVzIjogWwogICAgeyAiaHVtaWRpdHkiOiA4OCB9CiAgXQp9"
}
 ``` 
  
## Extension data transfer to clients

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
  "doc": "Extension-originated messages intended for clients of ECS",
  "fields": [
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
      "type": "string",
      "doc": "Application version name the request is made to."
    },
    {
      "name": "extensionInstanceName",
      "type": [
        "string",
        "null"
      ],
      "doc": "Unique name of the extension instance that originated the message."
    },
    {
      "name": "endpointId",
      "type": [
        "string",
        "null"
      ],
      "doc": "Unique endpoint ID. Null for endpoint-unaware extensions."
    },
    {
      "name": "resourcePath",
      "type": "string",
      "doc": "Action path field that is used to determine the message handling function and the payload format."
    },
    {
      "name": "requestId",
      "type": [
        "int",
        "null"
      ],
      "doc": "Request ID used by client to match responses."
    },
    {
      "name": "payload",
      "type": [
        "bytes",
        "null"
      ],
      "doc": "Serialized message content."
    },
    {
      "name": "statusCode",
      "type": "int",
      "default": 200,
      "doc": "Message processing status code."
    },
    {
      "name": "reasonPhrase",
      "type": [
        "null",
        "string"
      ],
      "default": null,
      "doc": "Is intended to give a short textual description of the statusCode."
    }
  ]
}
```

Example of an ExtensionData message:
```json
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "extensionInstanceName" : "humidity-sensor-cmx-1",
  "appVersionName" : "humidity-sensor-v3",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "resourcePath" : "/push/json",
  "requestId": {
    "int" : 42
  },
  "payload" : {
    "bytes" : "ewogICJzYW1wbGluZyIgOiAyMDAKfQ=="
  },
  "status" : 200,
  "reasonPhrase": "OK"
}
```
