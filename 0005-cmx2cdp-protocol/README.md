---
name: Kaa Protocol
shortname: 5/CMX2CDP
status: raw
editor: Andrew Pasika <apasika@cybervisiontech.com>
---

## Introduction

The CMX2CDP protocol is a [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) protocol extension.

It is intended to solve the problem of communication between CMX and CDP.

## Requirements and constraints

#### Problems and possible solutions

1. _Configuration delivery confirmation._

    Solutions:
    - CMX should broadcast event that particular endpoint has updated its configuration to specific version.

2. _Configuration push._

    Solutions:
    - CPD should broadcast event that new configuration is available.

3. _It is possible that there are two CPD instances configured to work with different 
content types and one originator service instance. When originator service publishes config event 
message with certain content type it is consumed by both CDPs._

    Solutions:
    - CDP should filter messages by their content types.

4. If endpoint asks for its configuration but there is no configuration currently available, 
CDP should somehow inform endpoint about it.

    Solutions:
    - Use status codes which encapsulate meaningful information: 0 - OK, 1 - FAIL, 2 - NO CONTENT.

#### Key concept

There are two messaging approaches used in CMX-CDP communication:
* [Targeted messaging](#targeted-messaging)
* [Broadcast messaging](#broadcast-messaging)

### Targeted messaging

Is used when CMX is intended to request particular configuration.

#### Subject structure

CMX should send message using NATS to the next subject:

    kaa.{protocol-version}.service.{service-instance-id}
    
Also, CMX should include NATS `replyTo` field which points to CMX replica that will handle the response:
    
    kaa.{protocol-version}.service.{service-instance-id}.{service-replica-id}
    
Refer to [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) for explanation of above subject parts.

#### Targeted message types:

There are next types of such messages:
* _“config-request"_ message - is sent by CMX.
* _“config-response"_ message - is sent by CDP.

_“config-request”_ message structure:

- `correlationId` (string, required) - refer to [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) documentation for description.
- `timestamp` (number, required) - refer to [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) documentation for description.
- `timeout` (number, required) - refer to [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) documentation for description.
- `appVersionToken` (string, required) - application version to which configuration schema is applicable.
- `endpointId` (string, required) - unique identifier of endpoint to which configuration schema is applicable.
- `configVersion` (int, optional) - version of endpoint configuration. If not present, response message will
hold configuration with latest version.

Example:

```
{
  "correlationId": "07d78e95-2c4d-4899-957c-b9e5a3701fbb",
  "timestamp": 1490303342158,
  "timeout": 3000,
  "appVersionToken": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "configVersion": 7
}
```

_“config-response”_ message structure:

- `correlationId` (string, required) - refer to [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) documentation for description.
- `timestamp` (number, required) - refer to [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) documentation for description.
- `timeout` (number, required) - refer to [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) documentation for description.
- `appVersionToken` (string, required) - application version to which configuration schema is applicable.
- `endpointId` (string, required) - endpoint unique identifier to which configuration schema is applicable.
- `contentType` (string, required) - content type of configuration schema, e.g.: json, protobuf.
- `configVersion` (int, required) - version of configuration schema.
- `statusCode` (int, required) - status code that holds meaningful information about the result of inbound message processing.
- `content` (byte[], optional) - content with configuration.

Example:

```
{
  "correlationId": "07d78e95-2c4d-4899-957c-b9e5a3701fbb",
  "timestamp": 1490303342158,
  "timeout": 3000,
  "appVersionToken": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "configVersion": 7,
  "statusCode": 0,
  "content": {
    "bytes": "d2FpdXJoM2pmbmxzZGtjdjg3eTg3b3cz"
  }
}
```

### Broadcast messaging

Event-based approach as described in [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) documentation can be used as alternative, 
thus manipulations with configuration schemas are accepted as events.

#### Subject structure

CMX and CDP should listen on and send broadcast messages to the next subject:

    kaa.v1.events.{originator-service-instance-id}.endpoint.config.{event-type}

For subject parts explanation refer to [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md).

There are next types of such messages:

* _"updated"_ message
* _"new-available"_ message 

_“updated”_ message structure:

- `appVersionToken` (string, required) - application version to which configuration schema is applicable.
- `endpointId` (string, required) - unique identifier of endpoint to which configuration schema is applicable.
- `configVersion` (string, required) - version of endpoint configuration.

Example:

```
{
  "correlationId": "6fd9b270-2b74-428b-a86f-fee018b932f0",
  "eventTimestamp": 1490350896332,
  "originatorReplicaId": "1758dc39-63d2-47d0-9b58-6f617a4e0bba",
  "appVersionToken": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "configVersion": 7
}
```

_“new-available”_ message structure:

- `appVersionToken` (string, required) - application version to which configuration schema is applicable.
- `endpointId` (string, required) - endpoint unique identifier to which configuration schema is applicable.
- `configVersion` (string, required) - version of configuration schema.
- `contentType` (string, required) - content type of configuration schema, e.g.: json, protobuf.
- `content` (byte[], required) - content with configuration.

Example:

```
{
  "correlationId": "0fce883f-1104-4da7-8a35-e790ecced6ac",
  "eventTimestamp": 1490351044418,
  "originatorReplicaId": "3b823589-d90b-497e-91f8-0209ecaef908",
  "appVersionToken": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "configVersion": 7,
  "content": {
    "bytes": "d2FpdXJoM2pmbmxzZGtjdjg3eTg3b3cz"
  }
}
```

Regardless of specified above broadcast types all other restriction concerning message 
fields are applicable from [Kaa BB: messaging IPC](/0003-messaging-ipc/README.md) documentation.

## Use cases

### UC1

Once CDP service accepts new configuration, it publishes event message 
with configuration content on NATS subject and all other services which have previously subscribed to 
this subject receive the message. On the other side, CDP also listens to config-based 
events and replies on them accordingly.

### UC2

Once endpoint was connected it can ask for its configuration initiating request to CMX which 
forwards it to specific CDP.

## Flow chart

![](cmx2cdp-ipc.png?raw=true)

## Glossary

- CMX - short name for Configuration Management Extension service.
- CDP - short name for Configuration Data Provider.
- CMX2CDP - name of the protocol used in communication between CMX and any CDP implementation.