---
name: Tenant Metadata Management Protocol
shortname: 21/TMMP
status: draft
editor: Andrew Pasika <apasika@kaaiot.io>
---

<!-- toc -->


# Introduction

Tenant Metadata Management Protocol (TMMP) is designed to communicate tenant metadata between Kaa services.

TMMP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Tenant metadata repository (repository)**: service that exposes TMMP to other services for managing tenant metadata.

- **Tenant metadata client (client)**: service that uses TMMP interface exposed by a repository to manage tenant metadata.


# Design

## Tenant metadata get request

*Tenant metadata get request* message is a [targeted message](/0003/README.md#targeted-messaging) that client sends to repository to retrieve the tenant metadata.

The client MUST send tenant metadata get request messages using the following NATS subject:

```
kaa.v1.service.{repository-service-instance-name}.tmmp.tenant-metadata-get-request
```

The client MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):

```
kaa.v1.replica.{client-service-replica-id}.tmmp.tenant-metadata-get-response
```

Tenant metadata get request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0021-tenant-metadata-get-request.avsc](./0021-tenant-metadata-get-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.tmmp.gen.v1",
    "name":"TenantMetadataGetRequest",
    "type":"record",
    "doc":"Tenant metadata get request",
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
            "name":"appName",
            "type":[
              "null",
              "string"
            ],
            "default":null,
            "doc":"Application that tenant information is requested for"
        },
        {
            "name":"appVersionName",
            "type":[
              "null",
              "string"
            ],
            "default":null,
            "doc":"Application version that tenant information is requested for"
        }
    ]
}
```


## Tenant metadata get response

*Tenant metadata get response* message MUST be sent by repository in response to an [tenant metadata get request](#tenant-metadata-get-request).
Repository MUST publish tenant metadata get response message to the subject provided in the NATS `replyTo` field of the request.

Tenant metadata get response message payload MUST be an Avro-encoded object with the following schema ([0021-tenant-metadata-get-response.avsc](./0021-tenant-metadata-get-response.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.tmmp.gen.v1",
    "name":"TenantMetadataGetResponse",
    "type":"record",
    "doc":"Tenant metadata get response",
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
            "name":"tenantId",
            "type":"string",
            "doc":"Tenant ID"
        },
        {
            "name":"appName",
            "type":"string",
            "doc":"Tenant's application"
        },
        {
            "name":"appVersionName",
            "type":"string",
            "doc":"Tenant's application version"
        }
    ]
}
```
