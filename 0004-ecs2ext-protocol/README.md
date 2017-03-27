---
name: Communication protocol between Endpoint Communication Service and extension service
shortname: 4/ECS2EXT
status: raw
editor: Dmitry Sergeev <dsergeev@cybervisiontech.com>
---

## Introduction

The ECS2EXT protocol is a [Kaa event-based protocol](/) extension.

It is intended to solve the problem of communication between ECS and any extension service.

## Requirements and constraints
### Problems and possible solutions

1. _Message delivery confirmation._

   Solutions:
   - Protocol contains field named "status" that is used to send error message or confirm successful delivery. 
   
2. _Ability to determine which Avro schema should be used on extension to deserialize `payload` field._ 

   Solutions:
   - MQTT path (e.g. "config/pull", "log/send" etc) will be passed as a `path` field in ECS2EXT protocol.

3. _Ability to send payload in two directions._ 

   Solutions:
   - Protocol message contains `payload` field for both directions: from ECS to extension and vice versa.
   
4. _Ability to send acknowledgement (status) message without payload._

   Solutions:
   - `payload` and `contentType` fields are optional.
   
## Use cases

### UC1
Endpoint can send some data to extension that should not respond with any data excepting status message for delivery confirmation.
In that case `payload` and `contentType` fields will be filled only in messages that goes from ECS to extension. Message from extension to ECS should contain `status` field filled, `payload` and `contentType` can be dropped.
Example of extension with such communication:```Data collection extension```

### UC2
Endpoint can receive messages from extension with `payload` but respond with `status` message only.
Example of such case is push notifications from server to client.

### UC3
Endpoint and extension can send and receive messages with `payload`.


### Formats
#### Avro JSON
![](ecs2ext-ipc.png?raw=true)
##### Message from ECS to extension
ECS should send message using NATS to the next subject:
```
kaa.{protocol-version}.service.{service-instance-id}
```
Also, ECS should include NATS `replyTo` field which should point to ECS replica that will handle response:
```
kaa.{protocol-version}.service.{service-instance-id}.{service-instance-replica-id} 
```
Refer to [Kaa event-based protocol documentation](/) for `protocol-version`, `service-instance-id` and `service-instance-replica-id` parameters.

Format of the ECS-to-extension message:
- `correlationId` (string, required) - refer to [Kaa event-based protocol documentation](/) for description.
- `timestamp` (number, required) - message creation timestamp.
- `timeout` (number, required) - amount of milliseconds from `timestamp`. Value '0' means "the request does not time out".
- `appVersionToken` (string, required) - refer to [??](/) for description.
- `endpointId` (string, optional) - refer to [??](/) for description.
- `contentType` (string, optional) - content-type of the payload content (e.g. "json", "protobuf"). Can be skipped in `status` message.
- `path` (string, required) - action path from MQTT topic name. For example if MQTT topic is "kaa/<application_token>/<extension_instance_id>/<endpoint_token>/pull/json" then "/pull/json" part is the value for `path` field. This is used by extension to determine which function should be applied to message. Also, ECS uses this field to determine destination topic of the response.
- `payload` (bytes, optional) - serialized message content. Can be skipped in `status` message.
- `status` (string, optional) - message status. Main field for `status` message.
Avro schema can be found [here](./ecs2ext-message.avsc).

Example of message from ECS to extension with payload:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionToken" : "89556d5962",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "contentType" : {
    "string" : "json"
  },
  "payload" : {
    "bytes" : "skodg'j394gj3q9i0jg03[09j0[3q[jj39g4"
  },
  "status" : null,
  "path" : "/push/json"
}
```
Example of message from ECS to extension with payload and status:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionToken" : "89556d5962",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "contentType" : {
    "string" : "json"
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
Example of status message from ECS to extension:
```
{
  "correlationId" : "1",
  "timestamp" : 1490262793349,
  "timeout" : 3600000,
  "appVersionToken" : "89556d5962",
  "endpointId" : {
    "string" : "7ad263ec-3347-4c7d-af89-50c67061367a"
  },
  "contentType" : {
    "string" : ""
  },
  "payload" : null,
  "status" : {
    "string" : "ok"
  },
  "path" : "/push/json"
}
```

##### Message from extension to ECS
Extension should send message to ECS using `reply to` NATS field value or to the next subject:
```
kaa.{protocol-version}.service.{service-instance-id}
```
Refer to [Kaa event-based protocol documentation](/) for `protocol-version` and `service-instance-id` parameters.

Format of the extension-to-ECS message:
- `correlationId` (string, required) - refer to [Kaa event-based protocol documentation](/) for description.
- `timestamp` (number, required) - message creation timestamp.
- `timeout` (number, required) - amount of milliseconds from `timestamp`. Value '0' means "the request does not time out".
- `appVersionToken` (string, required) - refer to [??](/) for description.
- `extensionInstanceId` (string, required) - id of the instance that is message's source or destination. 
- `endpointId` (string, optional) - refer to [??](/) for description.
- `contentType` (string, optional) - content-type of the payload content (e.g. "json", "protobuf"). Can be skipped in `status` message.
- `path` (string, required) - action path from MQTT topic name. For example if MQTT topic is "kaa/<application_token>/<extension_instance_id>/<endpoint_token>/pull/json" then "/pull/json" part is the value for `path` field. This is used by extension to determine which function should be applied to message. Also, ECS uses this field to determine destination topic of the response.
- `payload` (bytes, optional) - serialized message content. Can be skipped in `status` message.
- `status` (string, optional) - message status. Main field for `status` message.
Avro schema can be found [here](./ext2ecs-message.avsc).

Example of message from extension to ECS with payload:
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
  "contentType" : {
    "string" : "json"
  },
  "payload" : {
    "bytes" : "skodg'j394gj3q9i0jg03[09j0[3q[jj39g4"
  },
  "status" : null,
  "path" : "/push/json"
}
```
Example of message from extension to ECS with payload and status:
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
  "contentType" : {
    "string" : "json"
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
Example of status message from extension to ECS:
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
  "contentType" : {
    "string" : ""
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
- `status` message -- message that contains `status` field value, but `payload` and `contentType` values are missing. Used in delivery confirmation process.
