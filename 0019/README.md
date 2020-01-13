---
name: Endpoint Metadata Management Protocol
shortname: 19/EPMMP
status: raw
editor: Vitalii Kozlovskyi <vkozlovskyi@kaaiot.io>
---


<!-- toc -->


# Introduction

Endpoint Metadata Management Protocol (EPMMP) is designed for managing endpoint metadata between Kaa services.

EPMMP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint metadata repository (repository)**: any service that exposes the EPMMP interface to other services for managing endpoint metadata.

- **Endpoint metadata management client (client)**: any service that uses exposed EPMMP interface to manage endpoint metadata.


# Design

## Get metadata


In order to request endpoint metadata, clients MUST [emit](/0003/README.md#targeted-messaging) `MetadataRequest` message to the following NATS subject:
```
kaa.v1.service.<metadata-service-instance-name>.epmmp.metadata-request
```
Where:
- `<metadata-service-instance-name>` is the instance name of endpoint registry or other metadata service.

The NATS message payload is an Avro object with the following schema ([0019-get-metadata.avsc](./0019-get-metadata.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.epmmp.gen.v1",
    "name":"GetMetadata",
    "type":"record",
    "doc":"Interservice endpoint metadata get request",
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
            "name":"endpointId",
            "type": "string",
            "doc":"Identifier of the endpoint, on behalf of which metadata is requested"
        },
        {
            "name":"include",
            "type":{
                "type":"array", 
                "items":"string"
            }, 
            "default":[],
            "doc":"List of metadata fields. If not specified all fields are icluded"
        }
    ]
}
```

Client MUST specify `replyTo` NATS subject in the emit message according to the [session affinity design](/0003/README.md#session-affinity).


## Metadata Response 
To receive a response, the client MUST be subscribed to the replyTo subject specified in the request message.

The NATS message payload is an Avro object with the following schema ([0019-metadata.avsc](./0019-metadata.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.epmmp.gen.v1",
    "name":"Metadata",
    "type":"record",
    "doc":"Endpoint metadata response sent to specific `replyto` subject",
    "fields":[
        {
            "name":"correlationId",
            "type":"string",
            "doc":"message id primarily used to track message processing across services"
        },
        {
            "name":"timestamp",
            "type":"long",
            "doc":"message creation unix timestamp in milliseconds"
        },
        {
            "name":"timeout",
            "type":"long",
            "default":0,
            "doc":"amount of milliseconds since the timestamp until the message expires. value of 0 is reserved to indicate no expiration."
        },
        {
            "name":"endpointId",
            "type": "string",
            "doc":"identifier of the endpoint, on behalf of which metadata is requested"
        },
        {
            "name":"payload",
            "type":"bytes",
            "doc":"serialized endpoint metadata content"
        },
        {
            "name":"statuscode",
            "type":"int",
            "doc":"http status code of the request processing"
        },
        {
            "name":"reasonphrase",
            "type":[
              "null",
              "string"
            ],
            "default":null,
            "doc":"human-readable status reason phrase"
        }
    ]
}
```

### Timeout and retry
There is no garantee, that request and/or response will be delivered. Client SHOULD implement timeout and retry logic if required.
