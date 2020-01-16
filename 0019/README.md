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

## Endpoint metadata get request

*Endpoint metadata get request* message is a [targeted message](/0003/README.md#targeted-messaging) that the client sends to the repository to retrieve endpoint metadata.

The client MUST send endpoint metadata get request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.epmmp.ep-metadata-get-request
```
where:
- `{repository-service-instance-name}` is the endpoint metadata repository service instance name.

The client MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.epmmp.ep-metadata-get-response
```

Endpoint metadata get request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0019-endpoint-metadata-get-request.avsc](./0019-endpoint-metadata-get-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.epmmp.gen.v1",
    "name":"EndpointMetadataGetRequest",
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
            "doc":"List of metadata fields. If not specified all fields are included"
        }
    ]
}
```


## Endpoint metadata response

*Endpoint metadata response* message MUST be sent by the repository in response to [Endpoint metadata get  request message](#endpoint-metadata-get-request).
Repository MUST publish endpoint metadata get response message to the subject provided in the NATS `replyTo` field of the request.

Endpoint metadata get response message payload MUST be an Avro-encoded object with the following schema ([0019-endpoint-metadata.avsc](./0019-endpoint-metadata-get-response.avsc)):

```json
{
     "namespace":"org.kaaproject.ipc.epmmp.gen.v1",
     "name":"EndpointMetadataGetResponse",
     "type":"record",
     "doc":"Endpoint metadata get response",
     "fields":[
         {
             "name":"correlationId",
             "type":"string",
             "doc":"Message id primarily used to track message processing across services"
         },
         {
             "name":"timestamp",
             "type":"long",
             "doc":"Message creation unix timestamp in milliseconds"
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
             "name":"payload",
             "type":[
               "null",
               "bytes"
             ],
             "default":null,
             "doc":"Serialized endpoint metadata content"
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

### Timeout and retry

There is no garantee, that request and/or response will be delivered. Client SHOULD implement timeout and retry logic if required.
