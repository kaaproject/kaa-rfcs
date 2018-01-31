---
name: Endpoint Filter Events
shortname: 18/EFE
status: draft
editor: Oleksandr Didukh <odidukh@kaaiot.io>
---

<!-- toc -->


# Introduction

Endpoint Filter events is a method for Kaa services to communicate events that occur to endpoint filters.
The events are defined as [broadcast messages](/0003/README.md#broadcast-messaging) with `endpoint-filter` target entity type.
This document describes the NATS subject and Avro message format for each endpoint filter event message.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Event originator (originator)**: service that broadcasts endpoint filter events.
- **Event listener (listener)**: service that subscribes to endpoint filter events.


# NATS subject format

Originators MUST publish endpoint filter events to the following NATS subjects:
```
kaa.v1.events.{originator-service-instance-name}.endpoint-filter.{event-group}.{event-type}
```

where:
- `{originator-service-instance-name}` is the name of the originator service instance. Allows listeners to subscribe to endpoint filter events from a specific originator.
- `{event-group}` is a logical group of endpoint filter events. Equal to an endpoint filter identifier.
- `{event-type}` is a specific endpoint filter event type.


# Endpoint filter events


## Endpoint filter activated

Endpoint filter activated event is published when endpoint filter status changed from `pending` to `active`.
The `{event-type}` is `activated`.

Originators MUST use the following NATS subject format for endpoint filter activated events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint-filter.{endpoint-filter-id}.activated
```

The NATS message payload is an Avro object with the following schema ([0018-epf-activated.avsc](./0018-epf-activated.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.filter",
    "name":"EndpointFilterActivatedEvent",
    "type":"record",
    "doc":"Endpoint filter activated event message",
    "fields":[
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
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        },
        {
            "name":"id",
            "type":"string",
            "doc":"Identifier of the endpoint filter"
        },
        {
            "name":"applicationName",
            "type":"string",
            "doc":"Application name the endpoint registered with"
        },
        {
            "name":"name",
            "type":[
              "null",
              "string"
            ],
            "doc":"Endpoint filter name, unique in scope of application"
        }
    ]
}
```


## Endpoint filter deactivated(pending)

Endpoint filter deactivated event is published when endpoint filter status changed from `active` to `pending`.
The `{event-type}` is `deactivated`.

Originators MUST use the following NATS subject format for endpoint filter deactivated events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint-filter.{endpoint-filter-id}.deactivated
```

The NATS message payload is an Avro object with the following schema ([0018-epf-deactivated.avsc](./0018-epf-deactivated.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.filter",
    "name":"EndpointFilterPendingEvent",
    "type":"record",
    "doc":"Endpoint filter deactivated event message",
    "fields":[
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
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        },
        {
            "name":"id",
            "type":"string",
            "doc":"Identifier of the endpoint filter"
        },
        {
            "name":"applicationName",
            "type":"string",
            "doc":"Application name the endpoint registered with"
        },
        {
            "name":"name",
            "type":[
              "null",
              "string"
            ],
            "doc":"Endpoint filter name, unique in scope of application"
        }
    ]
}
```


## Endpoint matched filter

Endpoint matched filter event is published when:

- New endpoint registered and matches existing filter.
- Application version or metadata of endpoint was updated and endpoint now matches to filter.

The `{event-type}` is `ep-matched`.

Originators MUST use the following NATS subject format for endpoint filter deactivated events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint-filter.{endpoint-filter-id}.ep-matched
```

The NATS message payload is an Avro object with the following schema ([0018-epf-ep-matched.avsc](./0018-epf-ep-matched.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.filter",
    "name":"EndpointMatchedFilterEvent",
    "type":"record",
    "doc":"Endpoint matched filter event message",
    "fields":[
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
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        },
        {
            "name":"id",
            "type":"string",
            "doc":"Identifier of the endpoint filter"
        },
        {
            "name":"applicationName",
            "type":"string",
            "doc":"Application name the endpoint registered with"
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Application version name the endpoint registered with"
        },
        {
            "name":"name",
            "type":[
              "null",
              "string"
            ],
            "doc":"Endpoint filter name, unique in scope of application"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Identifier of the endpoint that registered with the server"
        }
    ]
}
```


## Endpoint unmatched filter

Endpoint unmatched filter event is published when:

- Application version or metadata of endpoint was updated and endpoint no longer matches to filter.
- Endpoint was unregistered.

The `{event-type}` is `ep-unmatched`.

Originators MUST use the following NATS subject format for endpoint filter deactivated events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint-filter.{endpoint-filter-id}.ep-unmatched
```

The NATS message payload is an Avro object with the following schema ([0018-epf-ep-unmatched.avsc](./0018-epf-ep-unmatched.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.filter",
    "name":"EndpointUnmatchedFilterEvent",
    "type":"record",
    "doc":"Endpoint unmatched filter event message",
    "fields":[
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
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        },
        {
            "name":"id",
            "type":"string",
            "doc":"Identifier of the endpoint filter"
        },
        {
            "name":"applicationName",
            "type":"string",
            "doc":"Application name the endpoint registered with"
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Application version name the endpoint registered with"
        },
        {
            "name":"name",
            "type":[
              "null",
              "string"
            ],
            "doc":"Endpoint filter name, unique in scope of application"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Identifier of the endpoint that registered with the server"
        }
    ]
}
```
