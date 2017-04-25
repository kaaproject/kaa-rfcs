---
name: Endpoint events IPC
shortname: 9/ENDPOINT_EVENTS
status: draft
editor: Volodymyr Tkhir <vtkhir@cybervisiontech.com>
---

* [Introduction](#introduction)
* [Connectivity events](#connectivity-events)
  * [Endpoint connected](#endpoint-connected)
  * [Endpoint disconnected](#endpoint-disconnected)
* [Lifecycle events](#lifecycle-events)
  * [Endpoint registered](#endpoint-registered)
  * [Endpoint updated application version](#endpoint-updated-application-version)
* [Metadata events](#metadata-events)
  * [Endpoint metadata updated](#endpoint-metadata-updated)
* [Configuration events](#configuration-events)
  * [New endpoint configuration available](#new-endpoint-configuration-available)
  * [Endpoint applied the configuration](#endpoint-applied-the-configuration)


## Introduction

The Endpoint events IPC protocol is a [Broadcast Messaging](/0003-messaging-ipc/README.md#broadcast-messaging) protocol extension.

Document describes the endpoint events that are published into NATS message broker. The document describes the NATS subject and Avro message format for each message.
The NATS subject format is:

  `kaa.v1.events.{originator-service-instance-name}.endpoint.{event-group}.{event-type}`

  where:

  - `{originator-service-instance-name}` - the name of the originator service instance. Allows consumers to subscribe to events from a specific source.
  - `{event-group}` - the logical group of events. For example, connectivity, lifecycle, configuration, metadata, etc.
  - `{event-type}` - specific event type. For example, endpoint connectivity events the types will be connected, disconnected, dormant, awake, etc.

## Connectivity events
The `{event-group}` is `connectivity`.
### Endpoint connected
Published when endpoint opened the communication session with the Kaa Iot Platform.
The `{event-type}` is `connected`.

NATS subject format:

`kaa.v1.events.{originator-service-instance-name}.endpoint.connectivity.connected`

The Avro schema for the NATS message payload can be found ([here](./connected.avsc)).
### Endpoint disconnected
The `{event-type}` is `disconnected`.

Published when communication session with the Kaa Iot Platform has been closed.

NATS subject format:

`kaa.v1.events.{originator-service-instance-name}.endpoint.connectivity.disconnected`

The Avro schema for the NATS message payload can be found ([here](./disconnected.avsc)).

The endpoint disconnected event may contain a mutiple endpoint IDs in _endpointIds_ field because endpoint disconnected event can be published when the client (see [Client vs endpoint](/0001-kaa-protocol#client-vs-endpoint)) disconnected. And the client session may be associated with multiple endpoints.

## Lifecycle events
The `{event-group}` is `lifecycle`.
### Endpoint registered

The `{event-type}` is `registered`.

Published when endpoint registered for the first time in the system.

NATS subject format:

`kaa.v1.events.{originator-service-instance-name}.endpoint.lifecycle.registered`

The Avro schema for the NATS message payload can be found ([here](./registered.avsc)).

### Endpoint updated application version

The `{event-type}` is `appversion-updated`.

Published when endpoint registered in new application version.

NATS subject format:

`kaa.v1.events.{originator-service-instance-name}.endpoint.lifecycle.appversion-updated`

The Avro schema for the NATS message payload can be found ([here](./app-version-updated.avsc)).

## Metadata events
The `{event-group}` is `metadata`.
### Endpoint metadata updated

The `{event-type}` is `updated`.

Published on endpoint metadata updates.

NATS subject format:

`kaa.v1.events.{originator-service-instance-name}.endpoint.metadata.updated`

The Avro schema for the NATS message payload can be found ([here](./metadata-updated.avsc)).

## Configuration events
The `{event-group}` is `config`.
### New endpoint configuration available

The `{event-type}` is `new-available`.

Published when new configuration is available for the endpoint.

NATS subject format:

`kaa.v1.events.{originator-service-instance-name}.endpoint.config.new-available`

The Avro schema for the NATS message payload can be found ([here](./new-config-available.avsc)).
### Endpoint applied the configuration

The `{event-type}` is `updated`.

Published when confirmation received from endpoint that configuration was applied.

NATS subject format:

`kaa.v1.events.{originator-service-instance-name}.endpoint.config.updated`

The Avro schema for the NATS message payload can be found ([here](./config-updated.avsc)).


