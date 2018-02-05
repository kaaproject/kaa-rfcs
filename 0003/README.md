---
name: Inter-Service Messaging
shortname: 3/ISM
status: draft
editor: Andrew Kokhanovskyi <ak@kaaiot.io>
contributors: Volodymir Tkhir <vtkhir@kaaiot.io>
---

<!-- toc -->


# Introduction

This document defines general design recommendations for the [NATS-based](https://nats.io/) inter-service messaging protocols used by Kaa platform.

NATS-based messaging is classified into two categories:

- **Targeted messaging** is used when the message originator intentds to communicate with a specific service instance.
The sender may specify the exact target service replica, or allow NATS select one or more replicas to deliver the message to — depending on the subscription strategy of the target service instance.

- **Broadcast messaging** is used when the originator intends to publish message without specifying the exact recipients.
This type of messaging is typically applied to communicate events that occurred and may require reaction from one or more subscribing services.
Multiple service instances may subscribe to a broadcast event feed to receive message copies.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Event originator (originator)**: service that broadcasts endpoint filter events.
- **Event listener (listener)**: service that subscribes to endpoint filter events.


# Targeted messaging

## NATS subject format

### Service instance-wide subjects

Service instance-wide subjects are common for all replicas of a service instance.
Depending on the load balancing strategy, service instance replicas MAY subscribe in a queue group.
In such case the queue name SHOULD match the service instance identifier.

To exchange service instance-wide messages, services MUST use the following format for message subjects:
```
kaa.v1.service.{service-instance-name}.{protocol-name}.{message-type}
```

where:
- `{service-instance-name}` is the name of the target service instance.
- `{protocol-name}` is the name of the messaging protocol.
- `{message-type}` is the type of the message being sent.


### Service replica-specific subjects

Service replica-specific subjects are unique for each replica of a service instance.

To exchange service replica-specific messages, services MUST use the following format for message subjects:
```
kaa.v1.replica.{service-instance-replica-id}.{protocol-name}.{message-type}
```

where:
- `{service-instance-replica-id}` is the name of the target service replica.
- `{protocol-name}` is the name of the messaging protocol.
- `{message-type}` is the type of the message being sent.


## Targeted message common fields

Each targeted message SHOULD contain the following fields.

- `timestamp` (long, required): [Unix timestamp](https://en.wikipedia.org/wiki/Unix_time) in milliseconds specifying when the message was created.

- `timeout` (long, optional): period of time, in milliseconds, between the message timestamp and the message expiry.
Value of 0 is reserved to indicate no expiration.

- `correlationId` (string, required): message ID primarily used to track message processing across services.
This field SHOULD be logged according to the [Kaa logging standards](<!--TODO-->).


# Broadcast messaging

## NATS subject format

To broadcast messages, services MUST use the following prefix for message subject format:
```
kaa.v1.events.{originator-service-instance-name}.{target-entity-type}.{event-group}.{event-type}
```

where:

- `{originator-service-instance-name}` is the name of the originator service instance.
Allows listeners to subscribe to events from a specific source.

- `{target-entity-type}` is the type of the target entity in which a specific event occurred.
Examples: endpoint, client, EP group, etc.

- `{event-group}` is the logical group of events.
E.g. for endpoint event groups, it can be events related to connectivity, lifecycle, token (to send notifications of EP token invalidation across all services that cached EP tokens), etc.

- `{event-type}` is the specific event type.
For example, types of EP connectivity events are *connected*, *disconnected*, *dormant*, *awake*, etc.

Extra subject tokens MAY be appended to the defined above prefix in specific protocol implementations.


## Broadcast message common fields

Each broadcast message SHOULD contain the following fields.

- `timestamp` (long, required): [Unix timestamp](https://en.wikipedia.org/wiki/Unix_time) in milliseconds specifying when the message was created.

- `timeout` (long, optional): period of time, in milliseconds, between the message timestamp and the message expiry.
Value of 0 is reserved to indicate no expiration.

- `correlationId` (string, required): message ID primarily used to track message processing across services.
This field SHOULD be logged according to the [Kaa logging standards](<!--TODO-->).

- `originatorReplicaId` (string, optional): identifier of the service replica that generated the event.
Some services MAY use this field to filter out and ignore events they generated themselves.


## Event listeners

Event listeners MAY subscribe to wildcard subjects.
For example, `kaa.v1.events.*.endpoint.lifecycle.*` — to subscribe to all endpoint lifecycle events from any originator service,

or

`kaa.v1.events.service-instance-name.endpoint.>` — to subscribe to all endpoint events from the service with instance ID `service-instance-name`.
Whenever load balancing across service instance replicas is desired, the service instance name SHOULD be used as a NATS queue group name.
When all replicas receive the message, no NATS queue group should be specified in the subscription.
It is up to event listener implementations, whether to subscribe with or without the queue groups.


# Session affinity

Some services may expect responses to the messages (requests) they send.
When it is desired that the response is delivered to the same service replica that originated the request, the requesting service SHOULD set the NATS `replyTo` to the [targeted service replica-specific subject](#service-replica-specific-subjects), where:
- `{service-instance-replica-id}` is the identifier of the requesting service replica.
- `{protocol-name}` is the name of the messaging protocol.
- `{message-type}` is the expected type of response message.

Whenever the NATS `replyTo` subject is set in a request message, the recipients SHOULD use that subject to respond with messages of a matching message type.
In cases when the recipient service needs to respond with a different message type, it SHOULD substitute the last section in the `replyTo` subject with the corresponding message type.
