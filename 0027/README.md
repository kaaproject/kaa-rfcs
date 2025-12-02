---
name: System Configuration Management Protocol
shortname: 27/SCMP
status: raw
editor: Andrew Pasika <apasika@kaaiot.io>
---


<!-- toc -->


# Introduction

System Configuration Management Protocol (SCMP) is designed for broadcasting system configuration updates between Kaa services.

SCMP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Configuration repository (repository)**: any service that manages system configurations and publishes SCMP events on configuration changes.

- **Configuration client (client)**: any service that subscribes to SCMP events to receive system configuration updates.

- **Configuration level**: the hierarchy level at which a configuration is defined. Supported levels are:
  - `TENANT` - tenant-level configuration
  - `APPLICATION` - application-level configuration
  - `APP_VERSION` - application version-level configuration
  - `ENDPOINT` - endpoint-level configuration


# Design

## System configuration updated event

Repository MUST publish a system configuration updated event whenever a system configuration is created or updated at any configuration level.

The `{target-entity-type}` is `system`.
The `{event-group}` is `config`.
The `{event-type}` is `updated`.

Repositories MUST publish events to the following NATS subjects:

```
kaa.v1.events.{repository-service-instance-name}.system.config.updated
```

where `{repository-service-instance-name}` is the name of the repository service instance.
It allows listeners to subscribe to events from a specific repository.

The NATS message payload is an Avro object with the following schema ([0027-system-config-updated.avsc](./0027-system-config-updated.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.system.config",
    "type":"record",
    "name":"SystemConfigUpdated",
    "doc":"Broadcast message to indicate that system configuration data got updated",
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
            "name":"configName",
            "type":"string",
            "doc":"Name of the system configuration that was updated"
        },
        {
            "name":"configLevel",
            "type":"string",
            "doc":"Level of the configuration hierarchy, e.g.: TENANT, APPLICATION, APP_VERSION, ENDPOINT"
        },
        {
            "name":"configLevelId",
            "type":"string",
            "doc":"Identifier at the configuration level, e.g.: tenant ID, application name, app version name, or endpoint ID"
        },
        {
            "name":"contentType",
            "type":"string",
            "default":"application/json",
            "doc":"Type of the configuration data, e.g.: application/json, application/x-protobuf, etc."
        },
        {
            "name":"content",
            "type":"bytes",
            "doc":"Configuration data encoded according to the contentType"
        },
        {
            "name":"originatorReplicaId",
            "type":[
                "null",
                "string"
            ],
            "default":null,
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```


### Configuration levels

The `configLevel` field indicates at which level in the hierarchy the configuration was defined:

| Level | Description | Example `configLevelId` |
|-------|-------------|-------------------------|
| `TENANT` | Configuration applies to an entire tenant | `tenant-abc-123` |
| `APPLICATION` | Configuration applies to a specific application | `smart-meter-app` |
| `APP_VERSION` | Configuration applies to a specific application version | `smart-meter-app-v1.0.0` |
| `ENDPOINT` | Configuration applies to a specific endpoint | `endpoint-xyz-789` |

Configurations at more specific levels (e.g., `ENDPOINT`) typically override configurations at more general levels (e.g., `TENANT`).


### Example

A system configuration updated event for an application-level configuration:

```json
{
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 1701432000000,
  "timeout": 0,
  "configName": "logging-config",
  "configLevel": "APPLICATION",
  "configLevelId": "smart-meter-app",
  "contentType": "application/json",
  "content": "{\"logLevel\": \"DEBUG\", \"maxFileSize\": \"10MB\"}",
  "originatorReplicaId": "ecr-service-replica-1"
}
```
