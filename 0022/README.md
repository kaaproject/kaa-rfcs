---
name: Client Authentication Protocol
shortname: 22/CAP
status: draft
editor: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


# Introduction

Client Authentication Protocol (CAP) is designed for authentication services to provide [clients](/0001/README.md#language) authentication interface to other services in the Kaa platform.

CAP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Authentication provider service (provider)**: service that exposes CAP to other services for endpoint and client authentication.

- **Authentication consumer service (consumer)**: service that uses CAP interface exposed by a provider to authenticate endpoints and clients.


# Design

## Client authentication by username and password

### Client username and password verification request

*Client username and password verification request* message is a targeted message that consumer sends to provider to make sure that the client is allowed to connect and to resolve the client ID.

The consumer MUST send client username and password verification request messages using the following NATS subject:

```
kaa.v1.service.{provider-service-instance-name}.cap.basic-request
```

The consumer MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):

```
kaa.v1.replica.{consumer-service-replica-id}.cap.basic-response
```

Client username and password verification request message payload MUST be an Avro-encoded object with the following schema ([0022-basic-verification-request.avsc](./0022-basic-verification-request.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientUsernamePasswordVerificationRequest",
    "type": "record",
    "doc": "Client username/password combination verification request message",
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
            "doc": "Amount of milliseconds (since the timestamp) until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name": "tenantId",
            "type": "string",
            "doc": "Tenant ID, within the scope of which to perform verification"
        },
        {
            "name": "username",
            "type": "string",
            "doc": "Client's username for verification"
        },
        {
            "name": "password",
            "type": "string",
            "doc": "Client's password for verification"
        }
    ]
}
```


### Client username and password verification response

*Client username and password verification response* message MUST be sent by provider in response to a [client username and password verification request message](#client-username-and-password-verification-request).
Provider MUST publish client username and password verification response message to the subject provided in the NATS `replyTo` field of the request.

Client username and password verification response message payload MUST be an Avro-encoded object with the following schema ([0022-basic-verification-response.avsc](./0022-basic-verification-response.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientUsernamePasswordVerificationResponse",
    "type": "record",
    "doc": "Client username/password combination verification response message",
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
            "doc": "Amount of milliseconds (since the timestamp) until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name": "credentialsId",
            "type": [
                "string",
                "null"
            ],
            "doc": "ID of the credentials resolved by provided username/password combination. May be null in case the combination is not found."
        },
        {
            "name": "clientId",
            "type": [
                "string",
                "null"
            ],
            "doc": "ID of the client resolved by provided username/password combination. May be null in case the combination is not found or no client ID is known to provider."
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


## Client authentication by x.509 certificate

### Client certificate verification request

*Client certificate verification request* message is a targeted message that consumer sends to provider to make sure that the client is allowed to connect and to resolve the client ID.
Provider does not perform standard certificate verification (signature, authority, expiration, etc.).
Consumer MUST perform such verification prior to sending a client certificate verification request to provider.

The consumer MUST send client certificate verification request messages using the following NATS subject:

```
kaa.v1.service.{provider-service-instance-name}.cap.certificate-request
```

The consumer MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):

```
kaa.v1.replica.{consumer-service-replica-id}.cap.certificate-response
```

Client certificate verification request message payload MUST be an Avro-encoded object with the following schema ([0022-certificate-verification-request.avsc](./0022-certificate-verification-request.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientCertificateVerificationRequest",
    "type": "record",
    "doc": "Client certificate verification request message",
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
            "doc": "Amount of milliseconds (since the timestamp) until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name": "tenantId",
            "type": "string",
            "doc": "Tenant ID, within the scope of which to perform verification"
        },
        {
            "name": "issuer",
            "type": "string",
            "doc": "Issuer field of the client's certificate"
        },
        {
            "name": "serialNumber",
            "type": "string",
            "doc": "Serial number of the client's certificate, base 10 encoded"
        }
    ]
}
```


### Client certificate verification response

*Client certificate verification response* message MUST be sent by provider in response to an [client certificate verification request message](#client-certificate-verification-request).
Provider MUST publish client certificate verification response message to the subject provided in the NATS `replyTo` field of the request.

Client certificate verification response message payload MUST be an Avro-encoded object with the following schema ([0022-certificate-verification-response.avsc](./0022-certificate-verification-response.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientCertificateVerificationResponse",
    "type": "record",
    "doc": "Client certificate verification response message",
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
            "doc": "Amount of milliseconds (since the timestamp) until the message expires. Value of 0 is reserved to indicate no expiration."
        },
        {
            "name": "credentialsId",
            "type": [
                "string",
                "null"
            ],
            "doc": "ID of the credentials resolved by provided certificate fields. May be null in case certificate is not found."
        },
        {
            "name": "clientId",
            "type": [
                "string",
                "null"
            ],
            "doc": "ID of the client resolved by provided certificate fields. May be null in case certificate is not found or no client ID is known to provider."
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


## Client credentials revocation event

Provider MUST publish *client credentials revoked event* whenever previously valid client credentials becomes unusable for client authentication.
On receipt of this event consumers MUST take immediate measures for terminating active communication sessions with the specified client.

Client credentials revoked event is a [broadcast message](/0003/README.md#broadcast-messaging) with `client` target entity type, `credentials` event group, and `revoked` event type.

Providers MUST use the following NATS subject format for client credentials revoked events:

```
kaa.v1.events.{originator-service-instance-name}.client.credentials.revoked
```

Client credentials revoked event message payload MUST be an Avro-encoded object with the following schema ([0022-client-credentials-revoked.avsc](./0022-client-credentials-revoked.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientCredentialsRevokedEvent",
    "type": "record",
    "doc": "Client credentials revoked event",
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
            "name": "tenantId",
            "type": "string",
            "doc": "Tenant ID, within the scope of which credentials are revoked"
        },
        {
            "name": "credentialsId",
            "type": "string",
            "doc": "Identifier of the revoked credentials"
        },
        {
            "name": "originatorReplicaId",
            "type": "string",
            "doc": "Identifier of the service replica that generated the event"
        }
    ]
}
```
