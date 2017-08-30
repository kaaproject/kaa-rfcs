---
name: Endpoint Lifecycle and Connectivity Events
shortname: 9/ELCE
status: draft
editor: Volodymyr Tkhir <vtkhir@kaaiot.io>
contributors: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


## Introduction

Endpoint events is a method for Kaa services to communicate events that occur with endpoints.
The events are defined as [broadcast messages](/0003/README.md#broadcast-messaging) with `endpoint` target entity type.
This document describes the NATS subject and Avro message format for each event message.


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Event originator (originator)**: service that broadcasts endpoint events.
- **Event listener (listener)**: service that subscribes to endpoint events.


## NATS subject format

Originators MUST use the following NATS subject format for endpoint events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint.{event-group}.{event-type}
```

where:
- `{originator-service-instance-name}` - name of the originator service instance. Allows listeners to subscribe to events from a specific originator.
- `{event-group}` - a logical group of events. For example, connectivity, lifecycle, configuration, metadata, etc.
- `{event-type}` - specific event type. For example, endpoint connectivity event types could be connected, disconnected, dormant, awake, etc.


## Connectivity events

Connectivity events are for notifying listeners about the communication session status transitions between an endpoint and server.
The `{event-group}` is `connectivity`.


### Endpoint connected

Endpoint connected event is published when endpoint opens a communication session with server.
The `{event-type}` is `connected`.

Originators MUST use the following NATS subject format for endpoint connected events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint.connectivity.connected
```

The NATS message payload is an Avro object with the following schema ([connected.avsc](./connected.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.connectivity",
    "name":"EndpointConnectedEvent",
    "type":"record",
    "doc":"Endpoint connected event message",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"timestamp",
            "type":"long",
            "doc":"Message creation UNIX timestamp in milliseconds"
        },
        {
            "name":"timeout",
            "type":"long",
            "default":0,
            "doc":"Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Application version name the endpoint connected with"
        },
        {
            "name":"endpointId",
            "type":[
                "string",
                "null"
            ],
            "doc":"Identifier of the endpoint that connected to the server"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```


### Endpoint disconnected

Endpoint disconnected event is published when endpoint communication session with server becomes unavailable because of a disconnect, timeout, or other reasons.
The `{event-type}` is `disconnected`.

Originators MUST use the following NATS subject format for endpoint disconnected events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint.connectivity.disconnected
```

The NATS message payload is an Avro object with the following schema ([ep-disconnected.avsc](./ep-disconnected.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.connectivity",
    "name":"EndpointDisconnectedEvent",
    "type":"record",
    "doc":"Endpoint(s) disconnected event message",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"timestamp",
            "type":"long",
            "doc":"Message creation UNIX timestamp in milliseconds"
        },
        {
            "name":"timeout",
            "type":"long",
            "default":0,
            "doc":"Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"endpointIds",
            "doc":"Identifiers of the endpoint(s) that got disconnected from the server",
            "type":{
                "type":"array",
                "items":"string",
                "default":[

                ]
            }
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```

When originating service handles a disconnect of a client communication session, associated with multiple endpoints (see [Client vs endpoint](/0001/README.md#client-vs-endpoint)), it SHOULD publish an endpoint disconnected event with all endpoint IDs instead of publishing a separate event for every endpoint.


## Lifecycle events

Lifecycle events are for notifying listeners about the endpoint events such as the initial registration, appversion update, and removal.
The `{event-group}` is `lifecycle`.


### Endpoint registered

The `{event-type}` is `registered`.
Published when an endpoint is registered for the first time in the system.

Originators MUST use the following NATS subject format for endpoint registered events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint.lifecycle.registered
```

The NATS message payload is an Avro object with the following schema ([ep-registered.avsc](./ep-registered.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.lifecycle",
    "name":"RegisteredEvent",
    "type":"record",
    "doc":"Endpoint registered event message",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"timestamp",
            "type":"long",
            "doc":"Message creation UNIX timestamp in milliseconds"
        },
        {
            "name":"timeout",
            "type":"long",
            "default":0,
            "doc":"Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Application version name the endpoint registered with"
        },
        {
            "name":"endpointId",
            "type":[
                "string",
                "null"
            ],
            "doc":"Identifier of the endpoint that registered with the server"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```


### Endpoint application version updated

The `{event-type}` is `appversion-updated`.

Published when endpoint registered in new application version.

Originators MUST use the following NATS subject format for endpoint events:

`kaa.v1.events.{originator-service-instance-name}.endpoint.lifecycle.appversion-updated`

The NATS message payload is an Avro object with the following schema ([ep-appversion-updated.avsc](./ep-appversion-updated.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.lifecycle",
    "name":"AppVersionUpdatedEvent",
    "type":"record",
    "doc":"Endpoint's application version updated event message",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"timestamp",
            "type":"long",
            "doc":"Message creation UNIX timestamp in milliseconds"
        },
        {
            "name":"timeout",
            "type":"long",
            "default":0,
            "doc":"Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Application version name the endpoint updated to"
        },
        {
            "name":"previousAppVersionName",
            "type":"string",
            "doc":"The previous application version name the endpoint was registered in"
        },
        {
            "name":"endpointId",
            "type":[
                "string",
                "null"
            ],
            "doc":"Identifier of the endpoint that registered with the server"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```


### Endpoint unregistered

The `{event-type}` is `unregistered`.
Published when an endpoint is unregistered from the system.

Originators MUST use the following NATS subject format for endpoint unregistered events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint.lifecycle.unregistered
```

The NATS message payload is an Avro object with the following schema ([ep-unregistered.avsc](./ep-unregistered.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.lifecycle",
    "name":"EndpointUnregisteredEvent",
    "type":"record",
    "doc":"Endpoint unregistered event message",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"timestamp",
            "type":"long",
            "doc":"Message creation UNIX timestamp in milliseconds"
        },
        {
            "name":"timeout",
            "type":"long",
            "default":0,
            "doc":"Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Application version name the endpoint unregistered from"
        },
        {
            "name":"endpointId",
            "type":[
                "string",
                "null"
            ],
            "doc":"Identifier of the endpoint that unregistered from the server"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```
