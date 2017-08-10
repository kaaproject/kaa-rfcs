---
name: Configuration Data Transport protocol
shortname: 6/CDT
status: draft
editor: Andrew Pasika <apasika@cybervisiontech.com>
---

<!-- toc -->


## Introduction

Configuration Data Transport (CDT) protocol is designed to communicate endpoint configuration data between Kaa services.

Sending and receiving configuration data occurs within 2 communication patterns — *configuration push* and *configuration pull*.
See [Language](#language).

CDT protocol design is based on the [3/Messaging IPC][3/MIPC].


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **EP configuration data provider (provider)**: any service that sends endpoint configuration data to other service(s).
- **EP configuration data consumer (consumer)**: any service that receives endpoint configurations from other service(s).
- **Configuration push**: a communication pattern where a provider broadcasts new EP configuration data in an unsolicited manner.
Consumers may choose to subscribe to the broadcast events and react according to their design.
- **Configuration pull**: a communication pattern where a consumer explicitly requests EP configuration data from a provider.


## Requirements and constraints

- In case of a configuration push, the provider MAY receive a delivery confirmation from a consumer.
- In case of a configuration pull, the delivery confirmation is OPTIONAL, since the consumer or the target endpoint can detect delivery failure and pull configuration again.
- HTTP status codes and arbitrary reason phrases MUST be used to inform consumer about any errors that occur in the provider during the configuration pull.


## Design

### Configuration push

#### Configuration updated

Configuration updated is a [broadcast message](/0003-messaging-ipc/README.md#Broadcast-messaging) published by provider to indicate that configuration data for an endpoint got updated.
The message `{event-group}` is `config` and the `{event-type}` is `updated`.

NATS subject format is:
```
kaa.v1.events.{provider-service-instance-name}.endpoint.config.updated
```

Configuration updated message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([file](./config-updated.avsc)):

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
            "default":"",
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
            "default":"",
            "doc":"Application version name, for which the configuration was updated"
        },
        {
            "name":"endpointId",
            "type":"string",
            "default":"",
            "doc":"Endpoint identifier, for which the configuration was updated"
        },
        {
            "name":"configId",
            "type":"string",
            "default":"",
            "doc":"Identifier of the new configuration"
        },
        {
            "name":"contentType",
            "type":"string",
            "default":"",
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
    "content":{
        "bytes":"d2FpdXJoM2pmbmxzZGtjdjg3eTg3b3cz"
    }
}
```

#### Configuration delivered

Configuration delivered is a broadcast message that consumer MAY publish to indicate that configuration data was delivered to an endpoint.
Providers MAY subscribe to these messages to keep track of the last delivered to endpoints configurations.
The message `{event-group}` is `config` and the `{event-type}` is `delivered`.

NATS subject format is:
```
kaa.v1.events.{consumer-service-instance-name}.endpoint.config.delivered
```

Configuration delivered message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([file](./config-delivered.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.config",
    "type":"record",
    "name":"ConfigDelivered",
    "doc":"Broadcast message to indicate that configuration was delivered to an endpoint",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "default":"",
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
            "default":"",
            "doc":"Application version name, for which the configuration was delivered"
        },
        {
            "name":"endpointId",
            "type":"string",
            "default":"",
            "doc":"Endpoint identifier, for which the configuration was delivered"
        },
        {
            "name":"configId",
            "type":"string",
            "default":"",
            "doc":"Identifier of the delivered configuration"
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

### Configuration pull

#### Configuration request

Configuration request message is a [targeted message](/0003-messaging-ipc/README.md#Targeted-messaging) that is sent by consumer to provider to request the EP configuration.

The consumer MUST send configuration request messages using the following NATS subject:
```
kaa.v1.service.{provider-service-instance-name}.cdt.request
```

The consumer MUST include NATS `replyTo` field pointing to the consumer service replica that will handle the response:
```
kaa.v1.replica.{consumer-service-replica-id}.cdt.response
```

For more information, see [3/Messaging IPC][3/MIPC].


Configuration request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([file](./config-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.cdt.gen.v1",
    "type":"record",
    "name":"ConfigRequest",
    "doc":"EP configuration request message from consumer to provider",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "default":"",
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
            "default":"",
            "doc":"Endpoint's application version, for which the configuration is requested"
        },
        {
            "name":"endpointId",
            "type":"string",
            "default":"",
            "doc":"Endpoint identifier, for which the configuration is requested"
        },
        {
            "name":"configId",
            "type":[
                "null",
                "string"
            ],
            "default":null,
            "doc":"Identifier of the endpoint configuration known to the consumer at the time of the request. Optional. If absent, a non-error response MUST hold the latest configuration data."
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
    "configId":"6046b576591c75fd68ab67f7e4475311"
}
```

#### Configuration response

Configuration response message SHOULD be sent by provider in response to a [Configuration request message](#Configuration-request).
Provider MUST publish configuration response message to the subject provided in the NATS `replyTo` field of the request.

Configuration response message payload MUST be an Avro-encoded object with the following schema ([file](./config-response.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.cdt.gen.v1",
    "type":"record",
    "name":"ConfigResponse",
    "doc":"EP configuration response message from provider to consumer",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "default":"",
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
            "default":"",
            "doc":"Endpoint's application version, for which the configuration is returned"
        },
        {
            "name":"endpointId",
            "type":"string",
            "default":"",
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
            "default":"",
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
            "doc":"Indicates whether the current (returned) configuration ID matches the one previously delivered to the endpoint"
        },
        {
            "name":"statusCode",
            "type":"int",
            "default":200,
            "doc":"HTTP status code of the request processing"
        },
        {
            "name":"reasonPhrase",
            "type":"string",
            "default":"",
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
    "reasonPhrase":"OK"
}
```

After receiving a configuration response, consumer MAY broadcast a [Configuration delivered event](#Configuration-delivered) to indicate that the configuration was delivered to the endpoint.

[3/MIPC]: /0003-messaging-ipc/README.md