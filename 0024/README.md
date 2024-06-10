---
name: Endpoint Relations Management Protocol
shortname: 24/EPRMP
status: raw
editor: Andrew Pasika <apasika@kaaiot.io>
---


<!-- toc -->


# Introduction

Endpoint Relations Management Protocol (EPRMP) is designed for managing endpoint relations between Kaa services.

EPRMP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint relations repository (repository)**: any service that exposes the EPRMP interface to other services for managing endpoint relations.

- **Endpoint relations management client (client)**: any service that uses exposed EPRMP interface.


# Design

## Getting endpoint relations

### Endpoint relations get request

*Endpoint relations get request* message is a [targeted message](/0003/README.md#targeted-messaging) that the client sends to the repository to retrieve endpoint relations.

The client MUST send endpoint relations get request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.eprmp.ep-relations-get-request
```
where:
- `{repository-service-instance-name}` is the endpoint relations repository service instance name.

The client MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.eprmp.ep-relations-get-response
```

Endpoint relations get request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0024-endpoint-relations-get-request.avsc](./0024-endpoint-relations-get-request.avsc)):

```json
{
  "namespace": "org.kaaproject.ipc.eprmp.gen.v1",
  "name": "EndpointRelationsGetRequest",
  "type": "record",
  "doc": "Interservice endpoint relations get request",
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
      "name": "timeout",
      "type": "long",
      "default": 0,
      "doc": "Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
    },
    {
      "name": "endpointId",
      "type": "string",
      "doc": "Identifier of the endpoint, on behalf of which relations are requested"
    },
    {
      "name": "relation",
      "type": "string",
      "doc": "Filters endpoints by relation type and its direction"
    }
  ]
}
```


### Endpoint relations get response

*Endpoint relations get response* message MUST be sent by the repository in response to [Endpoint relations get request message](#endpoint-relations-get-request).
Repository MUST publish endpoint relations get response message to the subject provided in the NATS `replyTo` field of the request.

Endpoint relations get response message payload MUST be an Avro-encoded object with the following schema ([0024-endpoint-relations-get-response.avsc](./0024-endpoint-relations-get-response.avsc)):

```json
{
  "namespace": "org.kaaproject.ipc.eprmp.gen.v1",
  "name": "EndpointRelationsGetResponse",
  "type": "record",
  "doc": "Interservice endpoint relations get response",
  "fields": [
    {
      "name": "correlationId",
      "type": "string",
      "doc": "Message ID primarily used to track message processing across services"
    },
    {
      "name": "timestamp",
      "type": "long",
      "doc": "Message creation unix timestamp in milliseconds"
    },
    {
      "name": "timeout",
      "type": "long",
      "default": 0,
      "doc": "Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration."
    },
    {
      "name":"endpointIds",
      "type":{
        "type":"array",
        "items":"string"
      },
      "default":[],
      "doc":"List of endpoint IDs that has the specified relation to the requested endpoint"
    },
    {
      "name": "statusCode",
      "type": "int",
      "doc": "HTTP status code of the request processing"
    },
    {
      "name": "reasonPhrase",
      "type": [
        "null",
        "string"
      ],
      "default": null,
      "doc": "Human-readable status reason phrase"
    }
  ]
}
```


## Endpoint relations updated event

Originator MUST publish an endpoint relations updated event on any changes to endpoint relations.
The `{event-group}` is `relations`.
The `{event-type}` is `updated`.

Originators MUST publish events to the following NATS subjects:

```
kaa.v1.events.{originator-service-instance-name}.endpoint.relations.updated
```
where `{originator-service-instance-name}` is the name of the originator service instance.
It allows listeners to subscribe to events from a specific originator.

The NATS message payload is an Avro object with the following schema ([0024-endpoint-relations-updated.avsc](./0024-endpoint-relations-updated.avsc)):


### Timeout and retry

There is no guarantee, that request and/or response will be delivered. Client SHOULD implement timeout and retry logic if required.
