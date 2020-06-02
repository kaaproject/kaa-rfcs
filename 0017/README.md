---
name: Service Configuration Management Protocol
shortname: 17/SCMP
status: draft
editor: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


# 17/SCMP: Service Configuration Management Protocol

## Introduction

Service Configuration Management Protocol (SCMP) is designed to communicate configuration update messages from the configuration provider to consumer services in the Kaa platform.
The configuration updates may be tenant-, application-, and appversion-specific.

SCMP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Configuration provider (provider)**: service that provides application- and appversion-specific configuration data and generates configuration update events.

- **Configuration consumer (consumer)**: service that consumes application- and appversion-specific configuration data and subscribes to configuration update events.


## Design

Configuration provider service supplies configuration data to consumer services in a Kaa cluster.
The configuration may be specific to applications, appversions, or tenants.
SCMP defines a NATS-based protocol for provider to notify consumers about the changes to the configuration data.
The mechanism for the configuration data transport is beyond the scope of this RFC.

Some configuration changes, such as addition of a new Kaa application or application version, must be applied cluster-wide.
Other changes may only affect a single Kaa service instance.
Depending on the type of the configuration data update, provider may use broadcast or service-specific configuration update messages.


### Broadcast configuration updates

In order to notify all service instances in a Kaa cluster of a configuration change, provider MUST [broadcast](/0003/README.md#broadcast-messaging) a configuration update event to the following NATS subjects:

```plain
kaa.v1.events.<provider-service-instance-name>.service.configuration.<update-type>
```

where:

- `<provider-service-instance-name>` is the instance name of the provider service.
This allows consumers to subscribe to events from a specific provider.
- `<update-type>` is either `upsert` when configuration data gets created or updated, or `delete` when configuration data is removed from the provider.

The NATS message payload is an Avro object with the following schema ([0017-broadcast_configuration_update.avsc](./0017-broadcast_configuration_update.avsc)):

```json
{
    "namespace": "org.kaaproject.scmp.event.gen.v1.service.configuration",
    "name": "BroadcastConfigurationUpdateEvent",
    "type": "record",
    "doc": "Platform-wide configuration update event message",
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
            "name": "originatorReplicaId",
            "type": "string",
            "doc": "Identifier of the service replica that generated the event"
        },
        {
            "name": "tenantID",
            "type": [
                "null",
                "string"
            ],
            "default": null,
            "doc": "Tenant ID"
        },
        {
            "name": "appName",
            "type": [
                "null",
                "string"
            ],
            "default": null,
            "doc": "Application name"
        },
        {
            "name": "appVerName",
            "type": [
                "null",
                "string"
            ],
            "default": null,
            "doc": "Application version name"
        }
    ]
}
```

Note that `tenantID`, `appName` and `appVerName` fields are declared as optional.
Provider MUST use one of the following permitted value combinations:

| `tenantID` | `appName`  | `appVerName` | **Event cause**                                                                                | **Recommended consumer action**                                                                          |
| ---------- | ---------- | ------------ | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `null`     | `null`     | `null`       | Configuration change in more than one tenant or application.                                   | Update all tenant-, application-, and appversion-specific configurations.                                |
| non-`null` | `null`     | `null`       | Tenant-specific configuration change or change in more than one application within the tenant. | Update tenant-specific configuration and all tenant application- and appversion-specific configurations. |
| non-`null` | non-`null` | `null`       | Application-specific configuration change.                                                     | Update configuration for the specified application.                                                      |
| non-`null` | non-`null` | non-`null`   | Appversion-specific configuration change.                                                      | Update configuration for the specified application version.                                              |


### Targeted configuration updates

In order to notify a specific service instance of a configuration change, provider MUST send a [targeted](/0003/README.md#broadcast-messaging) a configuration update event to one of the following NATS subjects:

```plain
kaa.v1.service.<service-instance-name>.scmp.configuration.<update-type>
```

where:

- `<service-instance-name>` is the instance name of the target service.
- `<update-type>` is either `upsert` when configuration data gets created or updated, or `delete` when configuration data is removed from the provider.

The NATS message payload is an Avro object with the following schema ([0017-service_specific_configuration_update.avsc](./0017-service_specific_configuration_update.avsc)):

```json
{
    "namespace": "org.kaaproject.scmp.event.gen.v1.service.configuration",
    "name": "ServiceSpecificConfigurationUpdateEvent",
    "type": "record",
    "doc": "Service specific configuration update event message",
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
            "name": "tenantID",
            "type": "string",
            "doc": "Tenant ID"
        },
        {
            "name": "appName",
            "type": [
                "null",
                "string"
            ],
            "default": null,
            "doc": "Application name"
        },
        {
            "name": "appVerName",
            "type": [
                "null",
                "string"
            ],
            "default": null,
            "doc": "Application version name"
        }
    ]
}
```

Note that `appName` and `appVerName` fields are declared as optional.
Provider MUST use one of the following permitted value combinations:

| `appName`  | `appVerName` | **Event cause**                                                                                | **Recommended consumer action**                                                                          |
| ---------- | ------------ | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `null`     | `null`       | Tenant-specific configuration change or change in more than one application within the tenant. | Update tenant-specific configuration and all tenant application- and appversion-specific configurations. |
| non-`null` | `null`       | Application-specific configuration change.                                                     | Update configuration for the specified application.                                                      |
| non-`null` | non-`null`   | Appversion-specific configuration change.                                                      | Update configuration for the specified application version.                                              |


### No receipt acknowledgement

Consumers MUST NOT acknowledge the receipt of configuration update messages.
