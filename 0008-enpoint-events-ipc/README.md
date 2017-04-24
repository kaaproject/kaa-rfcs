---
name: Endpoint events IPC
shortname: 8/ENDPOINT_EVENTS
status: draft
editor: Volodymyr Tkhir <vtkhir@cybervisiontech.com>
---

## Introduction

The Endpoint events IPC protocol is a [Broadcast Messaging](/0003-messaging-ipc/README.md#broadcast-messaging) protocol extension.

Document describes the endpoint events that are published into NATS message broker. The document describes the NATS subject and Avro message format for each message.
The NATS subject format is:
  `kaa.v1.events.{originator-service-instance-name}.endpoint.{event-group}.{event-type}`
  where:
  - `{originator-service-instance-name}` - the name of the originator service instance. Allows consumers to subscribe to events from a specific source.
  - `{event-group}` - the logical group of events. For example, connectivity, lifecycle, configuration, metadata, etc.
  - `{event-type}` - specific event type. For example, endpoint connectivity events the types will be connected, disconnected, dormant, awake, etc.

## Endpoint connectivity events
The `{event-group}` is `connectivity`.
### Endpoint connected
Published when endpoint opened the communication session with the Kaa Iot Platform.
The `{event-type}` is `connected`.

NATS subject format:
`kaa.v1.events.{originator-service-instance-name}.endpoint.connectivity.connected`
The Avro schema for the NATS message payload can be found here ([file](./connected.avsc))
### Endpoint disconnected
Published when communication session with the Kaa Iot Platform has been closed.
The `{event-type}` is `disconnected`.

NATS subject format:
`kaa.v1.events.{originator-service-instance-name}.endpoint.connectivity.disconnected`
The Avro schema for the NATS message payload can be found here ([file](./disconnected.avsc))

