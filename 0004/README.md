---
name: Extension Service Protocol
shortname: 4/ESP
status: draft
editor: Dmitry Sergeev <dsergeev@kaaiot.io>
contributors: Andrew Kokhanovskyi <akokhanovskyi@kaaiot.io>, Alexey Shmalko <ashmalko@kaaiot.io>
---

<!-- toc -->


# Introduction

Extension Service Protocol (ESP) is a simple, generic inter-service protocol for Kaa extension services to receive application layer data from and send to clients via the communication services.
ESP helps abstracting out extension services from transport protocol stacks implemented in the communication services.
On the other hand, since Kaa extension services may significantly differ in their messaging patterns or the payload structure, ESP is designed to be unaware of these specifics.

ESP is based on the [targeted messaging IPC design](/0003/README.md#targeted-messaging).


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Communications service**: service that implements the transport protocol stack for communicating with Kaa clients.

- **Extension service**: service that implements the application layer data exchange with Kaa clients.


# Design

## Client data transfer to extension services

ClientData messages are used by communications services for transferring client-originated data to extension services.
Non-affinity session messages MUST be published to a [service instance-wide subject](/0003/README.md#service-instance-wide-subjects):

```
kaa.v1.service.<service-instance-name>.esp.ClientData
```
where `<service-instance-name>` is the target extension service instance name.

In case of affinity sessions, ClientData message MUST be published to the [service replica-specific subject](/0003/README.md#service-replica-specific-subjects) defined by the `replyTo` subject in the previously received message that set up the session affinity.

Communication services implementations SHOULD set `replyTo` subject when sending ClientData messages according to the [session affinity](/0003/README.md#session-affinity) rules.

The NATS message payload is an Avro object with the following schema ([client-data.avsc](./client-data.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.esp.gen.v1",
    "name":"ClientData",
    "type":"record",
    "doc":"Client-originated message from communication to extension services",
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
            "doc":"Application version name the data is sent for"
        },
        {
            "name":"endpointId",
            "type":[
                "string",
                "null"
            ],
            "doc":"Identifier of the endpoint, on behalf of which client sent data. Null for endpoint-unaware extensions."
        },
        {
            "name":"resourcePath",
            "type":"string",
            "doc":"Resource path used for determining the message handling function and the payload format"
        },
        {
            "name":"requestId",
            "type":[
                "int",
                "null"
            ],
            "doc":"Request ID used by endpoint to match messages. Optional."
        },
        {
            "name":"payload",
            "type":"bytes",
            "doc":"Serialized message content"
        }
    ]
}
```

Example:
```json
{
    "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
    "timestamp":1490262793349,
    "timeout":3600000,
    "appVersionName":"humidity-sensor-v3",
    "endpointId":{
        "string":"7ad263ec-3347-4c7d-af89-50c67061367a"
    },
    "resourcePath":"/json",
    "requestId":{
        "int":42
    },
    "payload":"ewogICJpZCI6IDQyLAogICJlbnRyaWVzIjogWwogICAgeyAiaHVtaWRpdHkiOiA4OCB9CiAgXQp9"
}
 ``` 


## Extension data transfer to clients

ExtensionData messages are used by extension services for sending data to clients via communication services.
Non-affinity session messages MUST be published to a [service instance-wide subject](/0003/README.md#service-instance-wide-subjects):

```
kaa.v1.service.<service-instance-name>.esp.ExtensionData
```
where `service-instance-name` is the target communications service instance name.

In case of affinity sessions ExtensionData message MUST be published to the [service replica-specific subject](/0003/README.md#service-replica-specific-subjects) defined by the `replyTo` subject in the previously received message that set up the session affinity.

Extensions MAY also set `replyTo` subject when sending ExtensionData messages according to the [session affinity](/0003/README.md#session-affinity) rules.

The NATS message payload is an Avro object with the following schema ([extension-data.avsc](./extension-data.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.ecs2ext.gen.v1",
    "name":"ExtensionData",
    "type":"record",
    "doc":"Extension-originated messages intended for clients of ECS",
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
            "doc":"Application version name the data is sent for"
        },
        {
            "name":"extensionInstanceName",
            "type":[
                "string",
                "null"
            ],
            "doc":"Name of the extension instance that originated the message"
        },
        {
            "name":"endpointId",
            "type":[
                "string",
                "null"
            ],
            "doc":"Identifier of the endpoint, to which the data is sent. Null for endpoint-unaware extensions."
        },
        {
            "name":"resourcePath",
            "type":"string",
            "doc":"Resource path used for determining the message handling function and the payload format"
        },
        {
            "name":"requestId",
            "type":[
                "int",
                "null"
            ],
            "doc":"Request ID used by endpoint to match messages. Optional."
        },
        {
            "name":"payload",
            "type":[
                "bytes",
                "null"
            ],
            "doc":"Serialized message content"
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
    "timestamp":1490262793349,
    "timeout":3600000,
    "extensionInstanceName":"humidity-sensor-cmx-1",
    "appVersionName":"humidity-sensor-v3",
    "endpointId":{
        "string":"7ad263ec-3347-4c7d-af89-50c67061367a"
    },
    "resourcePath":"/config/json",
    "requestId":{
        "int":42
    },
    "payload":{
        "bytes":"ewogICJzYW1wbGluZyIgOiAyMDAKfQ=="
    },
    "status":200,
    "reasonPhrase":"OK"
}
```
