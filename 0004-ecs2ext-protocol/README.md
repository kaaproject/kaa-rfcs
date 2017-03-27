---
name: Communication protocol between Endpoint Communication Service and extension services
shortname: 4/ECS2EXT
status: raw
editor: Dmitry Sergeev <dsergeev@cybervisiontech.com>
---

## Introduction
ECS2EXT is the messaging protocol for communicating between Endpoint Communication Service implementations and the various extension services.
It is based on the [targeted messaging IPC design](/0003-messaging-ipc/README.md#targeted-messaging).

## Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## Requirements and constraints
### Problems and possible solutions
1. _Message delivery confirmation._

   Solutions:
   - Protocol contains field named `status` that is used to send error message or confirm successful processing.

2. _Ability to determine which Avro schema should be used on extension to de-serialize the `payload` field._

   Solutions:
   - Part of the `path` protocol message field may specify the payload content-encoding (e.g. `json`, `pull/json`, etc.).

3. _Ability to send payload in both directions._

   Solutions:
   - Protocol message contains `payload` field for both directions: from ECS to extension and vice versa.

4. _Ability to send acknowledgement (status) message without a payload._

   Solutions:
   - `payload` field is optional.

## Use cases

### UC1
Endpoint can send some data to extension that should not respond with any data except of a status message for delivery confirmation.
In that case `payload` fields will be filled only in messages that goes from ECS to extension.
Message from extension to ECS should contain `status` field filled; `payload` can be dropped.
Example of extension with such communication: Data Collection Extension.

### UC2
Endpoint can receive messages from extension with `payload` but respond with `status` message only.
Example of such case is push notifications from server to client.

### UC3
Endpoint and extension can send and receive messages with `payload`.


## Message exchange
![](ecs2ext-ipc.png?raw=true)

#### Message from ECS to extension
ECS should send message using NATS to the next subject: `kaa.v1.service.{service-instance-name}`.
Also, ECS should include NATS `replyTo` field pointing to the ECS replica that will handle the response: `kaa.v1.service.{service-instance-name}.{service-instance-replica-id}`.

In addition to `correlationId`, `timestamp`, and `timeout` fields that are defined in the [targeted messaging IPC design](/0003-messaging-ipc/README.md), ECS-to-extension message contains the following fields:
- `appVersionName` (string, required) - refer to [??](/) for description.
- `endpointId` (string, optional) - refer to [??](/) for description.
- `path` (string, required) - action path from MQTT topic name. For example if MQTT topic is "kaa/<application_token>/<extension_instance_id>/<endpoint_token>/pull/json" then "/pull/json" part is the value for `path` field. This is used by extension to determine which function should be applied to message. Also, ECS uses this field to determine destination topic of the response.
- `payload` (bytes, optional) - serialized message content. Can be skipped in `status` message.
- `status` (string, optional) - message status. Main field for `status` message.

The NATS message is a JSON-encoded Avro object.
The Avro schema can be found [here](./ecs2ext-message.avsc).

Example of a message from ECS to extension with payload:
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

Example of a message from ECS to extension with payload and status:
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

Example of a status message from ECS to extension:
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

#### Message from extension to ECS
Extension should send message to ECS using `reply to` NATS field value or to the next subject: `kaa.v1.service.{service-instance-name}`.

In addition to `correlationId`, `timestamp`, and `timeout` fields that are defined in the [targeted messaging IPC design](/0003-messaging-ipc/README.md), extension-to-ECS message contains the following fields:
- `appVersionToken` (string, required) - refer to [??](/) for description.
- `extensionInstanceId` (string, required) - id of the instance that is message's source or destination.
- `endpointId` (string, optional) - refer to [??](/) for description.
- `path` (string, required) - action path from MQTT topic name. For example if MQTT topic is "kaa/<application_token>/<extension_instance_id>/<endpoint_token>/pull/json" then "/pull/json" part is the value for `path` field. This is used by extension to determine which function should be applied to message. Also, ECS uses this field to determine destination topic of the response.
- `payload` (bytes, optional) - serialized message content. Can be skipped in `status` message.
- `status` (string, optional) - message status. Main field for `status` message.

The NATS message is a JSON-encoded Avro object.
The Avro schema can be found [here](./ext2ecs-message.avsc).

Example of a message from extension to ECS with payload:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionToken" : "89556d5962",
  "extensionInstanceId" : "9070ad58-e5e6-482d-bd88-bdf79db23b63",
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
Example of a message from extension to ECS with payload and status:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionToken" : "89556d5962",
  "extensionInstanceId" : "9070ad58-e5e6-482d-bd88-bdf79db23b63",
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
Example of a status message from extension to ECS:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionToken" : "89556d5962",
  "extensionInstanceId" : "9070ad58-e5e6-482d-bd88-bdf79db23b63",
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
- `status` message -- message that contains `status` field value, but `payload` value is missing. Used in delivery confirmation process.
