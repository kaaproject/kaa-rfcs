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

## Introduction

Configuration Data Transport (CDT) protocol is designed to communicate endpoint configuration data between Kaa services.

In Kaa architecture, some services are designed to manage endpoint configuration in different ways — store it, process it, publish it, etc.
One of the core features in such context is the ability to send and receive endpoint configurations.

For this purpose, the CDT protocol is designed.
It defines the endpoint configuration data structure so that it's interpreted by all involved services in the same way.

Sending and receiving configuration data occurs within 2 communication patterns — *configuration push* and *configuration pull*.
See [Language](#language).

Design of the CDT protocol communication with other services is based on the [3/Messaging IPC][3/MIPC] protocol.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **EP configuration provider (provider)**: any service that sends an endpoint configuration to the other service.
- **EP configuration consumer (consumer)**: any service that receives an endpoint configuration from the other service.
- **Configuration push**: a communication pattern where provider broadcasts EP configuration data under a certain topic over the [NATS](http://nats.io/) protocol.
The broadcasted data will be received by the configuration consumer services that are subscribed to that topic.
- **Configuration pull**: a communication pattern where consumer requests an EP configuration data from a provider.

## Requirements and constraints

CDT requirements:

- In case of a configuration push, the provider must receive a receipt acknowledgement from the consumer.
- When a consumer tries to pull an endpoint configuration that does not exist, or in case of other errors during configuration pull, HTTP status codes and arbitrary reason phrases must be used to inform about the errors occurred.

## Design

### Configuration pull

Configuration pull is used when a service intends to request a particular EP configuration.

No delivery confirmation is required for configuration pull, as endpoint can detect delivery failure and request configuration again.

#### Subject structure

The [EP configuration consumer](#language) should send messages using this NATS subject:
```
kaa.v1.service.{cdp-service-instance-name}.cmx2cdp.{message-type}
```

Also, the consumer should include NATS `replyTo` field pointing to the consumer service replica that will handle the response:
```
kaa.v1.replica.{cmx-service-instance-replica-id}.cmx2cdp.{message-type}
```

For more information, see [3/Messaging IPC][3/MIPC].

#### Targeted message types

There are two types of targeted messages:
- `ConfigRequest` message is sent by EP configuration consumer.
- `ConfigResponse` message is sent by [EP configuration provider](#language) service.

`ConfigRequest` message structure:

- `correlationId` (string, required): message ID primarily used to track message processing across services.
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

- `correlationId` (string, required): message ID primarily used to track message processing across services.
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
- `applied` (boolean, required): shows if client has latest configuration (true if it does).

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
  },
  "applied": true
}
```

### Configuration push

CDP MUST publish broadcast event when new configuration is available and wait for delivery
confirmation in order to mark configuration as applied.

Configuration push is based on configuration events as described in
[9/Endpoint events IPC](/0009-endpoint-events-ipc/README.md#configuration-events).

[3/MIPC]: /0003-messaging-ipc/README.md