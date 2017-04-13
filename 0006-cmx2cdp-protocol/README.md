---
name: Communication protocol between CMX and CDP services
shortname: 6/CMX2CDP
status: raw
editor: Andrew Pasika <apasika@cybervisiontech.com>
---

## Introduction

The CMX2CDP protocol is a [3/Messaging IPC](/0003-messaging-ipc/README.md) protocol extension.

CMX2CDP is the protocol designed to communicate endpoint configurations from _configuration data provider_ 
implementations to the _configuration management extensions_.

## Requirements and constraints

#### Problems and possible solutions

1. _Configuration push._

    Solutions:
    - CDP should broadcast event that new configuration is available.

2. _Configuration delivery confirmation after configuration push._

    Solutions:
    - CMX should broadcast event that particular endpoint has updated its configuration to specific version.
    >**NOTE:** In case of CMX configuration pull no delivery confirmation is required since endpoint can 
    recognise delivery fail and request its configuration again.

3. _It is possible that there are two CDP instances configured to work with different 
content types and one originator service instance. When originator service publishes config event 
message with certain content type it is consumed by both CDPs._

    Solutions:
    - CDP should filter messages by their content types.

4. If CMX asks for endpoint configuration but there is no configuration currently available, 
CDP should somehow inform about it.

    Solutions:
    - Use HTTP status codes and arbitrary reason phrases.
        

#### Key concept

There are two messaging approaches used in CMX-CDP communication:
* [Configuration pull](#configuration-pull)
* [Configuration push](#configuration-push)

### Configuration pull

Is used when CMX is intended to request particular configuration.

#### Subject structure

CMX should send message using NATS to the next subject:

    kaa.v1.service.{cdp-service-instance-name}.cmx2cdp.{message-type}
    
Also, CMX should include NATS `replyTo` field which points to CMX replica that will handle the response:
    
    kaa.v1.replica.{cmx-service-instance-replica-id}.cmx2cdp.{message-type}
    
Refer to [3/Messaging IPC](/0003-messaging-ipc/README.md) for explanation of above subject parts.

#### Targeted message types:

There are next types of such messages:
* _“config-request"_ message - is sent by CMX.
* _“config-response"_ message - is sent by CDP.

_“config-request”_ message structure:

- `correlationId` (string, required) - refer to [3/Messaging IPC](/0003-messaging-ipc/README.md) documentation for description.
- `timestamp` (number, required) - timestamp of the message creation time.
- `timeout` (number, required) - the amount of time from timestamp that message is actual.
- `endpointMessageId` (string, required) - unique identifier of original endpoint message.
- `appVersionName` (string, required) - application version to which endpoint configuration is applicable.
- `endpointId` (string, required) - unique identifier of endpoint to which configuration is applicable.
- `configId` (string, optional) - unique identifier of endpoint configuration. If not present, response message will
hold configuration with latest version.

Example:

```
{
  "correlationId": "07d78e95-2c4d-4899-957c-b9e5a3701fbb",
  "timestamp": 1490303342158,
  "timeout": 3000,
  "endpointMessageId": "6b73cd7c-1de5-4c4e-bbad-b7eda079ccbd",
  "appVersionName": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "configId": "76d34f8b-c038-413f-b122-318dce49edd1"
}
```

_“config-response”_ message structure:

- `correlationId` (string, required) - see [3/Messaging IPC](/0003-messaging-ipc/README.md) for description.
- `timestamp` (number, required) - timestamp of the message creation time.
- `timeout` (number, required) - the amount of time from timestamp that message is actual.
- `endpointMessageId` (string, required) - unique identifier of original endpoint message.
- `appVersionName` (string, required) - application version to which endpoint configuration is applicable.
- `endpointId` (string, required) - endpoint unique identifier to which configuration is applicable.
- `contentType` (string, required) - content type of configuration, e.g.: json, protobuf.
- `configId` (string, required) - unique identifier of endpoint configuration.
- `statusCode` (number, required) - status code that holds meaningful information about the result of inbound message processing.
- `reasonPhrase` (string, optional) - status code that holds meaningful information about the result of inbound message processing.
- `content` (byte[], optional) - content with configuration data.

Example:

```
{
  "correlationId": "07d78e95-2c4d-4899-957c-b9e5a3701fbb",
  "timestamp": 1490303342158,
  "timeout": 3000,
  "endpointMessageId": "6b73cd7c-1de5-4c4e-bbad-b7eda079ccbd",
  "appVersionName": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "contentType": "json",
  "configId": "76d34f8b-c038-413f-b122-318dce49edd1",
  "statusCode": 200,
  "reasonPhrase": "OK",
  "content": {
    "bytes": "d2FpdXJoM2pmbmxzZGtjdjg3eTg3b3cz"
  }
}
```

### Configuration push

Event-based approach as described in [3/Messaging IPC](/0003-messaging-ipc/README.md) documentation can be used as alternative, 
thus manipulations with endpoint configurations are accepted as events.

#### Subject structure

CMX and CDP should listen on and send broadcast messages to the next subject:

    kaa.v1.events.{originator-service-instance-name}.endpoint.config.{event-type}

For subject parts explanation refer to [3/Messaging IPC](/0003-messaging-ipc/README.md).

There are next types of such messages:

* _"updated"_ message - is initiated by CMX when it receives notification that particular endpoint has updated 
configuration. 
* _"new-available"_ message - is initiated by CDP when it receives new configuration.

_“updated”_ message structure:

- `appVersionName` (string, required) - application version to which endpoint configuration is applicable.
- `endpointId` (string, required) - unique identifier of endpoint to which configuration is applicable.
- `configId` (string, required) - unique identifier of endpoint configuration.

Example:

```
{
  "correlationId": "6fd9b270-2b74-428b-a86f-fee018b932f0",
  "eventTimestamp": 1490350896332,
  "originatorReplicaId": "1758dc39-63d2-47d0-9b58-6f617a4e0bba",
  "appVersionName": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "configId": "76d34f8b-c038-413f-b122-318dce49edd1"
}
```

_“new-available”_ message structure:

- `appVersionName` (string, required) - application version to which endpoint configuration is applicable.
- `endpointId` (string, required) - endpoint unique identifier to which configuration is applicable.
- `configId` (string, required) - unique identifier of endpoint configuration.

Example:

```
{
  "correlationId": "0fce883f-1104-4da7-8a35-e790ecced6ac",
  "eventTimestamp": 1490351044418,
  "originatorReplicaId": "3b823589-d90b-497e-91f8-0209ecaef908",
  "appVersionName": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "configId": "76d34f8b-c038-413f-b122-318dce49edd1",
  "contentType": "json",
  "content": {
    "bytes": "d2FpdXJoM2pmbmxzZGtjdjg3eTg3b3cz"
  }
}
```

Regardless of specified above broadcast types all other restriction concerning message 
fields are applicable from [3/Messaging IPC](/0003-messaging-ipc/README.md) documentation.

## Use cases

### UC1

Once CDP service accepts new configuration, it publishes event message 
with configuration content on NATS subject and all other services which have previously subscribed to 
this subject receive the message. On the other side, CDP also listens to config-based 
events and replies on them accordingly.

### UC2

Once endpoint was connected it can ask for its configuration initiating request to CMX which 
forwards it to specific CDP.

## Glossary

- CMX - short name for Configuration Management Extension service.
- CDP - short name for Configuration Data Provider.
- CMX2CDP - name of the protocol used in communication between CMX and any CDP implementation.