---
name: Endpoint Metadata Management Protocol
shortname: 19/EPMMP
status: raw
editor: Vitalii Kozlovskyi <vkozlovskyi@kaaiot.io>, Dmitry Romantsov <dromantsov@kaaiot.io>
---


<!-- toc -->


# Introduction

Endpoint Metadata Management Protocol (EPMMP) is designed for managing endpoint metadata between Kaa services.

EPMMP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint metadata repository (repository)**: any service that exposes the EPMMP interface to other services for managing endpoint metadata.

- **Endpoint metadata management client (client)**: any service that uses exposed EPMMP interface.


# Design

## Getting endpoint metadata

### Endpoint metadata get request

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
            "doc":"List of the endpoint metadata keys. If not specified all metadata will be returned"
        },
        {
            "name":"includeMetadata",
            "type":"boolean",
            "default": true,
            "doc":"Whether to include metadata for this endpoint in response"
        }
    ]
}
```


### Endpoint metadata get response

*Endpoint metadata get response* message MUST be sent by the repository in response to [Endpoint metadata get request message](#endpoint-metadata-get-request).
Repository MUST publish endpoint metadata get response message to the subject provided in the NATS `replyTo` field of the request.

Endpoint metadata get response message payload MUST be an Avro-encoded object with the following schema ([0019-endpoint-metadata-get-response.avsc](./0019-endpoint-metadata-get-response.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.epmmp.gen.v1",
    "name": "EndpointMetadataGetResponse",
    "type": "record",
    "doc": "Endpoint metadata get response",
    "fields": [
        {
            "name": "correlationId",
            "type": "string",
            "doc": "Message id primarily used to track message processing across services"
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
            "name": "endpointId",
            "type": "string",
            "doc": "Identifier of the endpoint, on behalf of which metadata is requested"
        },
        {
            "name": "eTag",
            "type": [
                "null",
                "string"
            ],
            "default": null,
            "doc": "Identifier for a specific version of endpoint metadata. See ETag HTTP header for more details."
        },
        {
            "name": "payload",
            "type": [
                "null",
                "bytes"
            ],
            "default": null,
            "doc": "Serialized endpoint metadata content"
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
        },
        {
            "name":"appVersionName",
            "type":[
              "null",
              "string"
            ],
            "default":null,
            "doc":"Application version name of the endpoint of which metadata is requested"
        }
    ]
}
```


## Partial endpoint metadata update

Partial endpoint metadata update request MUST partially update endpoint metadata using the JSON Patch defined in [RFC 6902](https://tools.ietf.org/html/rfc6902).

### Partial endpoint metadata update request

*Partial endpoint metadata update request* message is a [targeted message](/0003/README.md#targeted-messaging) that the client sends to the repository to partially update endpoint metadata.

The client MUST send partial endpoint metadata update request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.epmmp.ep-metadata-patch-request
```
where:
- `{repository-service-instance-name}` is the endpoint metadata repository service instance name. 

The client MUST include NATS `replyTo` field to handle the response. It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.epmmp.ep-metadata-patch-response
```

Partial endpoint metadata update request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0019-endpoint-metadata-patch-request.avsc](./0019-endpoint-metadata-patch-request.avsc)):
```json
{
    "namespace": "org.kaaproject.ipc.epmmp.gen.v1",
    "name": "EndpointMetadataPatchRequest",
    "type": "record",
    "doc": "Interservice endpoint partial metadata update request",
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
            "doc": "Identifier of the endpoint, on behalf of which metadata is requested"
        },
        {
            "name": "ifMatch",
            "type": [
                "null",
                "string"
            ],
            "default": null,
            "doc": "Makes the request conditional. See If-Match HTTP header for more details."
        },
        {
            "name": "patch",
            "type": "bytes",
            "doc": "JSON Patch payload as defined in the RFC 6902"
        }
    ]
}
```

In case the client requests to create or update a restricted EP metadata key, the server MUST return an error.


### Partial endpoint metadata update response

*Partial metadata update response* message MUST be sent by the repository in response to [Partial endpoint metadata update request message](#partial-endpoint-metadata-update-request).
Repository MUST publish partial endpoint metadata update response message to the subject provided in the NATS `replyTo` field of the request.

Partial endpoint metadata update response message payload MUST be an Avro-encoded object with the following schema ([0019-endpoint-metadata.avsc](./0019-endpoint-metadata-patch-response.avsc)):
```json
{
    "namespace": "org.kaaproject.ipc.epmmp.gen.v1",
    "name": "EndpointMetadataPatchResponse",
    "type": "record",
    "doc": "Endpoint partial metadata update response",
    "fields": [
        {
            "name": "correlationId",
            "type": "string",
            "doc": "Message id primarily used to track message processing across services"
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
            "name": "endpointId",
            "type": "string",
            "doc": "Identifier of the endpoint, on behalf of which metadata update is requested"
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

## Endpoint metadata key update

Endpoint metadata key update request MUST update endpoint metadata key with specified value.

### Endpoint metadata key update request

*Endpoint metadata key update request* message is a [targeted message](/0003/README.md#targeted-messaging) that the client sends to the repository to update endpoint metadata key.

The client MUST send endpoint metadata key update request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.epmmp.ep-metadata-key-update-request
```
where:
- `{repository-service-instance-name}` is the endpoint metadata repository service instance name.

The client MUST include NATS `replyTo` field to handle the response. It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.epmmp.ep-metadata-key-update-response
```

Endpoint metadata key update request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0019-endpoint-metadata-key-update-request.avsc](./0019-endpoint-metadata-key-update-request.avsc)):
```json
{
  "namespace": "org.kaaproject.ipc.epmmp.gen.v1",
  "name": "EndpointMetadataKeyUpdateRequest",
  "type": "record",
  "doc": "Interservice endpoint metadata key update request",
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
      "doc": "Identifier of the endpoint, on behalf of which metadata key is updated"
    },
    {
      "name": "metadataKey",
      "type": "string",
      "doc": "Metadata key to update"
    },
    {
      "name": "metadataValue",
      "type": "bytes",
      "doc": "Metadata value to update key with"
    }
  ]
}
```

In case the client requests to create or update a restricted EP metadata key, the server MUST return an error.


### Endpoint metadata key update response

*Metadata key update response* message MUST be sent by the repository in response to [endpoint metadata key update request message](#endpoint-metadata-key-update-request).
Repository MUST publish endpoint metadata key update response message to the subject provided in the NATS `replyTo` field of the request.

Endpoint metadata key update response message payload MUST be an Avro-encoded object with the following schema ([0019-endpoint-metadata-key-update-response.avsc](./0019-endpoint-metadata-key-update-response.avsc)):
```json
{
  "namespace": "org.kaaproject.ipc.epmmp.gen.v1",
  "name": "EndpointMetadataKeyUpdateResponse",
  "type": "record",
  "doc": "Interservice endpoint metadata key update response",
  "fields": [
    {
      "name": "correlationId",
      "type": "string",
      "doc": "Message id primarily used to track message processing across services"
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
      "doc": "Identifier of the endpoint, on behalf of which metadata key is updated"
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

### Endpoint metadata key batch update request

*Endpoint metadata key batch update request* message is a [targeted message](/0003/README.md#targeted-messaging) that the client sends to the repository to batch update endpoint metadata key.

The client MUST send endpoint metadata key batch update request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.epmmp.ep-metadata-key-batch-update-request
```
where:
- `{repository-service-instance-name}` is the endpoint metadata repository service instance name.

The client MUST include NATS `replyTo` field to handle the response. It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.epmmp.ep-metadata-key-update-response
```

Endpoint metadata key batch update request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0019-endpoint-metadata-key-batch-update-request.avsc](./0019-endpoint-metadata-key-batch-update-request.avsc)):
```json
{
  "namespace": "org.kaaproject.ipc.epmmp.gen.v1",
  "name": "EndpointMetadataKeyBatchUpdateRequest",
  "type": "record",
  "doc": "Interservice endpoint metadata key update request",
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
      "name": "commands",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "EndpointMetadataKeyUpdateCommand",
          "fields": [
            {
              "name": "endpointId",
              "type": "string",
              "doc": "Identifier of the endpoint, on behalf of which metadata key is updated"
            },
            {
              "name": "metadataKey",
              "type": "string",
              "doc": "Metadata key to update"
            },
            {
              "name": "metadataValue",
              "type": "bytes",
              "doc": "Metadata value to update key with"
            }
          ]
        }
      }
    }
  ]
}
```

### Endpoint metadata key batch update response

*Metadata key batch update response* message MUST be sent by the repository in response to [endpoint metadata key batch update request message](#endpoint-metadata-key-batch-update-request).
Repository MUST publish endpoint metadata key batch update response message to the subject provided in the NATS `replyTo` field of the request.

Endpoint metadata key batch update response message payload MUST be an Avro-encoded object with the following schema ([0019-endpoint-metadata-key-batch-update-response.avsc](./0019-endpoint-metadata-key-batch-update-response.avsc)):
```json
{
  "namespace": "org.kaaproject.ipc.epmmp.gen.v1",
  "name": "EndpointMetadataKeyBatchUpdateResponse",
  "type": "record",
  "doc": "Interservice endpoint metadata key batch update response",
  "fields": [
    {
      "name": "correlationId",
      "type": "string",
      "doc": "Message id primarily used to track message processing across services"
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


## Getting endpoint metadata keys

### Endpoint metadata keys get request

*Endpoint metadata keys get request* message is a [targeted message](/0003/README.md#targeted-messaging) that the client sends to the repository to retrieve endpoint metadata keys.

The client MUST send endpoint metadata keys get request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.epmmp.ep-metadata-keys-get-request
```
where:
- `{repository-service-instance-name}` is the endpoint metadata repository service instance name.

The client MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.epmmp.ep-metadata-keys-get-response
```

Endpoint metadata keys get request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0019-endpoint-metadata-keys-get-request.avsc](./0019-endpoint-metadata-keys-get-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.epmmp.gen.v1",
    "name":"EndpointMetadataKeysGetRequest",
    "type":"record",
    "doc":"Interservice endpoint metadata keys get request",
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
            "doc":"Identifier of the endpoint, on behalf of which metadata keys are requested"
        }
    ]
}
```


### Endpoint metadata keys get response

*Endpoint metadata keys get response* message MUST be sent by the repository in response to [Endpoint metadata keys get request message](#endpoint-metadata-keys-get-request).
Repository MUST publish endpoint metadata keys get response message to the subject provided in the NATS `replyTo` field of the request.

Endpoint metadata keys get response message payload MUST be an Avro-encoded object with the following schema ([0019-endpoint-metadata-keys-get-response.avsc](./0019-endpoint-metadata-keys-get-response.avsc)):

```json
{
     "namespace":"org.kaaproject.ipc.epmmp.gen.v1",
     "name":"EndpointMetadataKeysGetResponse",
     "type":"record",
     "doc":"Endpoint metadata keys get response",
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
             "doc":"Identifier of the endpoint, on behalf of which metadata keys are requested"
         },
         {
             "name":"keys",
             "type":{
               "type":"array",
               "items":"string"
             },
             "default":[],
             "doc":"List of the endpoint metadata keys"
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

## Getting endpoints application version

### Endpoints application version request

*Endpoints application version request* message is a [targeted message](/0003/README.md#targeted-messaging) that client sends to repository to retrieve endpoints application version.

The client MUST send endpoint  application version request messages using the following NATS subject:
```
kaa.v1.service.{repository-service-instance-name}.epmmp.ep-app-version-get-request
```

The client MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.replica.{client-service-replica-id}.epmmp.ep-app-version-get-response
```

Endpoints application version request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0019-endpoints-app-version-request.avsc](./0019-endpoints-app-version-request.avsc)):

```json
{
  "namespace":"org.kaaproject.ipc.epmmp.gen.v1",
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


### Endpoint filters response

*Endpoints application version* message MUST be sent by repository in response to an [endpoints application version request message](#endpoints-application-version-request).
Repository MUST publish endpoint filters response message to the subject provided in the NATS `replyTo` field of the request.

Endpoints application version response message payload MUST be an Avro-encoded object with the following schema ([0019-endpoints-app-version-response.avsc](./0019-endpoints-app-version-response.avsc)):
```json
{
  "namespace":"org.kaaproject.ipc.epmmp.gen.v1",
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


### Timeout and retry

There is no guarantee, that request and/or response will be delivered. Client SHOULD implement timeout and retry logic if required.
