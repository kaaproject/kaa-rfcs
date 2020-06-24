---
name: Client Authentication Protocol
shortname: 22/CAP
status: raw
editor: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


# 22/CAP: Client Authentication Protocol

## Introduction

Client Authentication Protocol (CAP) is designed for authentication services to provide [client](/0001/README.md#language) authentication interface to other services in the Kaa platform.

CAP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Authentication provider service (provider)**: service that exposes CAP to other services for endpoint and client authentication.

- **Authentication consumer service (consumer)**: service that uses CAP interface exposed by a provider to authenticate endpoints and clients.


## Design

### Basic client authentication

#### Basic authentication request

*Basic authentication request* message is a targeted message that consumer sends to provider to make sure that the client is allowed to connect with the given username and password, and to resolve the client ID.

The consumer MUST send basic authentication request messages using the following NATS subject:

```
kaa.v1.service.{provider-service-instance-name}.cap.basic-request
```

The consumer MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):

```
kaa.v1.replica.{consumer-service-replica-id}.cap.basic-response
```

Basic authentication request message payload MUST be an Avro-encoded object with the following schema ([0022-basic-authentication-request.avsc](./0022-basic-authentication-request.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientBasicAuthenticationRequest",
    "type": "record",
    "doc": "Client basic authentication request message",
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
            "doc": "Tenant ID, within the scope of which to perform authentication"
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


#### Basic authentication response

*Basic authentication response* message MUST be sent by provider in response to a [basic authentication request](#basic-authentication-request).
Provider MUST publish basic authentication response message to the subject provided in the NATS `replyTo` field of the request.

Basic authentication response message payload MUST be an Avro-encoded object with the following schema ([0022-basic-authentication-response.avsc](./0022-basic-authentication-response.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientBasicAuthenticationResponse",
    "type": "record",
    "doc": "Client basic authentication response message",
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


### Client authentication by x.509 certificate

#### Client certificate authentication request

*Client certificate authentication request* message is a targeted message that consumer sends to provider to make sure that the client is allowed to connect and to resolve the client ID.
Provider does not perform standard certificate authentication (signature, authority, expiration, etc.).
Consumer MUST perform such authentication prior to sending a client certificate authentication request to provider.

The consumer MUST send client certificate authentication request messages using the following NATS subject:

```
kaa.v1.service.{provider-service-instance-name}.cap.certificate-request
```

The consumer MUST include NATS `replyTo` field to handle the response.
It is RECOMMENDED to follow the subject format described in [3/ISM session affinity section](/0003/README.md#session-affinity):

```
kaa.v1.replica.{consumer-service-replica-id}.cap.certificate-response
```

Client certificate authentication request message payload MUST be an Avro-encoded object with the following schema ([0022-certificate-authentication-request.avsc](./0022-certificate-authentication-request.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientCertificateAuthenticationRequest",
    "type": "record",
    "doc": "Client certificate authentication request message",
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


#### Client certificate authentication response

*Client certificate authentication response* message MUST be sent by provider in response to an [client certificate authentication request message](#client-certificate-authentication-request).
Provider MUST publish client certificate authentication response message to the subject provided in the NATS `replyTo` field of the request.

Client certificate authentication response message payload MUST be an Avro-encoded object with the following schema ([0022-certificate-authentication-response.avsc](./0022-certificate-authentication-response.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.cap.gen.v1",
    "name": "ClientCertificateAuthenticationResponse",
    "type": "record",
    "doc": "Client certificate authentication response message",
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
            "type": [
                "string",
                "null"
            ],
            "doc": "ID of the tenant, which owns the credentials. May be null in case certificate is not found."
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


### Client credentials revocation event

Provider MUST publish *client credentials revoked event* whenever previously valid client credentials becomes unusable for client authentication.
On receipt of this event consumers MUST take immediate measures for terminating active communication sessions with the specified client.

Client credentials revoked event is a [broadcast message](/0003/README.md#broadcast-messaging) with `client-credentials` target entity type and `revoked` event type.
The event group is `basic` for username/password credentials, and `certificate` for x.509 certificates.

Providers MUST use the following NATS subject format for client credentials revoked events:

- `kaa.v1.events.{originator-service-instance-name}.client-credentials.basic.revoked`---for basic credentials;
- `kaa.v1.events.{originator-service-instance-name}.client-credentials.certificate.revoked`---for x.509 credentials.

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
            "doc": "ID of the tenant, which owns the revoked credentials"
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
