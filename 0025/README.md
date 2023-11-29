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
