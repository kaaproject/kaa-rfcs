---
name: Endpoint Filter Management Protocol
shortname: 20/EFMP
status: draft
editor: Oleksandr Didukh <odidukh@kaaiot.io>
---

<!-- toc -->


# Introduction

Endpoint Filter Management Protocol (EFMP) is designed to communicate endpoint filter data between Kaa services.

EFMP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint filter repository (repository)**: service that exposes EFMP to other services for managing endpoint filters.

- **Endpoint filter management client (client)**: service that uses EFMP interface exposed by a repository to manage endpoint filters.


# Design

## Endpoint filters request

*Endpoint filters request* message is a [targeted message](/0003/README.md#targeted-messaging) that client sends to repository to retrieve the endpoint filters.

The client MUST send endpoint filters request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.efmp.ep-filters-request
```

The client MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.efmp.ep-filters-response
```

Endpoint filters request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0020-endpoint-filters-request.avsc](./0020-endpoint-filters-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.efmp.gen.v1",
    "name":"EndpointFiltersRequest",
    "type":"record",
    "doc":"EP filters request message",
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
            "name":"endpointId",
            "type":"string",
            "doc":"Endpoint identifier, for which the filters where requested"
        }
    ]
}
```


## Endpoint filters response

*Endpoint filters response* message MUST be sent by repository in response to an [endpoint filters request message](#endpoint-filters-request).
Repository MUST publish endpoint filters response message to the subject provided in the NATS `replyTo` field of the request.

Endpoint filters response message payload MUST be an Avro-encoded object with the following schema ([0020-endpoint-filters-response.avsc](./0020-endpoint-filters-response.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.efmp.gen.v1",
    "name":"EndpointFiltersResponse",
    "type":"record",
    "doc":"EP filters response message",
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
            "name":"endpointId",
            "type":"string",
            "doc":"Endpoint identifier, for which the filters where requested"
        },
        {
            "name":"filters",
            "type":{
                "type":"array",
                "items":"bytes"
            },
            "doc":"Array of endpoint filters"
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
