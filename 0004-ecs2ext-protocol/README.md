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

  `kaa.v1.service.{service-instance-name}.ecs2ext.ClientData`

  where `{service-instance-name}` is the target extension service instance name.

In case of affinity sessions ClientData message MUST be published to the [service replica-specific subject](/0003-messaging-ipc/README.md#service-replica-specific-subjects) defined by the `replyTo` subject in the previously received message that set up the session affinity.

ECS implementations SHOULD also set `replyTo` subject when sending ClientData messages according to the [session affinity](/0003-messaging-ipc/README.md#session-affinity) rules.

The NATS message payload is an Avro object.
The Avro schema can be found [here](./ClientData.avsc).

Example of a ClientData message with payload:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "humidity-sensor-v3",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "path" : "/push/json",
  "payload" : {
    "bytes" : "ewogICJpZCI6IDQyLAogICJlbnRyaWVzIjogWwogICAgeyAiaHVtaWRpdHkiOiA4OCB9CiAgXQp9"
  },
  "status" : null
}
```

Example of a ClientData message with payload and status:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "humidity-sensor-v3",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "path" : "/push/json",
  "payload" : {
    "bytes" : "ewogICJpZCI6IDQyLAogICJlbnRyaWVzIjogWwogICAgeyAiaHVtaWRpdHkiOiA4OCB9CiAgXQp9"
  },
  "status" : 200
}
```

Example of a ClientData status message:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "humidity-sensor-v3",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "path" : "/push/json",
  "payload" : null,
  "status" : 200
}
```

### Extension data transfer to clients

ExtensionData messages are used by extension services for transferring data to ECS services.
Non-affinity session messages MUST be published to a [service instance-wide subject](/0003-messaging-ipc/README.md#service-instance-wide-subjects):

  `kaa.v1.service.{service-instance-name}.ecs2ext.ExtensionData`

  where `{service-instance-name}` is the target ECS service instance name.

In case of affinity sessions ExtensionData message MUST be published to the [service replica-specific subject](/0003-messaging-ipc/README.md#service-replica-specific-subjects) defined by the `replyTo` subject in the previously received message that set up the session affinity.

Extensions MAY also set `replyTo` subject when sending ExtensionData messages according to the [session affinity](/0003-messaging-ipc/README.md#session-affinity) rules.

The NATS message payload is an Avro object.
The Avro schema can be found [here](./ExtensionData.avsc).

Example of an ExtensionData message with payload:
```
{
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
  "status" : null
}
```

Example of an ExtensionData message with payload and status:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "extensionInstanceName" : "humidity-sensor-dcx-1",
  "appVersionName" : "humidity-sensor-v3",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "path" : "/push/json/status",
  "payload" : {
    "bytes" : "ewogICJpZCI6IDQyLAogICJzdGF0dXMiOiAib2siCn0="
  },
  "status" : 200
}
```


## Glossary

- ECS2EXT -- name of the protocol used in communication between Endpoint Communication Service and extension service
- ECS -- short name for Endpoint Communication Service
