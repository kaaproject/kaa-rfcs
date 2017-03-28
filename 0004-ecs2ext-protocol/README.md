---
name: Communication protocol between Endpoint Communication Service and extension services
shortname: 4/ECS2EXT
status: raw
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
ECS implementations MUST send message using NATS to the next subject: `kaa.v1.service.{service-instance-name}.ecs2ext.ClientData`.
Also, ECS SHOULD include NATS `replyTo` field pointing to the ECS replica that will handle the response: `kaa.v1.service.{service-instance-name}.{service-instance-replica-id}`.

The NATS message is a JSON-encoded Avro object.
The Avro schema can be found [here](./ClientData.avsc).

Example of a ClientData message with payload:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "89556d5962",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "payload" : {
    "bytes" : "skodg'j394gj3q9i0jg03[09j0[3q[jj39g4"
  },
  "status" : null,
  "path" : "/push/json"
}
```

Example of a ClientData message with payload and status:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "89556d5962",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "payload" : {
    "bytes" : "skodg'j394gj3q9i0jg03[09j0[3q[jj39g4"
  },
  "status" : {
    "string" : "ok"
  },
  "path" : "/push/json"
}
```

Example of a ClientData status message:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "89556d5962",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "payload" : null,
  "status" : {
    "string" : "ok"
  },
  "path" : "/push/json"
}
```

### Extension data transfer to clients
Extensions MUST send message to ECS using `reply to` NATS field value or to the next subject: `kaa.v1.service.{service-instance-name}.ecs2ext.ExtensionData`.

The NATS message is a JSON-encoded Avro object.
The Avro schema can be found [here](./ExtensionData.avsc).

Example of an ExtensionData message with payload:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "89556d5962",
  "extensionInstanceName" : "9070ad58-e5e6-482d-bd88-bdf79db23b63",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "payload" : {
    "bytes" : "skodg'j394gj3q9i0jg03[09j0[3q[jj39g4"
  },
  "status" : null,
  "path" : "/push/json"
}
```

Example of an ExtensionData message:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "89556d5962",
  "extensionInstanceName" : "9070ad58-e5e6-482d-bd88-bdf79db23b63",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "payload" : {
    "bytes" : "skodg'j394gj3q9i0jg03[09j0[3q[jj39g4"
  },
  "status" : {
    "string" : "ok"
  },
  "path" : "/push/json"
}
```

Example of a status ExtensionData message:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionName" : "89556d5962",
  "extensionInstanceName" : "9070ad58-e5e6-482d-bd88-bdf79db23b63",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "payload" : null,
  "status" : {
    "string" : "ok"
  },
  "path" : "/push/json"
}
```

## Glossary

- ECS2EXT -- name of the protocol used in communication between Endpoint Communication Service and extension service
- ECS -- short name for Endpoint Communication Service
- payload-only message -- message that contains `payload` field value, but `status` value is missing.
- status-only message -- message that contains `status` field value, but `payload` value is missing. Used in delivery confirmation process.
