---
name: Automation Invocation Protocol
shortname: 23/AIP
status: raw
editor: Andrew Pasika <apasika@kaaiot.io>
---

<!-- toc -->

# 23/AIP: Automation Invocation Protocol

## Introduction

Automation Invocation Protocol (AIP) is designed for invocation of various automation types supported by Kaa services.

AIP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Automation trigger service**: service that triggers automation events.

- **Automation executor service**: service that listens to and executes automation events.


## Design

### Automation invocation

In order to run automation on an automation executor service, the automation trigger service MUST publish `AutomationInvocation` message to the following NATS subject:

```
kaa.v1.events.<automation-trigger-service-instance-name>.service.automation.<automation-type>
```

where:

- `<automation-trigger-service-instance-name>` is the instance name of the automation trigger service.
Allows automation executor service to subscribe to events from a specific trigger service

- `<automation-type>` is any automation type that the automation executor service supports

<!-- TODO: Add section for reliable automation invocation -->

The NATS message payload is an Avro object with the following schema ([0023-automation-invocation.avsc](0023-automation-invocation.avsc)):

```json
{
    "namespace": "org.kaaproject.ipc.aip.gen.v1",
    "name": "AutomationInvocation",
    "type": "record",
    "doc": "Automation invocation event",
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
            "doc": "Amount of milliseconds since the timestamp until the message expires. Value of 0 is reserved to indicate no expiration"
        },
        {
            "name": "tenantId",
            "type": "string",
            "doc": "Tenant ID"
        },
        {
            "name":"payload",
            "type":[
              "null",
              "bytes"
            ],
            "default":null,
            "doc":"Automation payload"
        }
    ]
}
```
