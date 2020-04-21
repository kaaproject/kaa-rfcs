---
name: Tenant Lifecycle Events
shortname: 21/TLE
status: draft
editor: Oleksandr Didukh <odidukh@kaaiot.io>
---

<!-- toc -->


# Introduction

Tenant lifecycle events is a method for Kaa services to communicate events that occur to tenant-related data update or tenant removal.
The events are defined as [broadcast messages](/0003/README.md#broadcast-messaging) with `tenant` target entity type and `lifecycle` event group.
This document describes the NATS subject and Avro message format for each event message.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).


# NATS subject format

Originators MUST publish tenant events to the following NATS subjects:
```
kaa.v1.events.{originator-service-instance-name}.tenant.lifecycle.{event-type}
```

where:
- `{originator-service-instance-name}` is the name of the originator service instance. It allows listeners to subscribe to events from a specific originator.
- `{event-type}` is a specific event type. For example, tenant lifecycle event types can be `updated` and `unregistered`.


# Tenant updated

The event is published when a tenant-related data was updated.
The `{event-type}` is `updated`.

Originators MUST use the following NATS subject format for tenant updated events:
```
kaa.v1.events.{originator-service-instance-name}.tenant.lifecycle.updated
```

The NATS message payload is an Avro object with the following schema ([0021-tenant-updated.avsc](./0021-tenant-updated.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.tenant.lifecycle",
    "name":"UpdatedEvent",
    "type":"record",
    "doc":"Tenant updated event message",
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
            "name":"tenantId",
            "type":"string",
            "doc":"Identifier of the tenant whose metadata was updated"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```


# Tenant unregistered

The event is published when a tenant is unregistered from the platform.
The `{event-type}` is `unregistered`.

Originators MUST use the following NATS subject format for tenant unregistered events:
```
kaa.v1.events.{originator-service-instance-name}.tenant.lifecycle.unregistered
```

The NATS message payload is an Avro object with the following schema ([0021-tenant-unregistered.avsc](./0021-tenant-unregistered.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.tenant.lifecycle",
    "name":"UnregisteredEvent",
    "type":"record",
    "doc":"Tenant unregistered event message",
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
            "name":"tenantId",
            "type":"string",
            "doc":"Identifier of the tenant that unregistered from the platform"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```
