---
name: Alert Management Protocol
shortname: 25/AMP
status: raw
editor: Andrew Pasika <apasika@kaaiot.io>
---


<!-- toc -->


# Introduction

Alert Management Protocol (AMP) is designed for managing alerts between Kaa services.

AMP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Alert storage (storage)**: any service that exposes the AMP interface to other services for managing alerts.

- **Alert client (client)**: any service that uses exposed AMP interface.


# Design

## Alert activation

*Activate alert command* message is a [targeted message](/0003/README.md#targeted-messaging) that the client sends to the storage to activate alert.

The client MUST send activate alert command using the following NATS subject:
```
kaa.v1.service.{storage-service-instance-name}.amp.activate-alert
```
where:
- `{storage-service-instance-name}` is the endpoint alert storage service instance name.

Activate alert command payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0025-activate-alert-command.avsc](./0025-activate-alert-command.avsc)):

```json
{
  "namespace": "org.kaaproject.ipc.amp.gen.v1",
  "name": "ActivateAlertCommand",
  "type": "record",
  "doc": "Alert activation command",
  "fields": [
    {
      "name": "correlationId",
      "type": "string",
      "doc": "Message ID primarily used to track message processing across services"
    },
    {
      "name": "timestamp",
      "type": "long",
      "doc": "Message creation UNIX timestamp in milliseconds"
    },
    {
      "name": "timeout",
      "type": "long",
      "default": 0,
      "doc": "Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
    },
    {
      "name": "alertType",
      "type": "string",
      "doc": "Alert type"
    },
    {
      "name": "severityLevel",
      "type": "string",
      "doc": "Alert severity level"
    },
    {
      "name": "entityType",
      "type": "string",
      "doc": "Entity type that alert raised at"
    },
    {
      "name": "entityId",
      "type": "string",
      "doc": "Entity ID that alert raised at"
    },
    {
      "name": "tenantId",
      "type": "string",
      "doc": "Entity's tenant ID that alert raised at"
    },
    {
      "name": "activateReason",
      "type": "string",
      "doc": "Alert activation reason"
    },
    {
      "name": "startedAt",
      "type": "long",
      "default": 0,
      "doc": "Time when alert started. Server timestamp is used if not specified"
    },
    {
      "name": "lastActiveAt",
      "type": "long",
      "default": 0,
      "doc": "Time when alert was last active. Server timestamp is used if not specified"
    }
  ]
}

```


## Alert lifecycle events

*Alert lifecycle event* message is a [broadcast message](/0003/README.md#broadcast-messaging) that the storage publishes after it commits an alert state change.
Listeners, such as the WebSocket service, can use these events to stream alert changes to clients in real time.

The storage MUST publish alert lifecycle events using the following NATS subject:
```
kaa.v1.events.{storage-service-instance-name}.{entity-type}.alert.{event-type}
```
where:
- `{storage-service-instance-name}` is the alert storage service instance name.
- `{entity-type}` is the lowercase type of the entity that the alert is raised at: `endpoint` or `tenant`.
- `{event-type}` is the lowercase, hyphen-separated lifecycle event type:

| `{event-type}`             | `eventType` field          | Published when                          |
|----------------------------|----------------------------|-----------------------------------------|
| `created`                  | `CREATED`                  | A new alert is created                  |
| `active`                   | `ACTIVE`                   | An alert is activated or re-activated   |
| `resolved`                 | `RESOLVED`                 | An alert is resolved                    |
| `acknowledged`             | `ACKNOWLEDGED`             | An alert is acknowledged                |
| `unacknowledged`           | `UNACKNOWLEDGED`           | An alert acknowledgement is revoked     |
| `severity-level-increased` | `SEVERITY_LEVEL_INCREASED` | An active alert's severity level rises  |
| `severity-level-decreased` | `SEVERITY_LEVEL_DECREASED` | An active alert's severity level drops  |

One alert state change MAY produce several events.
For example, a new alert produces `created` and then `active`.
The storage MUST publish events of one state change in the order they occurred.

The storage MUST publish an event only after the state change is durably stored.
Delivery is at most once: listeners MUST NOT rely on receiving every event and SHOULD use the storage's query API to reconcile state.

For example, listeners subscribe to `kaa.v1.events.*.endpoint.alert.*` to receive all endpoint alert lifecycle events from any storage.

Alert lifecycle event payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0025-alert-lifecycle-event.avsc](./0025-alert-lifecycle-event.avsc)):

```json
{
  "namespace": "org.kaaproject.ipc.amp.gen.v1",
  "name": "AlertLifecycleEvent",
  "type": "record",
  "doc": "Alert lifecycle event broadcast by the alert storage",
  "fields": [
    {
      "name": "correlationId",
      "type": "string",
      "doc": "Message ID primarily used to track message processing across services"
    },
    {
      "name": "timestamp",
      "type": "long",
      "doc": "Message creation UNIX timestamp in milliseconds"
    },
    {
      "name": "timeout",
      "type": "long",
      "default": 0,
      "doc": "Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
    },
    {
      "name": "eventId",
      "type": "string",
      "doc": "Unique identifier of the alert lifecycle event"
    },
    {
      "name": "eventType",
      "type": "string",
      "doc": "Alert lifecycle event type: CREATED, ACTIVE, RESOLVED, ACKNOWLEDGED, UNACKNOWLEDGED, SEVERITY_LEVEL_INCREASED, or SEVERITY_LEVEL_DECREASED"
    },
    {
      "name": "eventTimestamp",
      "type": "long",
      "doc": "UNIX timestamp in milliseconds when the lifecycle event occurred"
    },
    {
      "name": "alertId",
      "type": "string",
      "doc": "Alert ID"
    },
    {
      "name": "alertType",
      "type": "string",
      "doc": "Alert type"
    },
    {
      "name": "severityLevel",
      "type": "string",
      "doc": "Alert severity level at the time of the event"
    },
    {
      "name": "tenantId",
      "type": "string",
      "doc": "Tenant ID of the entity that the alert is raised at"
    },
    {
      "name": "entityType",
      "type": "string",
      "doc": "Entity type that the alert is raised at: ENDPOINT or TENANT"
    },
    {
      "name": "entityId",
      "type": "string",
      "doc": "Entity ID that the alert is raised at"
    },
    {
      "name": "appVersionName",
      "type": ["null", "string"],
      "default": null,
      "doc": "Application version name of the endpoint. Present only for ENDPOINT entity type when known."
    },
    {
      "name": "metadata",
      "type": ["null", "string"],
      "default": null,
      "doc": "JSON-encoded event metadata: activation metadata for CREATED and ACTIVE events, resolution metadata for RESOLVED events"
    },
    {
      "name": "systemMetadata",
      "type": {
        "type": "map",
        "values": "string"
      },
      "default": {},
      "doc": "Event system metadata, such as the activation, resolution, or transition reason"
    }
  ]
}
```
