---
name: Configuration Data Transport protocol
shortname: 6/CDT
status: draft
editor: Andrew Pasika <apasika@cybervisiontech.com>
---

- [Introduction](#introduction)
- [Language](#language)
- [Requirements and constraints](#requirements-and-constraints)
- [Design](#Design)
  - [Configuration pull](#configuration-pull)
    - [Subject structure](#subject-structure)
    - [Targeted message types](#targeted-message-types)
  - [Configuration push](#configuration-push)
    - [Subject structure](#subject-structure)

## Introduction

Configuration Data Transport (CDT) protocol is designed to communicate endpoint configuration data between Kaa services.

In Kaa architecture, some services are designed to manage endpoint configuration in different ways — store it, process it, publish it, etc.
One of the core features in such context is the ability to store and retrieve endpoint configurations.

For this purpose, the CDT protocol was designed.
It defines the endpoint configuration data structure so that it's interpreted by all concerned services in the same way.

In CDT lingo, storing an endpoint configuration to a service is called *configuration push*, while retrieving it from a service is called *configuration pull*.

To effectively perform these activities, endpoint configuration data should be communicated in a format recognized by all the concerned services.
To achieve this, CDT protocol was designed based on the [3/Messaging IPC][3/MIPC] protocol.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Configuration push**: sending an endpoint configuration data to a service for further storage and/or processing.
- **Configuration pull**: retrieving an endpoint configuration data from a service.
- **Source service**: any service that initiates a configuration push/pull.
- **Target service**: any service that a configuration push/pull is initiated towards.

## Requirements and constraints

CDT requirements:

- Once a new endpoint configuration is pushed to a service, this service must be able to broadcast an event over [NATS](http://nats.io/) about the new configuration availability.
- The service that pushed a configuration to another service must receive a confirmation that the configuration was successfully pushed.
- When a service tries to pull an endpoint configuration that does not exist, or in case of other errors during configuration pull, HTTP status codes and arbitrary reason phrases must be used to inform about the errors occurred.

## Design

### Configuration pull

Configuration pull is used when a service is intended to request particular configuration.

No delivery confirmation is required for configuration pull, as endpoint can detect delivery fail and request configuration again.

#### Subject structure

The source service should send messages using this NATS subject:
```
kaa.v1.service.{cdp-service-instance-name}.cmx2cdp.{message-type}
```

Also, the source service should include NATS `replyTo` field pointing to the source service replica that will handle the response:
```
kaa.v1.replica.{cmx-service-instance-replica-id}.cmx2cdp.{message-type}
```

For more information, see [3/Messaging IPC][3/MIPC].

#### Targeted message types

There are two types of targeted messages:
- `ConfigRequest` message is sent by source service.
- `ConfigResponse` message is sent by target service.

`ConfigRequest` message structure:

- `correlationId` (string, required): refer to [3/Messaging IPC][3/MIPC] RFC for description.<!--TODO-->
- `timestamp` (number, required): message creation timestamp.
- `timeout` (number, required): amount of time (starting from the timestamp) before the message gets ignored.
- `endpointMessageId` (string, required): unique identifier of original endpoint message.
- `appVersionName` (string, required): application version to which the endpoint configuration is applicable.
- `endpointId` (string, required): unique identifier of endpoint to which the configuration is applicable.
- `configId` (string, optional): unique identifier of endpoint configuration.
If not present, response message will hold the latest version of configuration.

Example:

```json
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

`ConfigResponse` message structure:

- `correlationId` (string, required): refer to [3/Messaging IPC][3/MIPC] RFC for description.<!--TODO-->
- `timestamp` (number, required): message creation timestamp.
- `timeout` (number, required): amount of time (starting from the timestamp) before the message gets ignored.
- `endpointMessageId` (string, required): unique identifier of original endpoint message.
- `appVersionName` (string, required): application version to which the endpoint configuration is applicable.
- `endpointId` (string, required): unique identifier of endpoint to which the configuration is applicable.
- `contentType` (string, required): type of configuration content, e.g.: JSON, Protocol Buffer.
- `configId` (string, required): unique identifier of endpoint configuration.
- `statusCode` (number, required): status code containing information about the result of inbound message processing.
- `reasonPhrase` (string, optional): status code containing information about the result of inbound message processing.
- `content` (byte[], optional): configuration data.

Example:

```json
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

Manipulations with endpoint configurations can be done as events.
Alternatively, the event-based approach can be used as described in [3/Messaging IPC][3/MIPC] RFC.

When an endpoint updates its configuration to a certain version, a source service should broadcast an event about this.

#### Subject structure

The source and the target services should listen to this subject and send messages to it:
```
kaa.v1.events.{originator-service-instance-name}.endpoint.config.{event-type}
```

For more information, see [3/Messaging IPC][3/MIPC].

There are two types of such messages:
- `ConfigUpdated` message is initiated by source service when it receives notification that particular endpoint has updated configuration.
- `ConfigNewAvailable` message is initiated by target service when it receives new configuration.

`ConfigUpdated` message structure:

- `appVersionName` (string, required): application version to which endpoint configuration is applicable.
- `endpointId` (string, required): unique identifier of endpoint to which configuration is applicable.
- `configId` (string, required): unique identifier of endpoint configuration.

Example:

```json
{
  "correlationId": "6fd9b270-2b74-428b-a86f-fee018b932f0",
  "eventTimestamp": 1490350896332,
  "originatorReplicaId": "1758dc39-63d2-47d0-9b58-6f617a4e0bba",
  "appVersionName": "39774993-a426-4092-9e38-02ec213272d0",
  "endpointId": "b197e391-1d13-403b-83f5-87bdd44888cf",
  "configId": "76d34f8b-c038-413f-b122-318dce49edd1"
}
```

`ConfigNewAvailable` message structure:

- `appVersionName` (string, required): application version to which the endpoint configuration is applicable.
- `endpointId` (string, required): endpoint unique identifier to which the configuration is applicable.
- `configId` (string, required): unique identifier of endpoint configuration.

Example:

```json
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

For more information about message field restrictions, see [3/Messaging IPC][3/MIPC] RFC.

[3/MIPC]: /0003-messaging-ipc/README.md