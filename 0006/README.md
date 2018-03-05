---
name: Configuration Data Transport Protocol
shortname: 6/CDTP
status: draft
editor: Andrew Pasika <apasika@kaaiot.io>
---

<!-- toc -->


# Introduction

Configuration Data Transport Protocol (CDTP) is designed to communicate endpoint configuration data between Kaa services.

CDTP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **EP configuration data provider (configuration provider)**: any service that sends endpoint configuration data to other service(s).
- **EP configuration data consumer (configuration consumer)**: any service that receives endpoint configurations from other service(s).
- **Configuration push**: a communication pattern where a configuration provider broadcasts new EP configuration data in an unsolicited manner.
Configuration consumers may choose to subscribe to the broadcast events and react according to their design.
- **Configuration pull**: a communication pattern where a configuration consumer explicitly requests EP configuration data from a configuration provider.


# Design

## Configuration push

### Configuration updated

*Configuration updated* is a [broadcast message](/0003/README.md#broadcast-messaging) published by configuration provider to indicate that configuration data for an endpoint has been updated.
The message `{event-group}` is `config` and the `{event-type}` is `updated`.

NATS subject format:
```
kaa.v1.events.{configuration-provider-service-instance-name}.endpoint.config.updated
```

*Configuration updated* message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0006-config-updated.avsc](./0006-config-updated.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.config",
    "type":"record",
    "name":"ConfigUpdated",
    "doc":"Broadcast message to indicate that endpoint configuration data got updated in the configuration provider service",
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
            "doc":"Application version name, for which the configuration was updated"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Endpoint identifier, for which the configuration was updated"
        },
        {
            "name":"configId",
            "type":"string",
            "doc":"Identifier of the new configuration"
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

Example:
```json
{
    "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
    "timestamp":1490303342158,
    "timeout":0,
    "appVersionName":"smartKettleV1",
    "endpointId":"b197e391-1d13-403b-83f5-87bdd44888cf",
    "configId":"6046b576591c75fd68ab67f7e4475311",
    "contentType":"application/json",
    "content":"d2FpdXJoM2pmbmxzZGtjdjg3eTg3b3cz"
}
```


### Configuration applied

*Configuration applied* is a broadcast message that configuration consumer MAY publish to indicate that an endpoint applied the configuration data.
Configuration providers MAY subscribe to these messages to keep track of the configurations that were last applied to the endpoints.
The message `{event-group}` is `config` and the `{event-type}` is `applied`.

NATS subject format:
```
kaa.v1.events.{configuration-consumer-service-instance-name}.endpoint.config.applied
```

*Configuration applied* message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0006-config-applied.avsc](./0006-config-applied.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.config",
    "type":"record",
    "name":"ConfigApplied",
    "doc":"Broadcast message to indicate that endpoint applied the configuration",
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
            "doc":"Application version name of the endpoint, which applied the configuration"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Identifier of the endpoint, which applied the configuration"
        },
        {
            "name":"configId",
            "type":"string",
            "doc":"Identifier of the applied configuration"
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

Example:
```json
{
    "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
    "timestamp":1490303342158,
    "timeout":0,
    "appVersionName":"smartKettleV1",
    "endpointId":"b197e391-1d13-403b-83f5-87bdd44888cf",
    "configId":"6046b576591c75fd68ab67f7e4475311"
}
```


## Configuration pull

### Configuration request

Configuration request message is a [targeted message](/0003/README.md#targeted-messaging) that configuration consumer sends to configuration provider to request the EP configuration.

The configuration consumer MUST send configuration request messages using the following NATS subject:
```
kaa.v1.service.{configuration-provider-service-instance-name}.cdtp.request
```

The configuration consumer MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{configuration-consumer-service-replica-id}.cdtp.response
```

For more information, see [3/Messaging IPC][3/MIPC].


*Configuration request* message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0006-config-request.avsc](./0006-config-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.cdtp.gen.v1",
    "type":"record",
    "name":"ConfigRequest",
    "doc":"EP configuration request message from consumer to provider",
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
            "doc":"Amount of milliseconds (since the timestamp) until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Endpoint's application version, for which the configuration is requested"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Endpoint identifier, for which the configuration is requested"
        },
        {
            "name":"configId",
            "type":[
                "null",
                "string"
            ],
            "default":null,
            "doc":"Identifier of the endpoint configuration known to the configuration consumer at the time of the request. Optional. If absent, a non-error response MUST hold the latest configuration data."
        }
    ]
}
```

Example:

```json
{
    "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
    "timestamp":1490303342158,
    "timeout":3000,
    "appVersionName":"smartKettleV1",
    "endpointId":"b197e391-1d13-403b-83f5-87bdd44888cf",
    "configId":"6046b576591c75fd68ab67f7e4475311"
}
```


### Configuration response

*Configuration response* message MUST be sent by configuration provider in response to a [Configuration request message](#configuration-request).
Configuration provider MUST publish configuration response message to the subject provided in the NATS `replyTo` field of the request.

*Configuration response* message payload MUST be an Avro-encoded object with the following schema ([0006-config-response.avsc](./0006-config-response.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.cdtp.gen.v1",
    "type":"record",
    "name":"ConfigResponse",
    "doc":"EP configuration response message from provider to consumer",
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
            "doc":"Amount of milliseconds (since the timestamp) until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Endpoint's application version, for which the configuration is returned"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Endpoint identifier, for which the configuration is returned"
        },
        {
            "name":"configId",
            "type":[
                "null",
                "string"
            ],
            "default":null,
            "doc":"Identifier of the returned configuration. Optional. May be absent in case of an error response, or when the configId in request matches the ID of the current configuration."
        },
        {
            "name":"contentType",
            "type":"string",
            "default":"application/json",
            "doc":"Type of the configuration data, e.g.: application/json, application/x-protobuf, etc."
        },
        {
            "name":"content",
            "type":[
                "null",
                "bytes"
            ],
            "default":null,
            "doc":"Configuration data encoded according to the contentType. Optional. May be absent in case of an error, or when the configId in request matches the ID of the current configuration."
        },
        {
            "name":"applied",
            "type":"boolean",
            "default":false,
            "doc":"Indicates whether the current (returned) configuration ID matches the one previously applied to the endpoint"
        },
        {
            "name":"statusCode",
            "type":"int",
            "doc":"HTTP status code of the request processing"
        },
        {
            "name":"reasonPhrase",
            "type":[
              "null",
              "string"
            ],
            "default":null,
            "doc":"Human-readable status reason phrase"
        }
    ]
}
```

Example:

```json
{
    "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
    "timestamp":1490303342158,
    "timeout":3000,
    "endpointMessageId":42,
    "appVersionName":"smartKettleV1",
    "endpointId":"b197e391-1d13-403b-83f5-87bdd44888cf",
    "configId":"6046b576591c75fd68ab67f7e4475311",
    "contentType":"application/json",
    "content":{
        "bytes":"d2FpdXJoM2pmbmxzZGtjdjg3eTg3b3cz"
    },
    "applied":true,
    "statusCode":200,
    "reasonPhrase":{
        "string":"OK"
    }
}
```

After receiving a configuration response, configuration consumer MAY broadcast a [*configuration applied* event](#configuration-applied) to indicate that the endpoint applied the configuration.

[3/MIPC]: /0003/README.md
