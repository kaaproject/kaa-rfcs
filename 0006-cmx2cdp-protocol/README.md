---
name: CMX to CDP protocol
shortname: 6/CMX2CDP
status: draft
editor: Andrew Pasika<apasika@cybervisiontech.com>
---

## Introduction

CMX to CDP protocol (CMX2CDP) is an extension of [3/Messaging IPC][3/MIPC] protocol.

CMX2CDP is designed to communicate endpoint configurations from [Configuration Data Provider service (CDP)]()<!--TODO--> to [Configuration Management service (CMX)]()<!--TODO-->.

## Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **CMX**: Configuration Management service.
- **CDP**: Configuration Data Provider service.
- **CMX2CDP**: name of the protocol used in communication between CMX and any CDP implementation.

## Requirements and constraints

CMX2CDP requirements:

- Configuration push: broadcasting events about new configuration availability.
- Configuration delivery confirmation.
- Filtering messages by content type and communicating them to different CDP instances.
- Using HTTP status codes and arbitrary reason phrases to inform about configuration unavailability.

## Use cases

### UC1

Once CDP service accepts new configuration, it publishes an event message containing the configuration data.
The message is published with a [NATS](http://nats.io/) subject, so all services subscribed to this subject will receive the event message.

CDP also listens to configuration-based events and replies to them accordingly.

### UC2

Once endpoint is connected, it can send a configuration request to CMX which forwards it to specific CDP.

## Design

There are two messaging approaches used in CMX-CDP communication:
- [Configuration pull](#configuration-pull)
- [Configuration push](#configuration-push)

### Configuration pull

Configuration pull is used when CMX is intended to request particular configuration.

No delivery confirmation is required for configuration pull, as endpoint can detect delivery fail and request configuration again.

#### Subject structure

CMX should send messages using this NATS subject:
```
kaa.v1.service.{cdp-service-instance-name}.cmx2cdp.{message-type}
```

Also, CMX should include NATS `replyTo` field pointing to CMX replica that will handle the response:
```
kaa.v1.replica.{cmx-service-instance-replica-id}.cmx2cdp.{message-type}
```

For more information, see [3/Messaging IPC][3/MIPC].

#### Targeted message types

There are two types of targeted messages:
- `ConfigRequest` message is sent by CMX.
- `ConfigResponse` message is sent by CDP.

`ConfigResponse` message structure:

- `correlationId` (string, required): refer to [3/Messaging IPC][3/MIPC] RFC for description.<!--TODO-->
- `timestamp` (number, required): message creation timestamp.
- `timeout` (number, required): amount of time that the message remains actual — from timestamp to termination.
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
- `timeout` (number, required): amount of time that the message remains actual — from timestamp to termination.
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

When an endpoint updates its configuration to a certain version, CMX should broadcast an event about this.

#### Subject structure

CMX and CDP should listen to this subject and send messages to it:
```
kaa.v1.events.{originator-service-instance-name}.endpoint.config.{event-type}
```

For more information, see [3/Messaging IPC][3/MIPC].

There are two types of such messages:
- `ConfigUpdated` message is initiated by CMX when it receives notification that particular endpoint has updated configuration.
- `ConfigNewAvailable` message is initiated by CDP when it receives new configuration.

`ConfigUpdated` message structure:

- `appVersionName` (string, required) - application version to which endpoint configuration is applicable.
- `endpointId` (string, required) - unique identifier of endpoint to which configuration is applicable.
- `configId` (string, required) - unique identifier of endpoint configuration.

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