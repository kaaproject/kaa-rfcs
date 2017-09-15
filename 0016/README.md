---
name: Endpoint and Client Authentication Protocol
shortname: 16/ECAP
status: draft
editor: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


# Introduction

Endpoint and Client Authentication Protocol (ECAP) is designed for authentication services to provide [endpoints and clients](/0001/README.md#language) authentication and identification interface to other services in the Kaa platform.

ECAP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Authentication provider service (provider)**: service that exposes ECAP to other services for endpoint and client authentication.

- **Authentication consumer service (consumer)**: service that uses ECAP interface exposed by a provider to authenticate endpoints and clients.


# Design

## Endpoint identification

### Endpoint identification by token

#### Endpoint token validation request

*Endpoint token validation request* message is a [targeted message](/0003/README.md#targeted-messaging) that consumer sends to provider to make sure that the endpoint is allowed to connect and to resolve the endpoint ID.

The consumer MUST send endpoint token validation request messages using the following NATS subject:
```
kaa.v1.service.{provider-service-instance-name}.ecap.ep-token-request
```

The consumer MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.service.{consumer-service-replica-id}.ecap.ep-token-response
```

Endpoint token validation request message payload MUST be an [Avro-encoded](https://avro.apache.org/) object with the following schema ([0016-endpoint-token-validation-request.avsc](./0016-endpoint-token-validation-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.ecap.gen.v1",
    "name":"EndpointTokenValidationRequest",
    "type":"record",
    "doc":"EP token validation request message",
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
            "name":"appVersionName",
            "type":"string",
            "doc":"Endpoint's application version, for which the token validation is requested"
        },
        {
            "name":"token",
            "type":"string",
            "doc":"Endpoint token to validate"
        }
    ]
}
```


#### Endpoint token validation response

*Endpoint token validation response* message MUST be sent by provider in response to an [endpoint token validation request message](#endpoint-token-validation-request).
Provider MUST publish endpoint token validation response message to the subject provided in the NATS `replyTo` field of the request.

Endpoint token validation response message payload MUST be an Avro-encoded object with the following schema ([0016-endpoint-token-validation-response.avsc](./0016-endpoint-token-validation-response.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.ecap.gen.v1",
    "name":"EndpointTokenValidationResponse",
    "type":"record",
    "doc":"EP token validation response message",
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
            "type":[
                "string",
                "null"
            ],
            "doc":"Identifier of the endpoint resolved by the provided token. May be null in case token is not found."
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


#### Endpoint token revocation event

Provider MUST publish *endpoint token revoked event* whenever one or more previously valid endpoint tokens become unusable for endpoint identification.
On receipt of this event consumers MUST take immediate measures for terminating active endpoint communication sessions where listed tokens are used for endpoint identification.

Endpoint token revoked event is a [broadcast message](/0003/README.md#broadcast-messaging) with `endpoint` target entity type, `token` event group, and `revoked` event type.

Providers MUST use the following NATS subject format for endpoint token revoked events:
```
kaa.v1.events.{originator-service-instance-name}.endpoint.token.revoked
```

Endpoint token revoked event message payload MUST be an Avro-encoded object with the following schema ([0016-endpoint-token-revoked.avsc](./0016-endpoint-token-revoked.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.ecap.gen.v1",
    "name":"EndpointTokenRevokedEvent",
    "type":"record",
    "doc":"Endpoint token revoked event",
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
            "type":"string",
            "doc":"Identifier of the endpoint associated with revoked tokens"
        },
        {
            "name":"tokens",
            "type":{
                "type":"array",
                "items":"string"
            },
            "doc":"List of revoked endpoint tokens"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```

## Client authentication

### Client authentication by username and password

#### Client username and password validation request

*Client username and password validation request* message is a targeted message that consumer sends to provider to make sure that the client is allowed to connect and to resolve the client ID.

The consumer MUST send client username and password validation request messages using the following NATS subject:
```
kaa.v1.service.{provider-service-instance-name}.ecap.client-username-password-request
```

The consumer MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.service.{consumer-service-replica-id}.ecap.client-username-password-response
```

Client username and password validation request message payload MUST be an Avro-encoded object with the following schema ([0016-client-username-password-validation-request.avsc](./0016-client-username-password-validation-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.ecap.gen.v1",
    "name":"ClientUsernamePasswordValidationRequest",
    "type":"record",
    "doc":"Client username/password combination validation request message",
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
            "name":"username",
            "type":[
                "string",
                "null"
            ],
            "doc":"Client's username for validation"
        },
        {
            "name":"password",
            "type":[
                "string",
                "null"
            ],
            "doc":"Client's password for validation"
        }
    ]
}
```


#### Client username and password validation response

*Client username and password validation response* message MUST be sent by provider in response to a [client username and password validation request message](#client-username-and-password-validation-request).
Provider MUST publish client username and password validation response message to the subject provided in the NATS `replyTo` field of the request.

Client username and password validation response message payload MUST be an Avro-encoded object with the following schema ([0016-client-username-password-validation-response.avsc](./0016-client-username-password-validation-response.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.ecap.gen.v1",
    "name":"ClientUsernamePasswordValidationResponse",
    "type":"record",
    "doc":"Client username/password combination validation response message",
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
            "name":"clientId",
            "type":[
                "string",
                "null"
            ],
            "doc":"ID of the client resolved by provided username/password combination. May be null in case the combination is not found."
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


### Client authentication by SSL certificate

#### Client certificate validation request

*Client certificate validation request* message is a targeted message that consumer sends to provider to make sure that the client is allowed to connect and to resolve the client ID.
Provider does not perform standard certificate verification (signature, authority, expiration, etc.).
Consumer MUST perform such verification prior to sending a client certificate validation request to provider.

The consumer MUST send client certificate validation request messages using the following NATS subject:
```
kaa.v1.service.{provider-service-instance-name}.ecap.client-certificate-request
```

The consumer MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):
```
kaa.v1.service.{consumer-service-replica-id}.ecap.client-certificate-response
```

Client certificate validation request message payload MUST be an Avro-encoded object with the following schema ([0016-client-certificate-validation-request.avsc](./0016-client-certificate-validation-request.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.ecap.gen.v1",
    "name":"ClientCertificateValidationRequest",
    "type":"record",
    "doc":"Client certificate validation request message",
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
            "name":"issuer",
            "type":"string",
            "doc":"Issuer field of the client's certificate"
        },
        {
            "name":"serialNumber",
            "type":"string",
            "doc":"Serial number of the client's certificate"
        }
    ]
}
```


#### Client certificate validation response

*Client certificate validation response* message MUST be sent by provider in response to an [client certificate validation request message](#client-certificate-validation-request).
Provider MUST publish client certificate validation response message to the subject provided in the NATS `replyTo` field of the request.

Client certificate validation response message payload MUST be an Avro-encoded object with the following schema ([0016-client-certificate-validation-response.avsc](./0016-client-certificate-validation-response.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.ecap.gen.v1",
    "name":"ClientCertificateValidationResponse",
    "type":"record",
    "doc":"Client certificate validation response message",
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
            "name":"clientId",
            "type":[
                "string",
                "null"
            ],
            "doc":"ID of the client resolved by provided certificate fields. May be null in case certificate is not found."
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


### Client credential revocation event

Provider MUST publish *client credential revoked event* whenever previously valid client credential becomes unusable for client authentication.
On receipt of this event consumers MUST take immediate measures for terminating active communication sessions with the specified client.

Client credential revoked event is a [broadcast message](/0003/README.md#broadcast-messaging) with `client` target entity type, `credential` event group, and `revoked` event type.

Providers MUST use the following NATS subject format for client credential revoked events:
```
kaa.v1.events.{originator-service-instance-name}.client.credential.revoked
```

Client credential revoked event message payload MUST be an Avro-encoded object with the following schema ([0016-client-credential-revoked.avsc](./0016-client-credential-revoked.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.ecap.gen.v1",
    "name":"ClientCredentialRevokedEvent",
    "type":"record",
    "doc":"Client credential revoked event",
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
            "name":"clientId",
            "type":"string",
            "doc":"Identifier of the client associated with revoked credential"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```
