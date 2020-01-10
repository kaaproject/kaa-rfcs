---
name: Metadata Transport Protocol
shortname: 19/MDTP
status: raw
editor: Vitalii Kozlovskyi <vkozlovskyi@kaaiot.io>
---


<!-- toc -->


# Introduction

Metadata Transport Protocol (MDTP) is designed to request device metadata from Endpoint Registry (EPR) and receive.

MDTP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint metadata request (request):** endpoint identification and list of required metadata fields

- **Endpoint metadata response (response):** block of serialised device metadata

- **Data transmitter service (server)**: service that exposes MDTP interface to other services for access to device metadata

- **Data receiver service (client)**: service that uses exposed MDTP interface to get device metadata.


# Design

## Request metadata


In order to request endpoint metadata, clients MUST [emit](/0003/README.md#targeted-messaging) `MetadataRequest` message to the following NATS subject:
```
kaa.v1.service.<metadata-service-instance-name>.mdtp.metadata-request
```
Where:
- `<metadata-service-instance-name>` is the instance name of endpoint registry or other metadata service.

The NATS message payload is an Avro object with the following schema ([0019-metadata-request.avsc](./0019-metadata-request.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.mdtp.gen.v1",
    "name":"MetadataRequest",
    "type":"record",
    "doc":"Interservice metadata request",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services. In context of mdtp also used to match request and response events"
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
            "name":"endpointId",
            "type": "string",
            "doc":"Identifier of the endpoint, on behalf of which metadata is requested"
        },
        {
            "name":"fields",
            "type":{
                "type":"array", 
                "items":"string"
            }, 
            "default":[],
            "doc":"List of metadata fields. If not specified all fields are returned."
        },
        {
            "name":"replyTo",
            "type":"string",
            "doc":"NATS inbox name of replica according to 3/ISM session affinity."
        }
    ]
}
```

Client MUST specify `replyTo` NATS subject in the emit message according to the [session affinity design](/0003/README.md#session-affinity).
```
kaa.v1.replica.<service-instance-replica-id>.mdtp.metadata-response
```
Where:
- `<service-instance-replica-id>` is the name of current instance replica.


## Metadata Response 
Server MAY process requests in any order and client MUST match Response by `correlationId`

To read a response, service MUST be subscribed to NATS topic, specified in `replyTo` field of request message.

The NATS message payload is an Avro object with the following schema ([0019-metadata-response.avsc](./0019-metadata-response.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.mdtp.gen.v1",
    "name":"MetadataResponse",
    "type":"record",
    "doc":"Metadata response sent to specific `replyTo` replica.",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"Message ID primarily used to track message processing across services. In context of mdtp shall be used to match request and response events"
        },
        {
            "name":"payload",
            "type":"bytes",
            "doc":"Serialized metadata content"
        }
    ]
}
```

### Timeout and retry
There is no garantee, that request and/or response will be delivered. Client SHOULD implement timeout and retry logic if required.
