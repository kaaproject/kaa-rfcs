---
name: Messaging IPC
shortname: 3/MIPC
status: draft
editor: Andrew Kokhanovskyi <akokhanovskyi@cybervisiontech.com>
contributors: Volodymir Tkhir <vtkhir@cybervisiontech.com>
---

<!-- toc -->

## Introduction

Kaa Banana Beach release uses REST / HTTP API and NATS-based messaging for communication between microservices.
This document defines the general design principle of the latter type of messaging.

Messaging IPC is classified into two categories:

- **Targeted messaging** is used in the cases when the message originator has the intention of communicating with a specific service instance.
The sender may specify the exact target service replica, or allow NATS select one or more replicas to deliver the message to - depending on the target service instance subscription strategy.

- **Broadcast messaging** is used when the originator intends to publish message without specifying the exact recipients.
This type of messaging is typically applied to communicate events that occurred and may require reaction from one or more subscribing services.
Multiple service instances may subscribe to a broadcast event feed for receiving copies of messages.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## Targeted messaging

### NATS subjects format
In targeted messaging typically two subscription subject prefixes are used per service instance:

#### Service instance-wide subjects
These subjects are common for all service instance replicas.
Depending on the load balancing strategy, service instance replicas may subscribe in a queue group.
In such case the queue name SHOULD match the service instance identifier.

  The subject format is:

  `kaa.v1.service.{service-instance-name}.{protocol-name}.{message-type}`

  where:
  - `{service-instance-name}` is the target service instance name;
  - `{protocol-name}` is the name of the specific messaging protocol;
  - `{message-type}` is the specific type of the message being sent.

#### Service replica-specific subjects
These subjects are specific for each service instance replica.

  The subject format is:

  `kaa.v1.replica.{service-instance-replica-id}.{protocol-name}.{message-type}`

  where:
  - `{service-instance-replica-id}` is the target service replica name;
  - `{protocol-name}` is the name of the specific messaging protocol;
  - `{message-type}` is the specific type of the message being sent.

### Targeted message common fields

Each targeted message MUST contain the following fields:

- `timestamp` (long, required) - [Unix time](https://en.wikipedia.org/wiki/Unix_time) in milliseconds when the message was created.

- `timeout` (long, optional) - the amount of milliseconds since `timestamp` when the message expires at the originating entity.
`Null` value or `-1` indicates no expiration.

- `correlationId` (String, required) - message tracing ID primarily used for tracking the message processing across services.
This field SHOULD be logged according to the (Kaa logging standards)[].

## Broadcast messaging

### NATS subjects format

The proposed format for message subjects is:

`kaa.v1.events.{originator-service-instance-name}.{target-entity-type}.{event-group}.{event-type}`

where:

- `{originator-service-instance-name}` - the name of the originator service instance.
Allows consumers to subscribe to events from a specific source.

- `{target-entity-type}` - the target entity type on which a specific event occurred.
Examples: endpoint, client, EP group, etc.

- `{event-group}` - the logical group of events.
For example, for endpoint event groups can be connectivity, lifecycle, token (for EP token invalidation notifications across in all services that cached EP tokens), etc.

- `{event-type}` - specific event type.
For example, EP connectivity events the types will be connected, disconnected, dormant, awake, etc.

### Broadcast message common fields

Each broadcast message MUST contain the following fields:

- `timestamp` (long, required) - [Unix time](https://en.wikipedia.org/wiki/Unix_time) in milliseconds when the event was created.

- `timeout` (long, optional) - the amount of milliseconds since `timestamp` when the message expires at the originating entity.
`Null` value or `-1` indicates no expiration.

- `correlationId` (String, required) - message tracing ID primarily used for tracking the message processing across services.
This field SHOULD be logged according to the (Kaa logging standards)[].

- `originatorReplicaId` (String, optional) - identifier of the service replica that generated the event.
Some services may use this field to filter out and ignore events they generated themselves.

### Event listeners

Event listeners may subscribe to wildcard subjects.
For example, `kaa.v1.events.*.endpoint.lifecycle.*` - to subscribe to all endpoint lifecycle events from any originator service

or

`kaa.v1.events.service-instance-name.endpoint.>` - to subscribe to all endpoint events from the service with instance ID `service-instance-name`.
Whenever load balancing across service instance replicas is desired, the service instance name SHOULD be used as a NATS queue group name.
When all replicas should receive the message, no NATS queue group should be specified on subscription.
It is up to event listener implementations to subscribe with or without queue groups.

## Session affinity

Some services may expect responses to the messages (requests) they send.
In cases when it is desired that the response is delivered to the same service replica that originated the request, the requesting service MUST set the NATS `replyTo` to the [targeted service replica-specific subject](#service-replica-specific-subjects), where:
  - `{service-instance-replica-id}` is the requesting service replica identifier;
  - `{protocol-name}` is the name of the messaging protocol;
  - `{message-type}` is the expected type of response message.

Whenever the NATS `replyTo` subject is set in a request message, the recipients MUST use that subject to respond with messages of a matching message type.
In cases when the recipient service needs to respond with a different message type, it is responsible for substituting the last token in the `replyTo` subject accordingly.
