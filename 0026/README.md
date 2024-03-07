---
name: Endpoints Application Version Management Protocol
shortname: 26/EAVP
status: draft
editor: Andrew Pasika <apasika@kaaiot.io>
---

<!-- toc -->


# Introduction

Endpoints Application Version Management Protocol (EAVP) is designed for managing endpoints application version between Kaa services.

EAVP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoints application version repository (repository)**: service that exposes EAVP to other services for managing endpoints application version.

- **Endpoints application version management client (client)**: service that uses EAVP interface exposed by a repository to manage endpoints application version.


# Design

## Endpoints application version request

*Endpoints application version request* message is a [targeted message](/0003/README.md#targeted-messaging) that client sends to repository to retrieve endpoints application version.

The client MUST send endpoint  application version request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.eavp.ep-app-version-get-request
```

The client MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.eavp.ep-app-version-get-response
```

Endpoints application version request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0026-app-versions-by-endpoints-request.avsc](./0026-app-versions-by-endpoints-request.avsc)):

```json
{
  "namespace":"org.kaaproject.ipc.eavp.gen.v1",
  "name":"AppVersionsByEndpointsRequest",
  "type":"record",
  "doc":"Application versions by endpoints request message",
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
      "name":"endpointIds",
      "type":{
        "type":"array",
        "items":"string"
      },
      "doc":"Endpoint identifiers for which the application versions were requested"
    }
  ]
}
```


## Endpoint filters response

*Endpoints application version* message MUST be sent by repository in response to an [endpoints application version request message](#endpoints-application-version-request).
Repository MUST publish endpoint filters response message to the subject provided in the NATS `replyTo` field of the request.

Endpoints application version response message payload MUST be an Avro-encoded object with the following schema ([0026-app-versions-by-endpoints-response.avsc](./0026-app-versions-by-endpoints-response.avsc)):
```json
{
  "namespace":"org.kaaproject.ipc.eavp.gen.v1",
  "name":"AppVersionsByEndpointsResponse",
  "type":"record",
  "doc":"Application versions by endpoints response message",
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
      "name":"endpointIds",
      "type":{
        "type":"array",
        "items":"string"
      },
      "doc":"Endpoint identifiers for which the application versions were requested"
    },
    {
      "name":"appVersionsToEndpoints",
      "type":{
        "type":"map",
        "values":{
          "type":"array",
          "items":"string"
        }
      },
      "doc":"Map of application version names to endpoints that match requested endpoints"
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
