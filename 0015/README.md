---
name: Endpoint Metadata Events
shortname: 15/EME
status: draft
editor: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


# Introduction

Endpoint Metadata Events is a method for Kaa services to communicate metadata updates that occur to endpoints.
This RFC reuses the Endpoint Lifecycle and Connectivity Events design documented in [9/ELCE](/0009/README.md), including the [broadcast messaging style](/0003/README.md#broadcast-messaging) and the [NATS subject format](/0009/README.md#nats-subject-format), and defines a new event group and event type for metadata events.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).


# Design

## Endpoint metadata updated event

Originator MUST publish an endpoint metadata updated event on any changes to endpoint metadata.
The `{event-group}` is `metadata`.
The `{event-type}` is `updated`.

Originators MUST publish endpoint events to the following NATS subjects:

```
kaa.v1.events.{originator-service-instance-name}.endpoint.metadata.updated
```
where `{originator-service-instance-name}` - name of the originator service instance.
It allows listeners to subscribe to events from a specific originator.

The NATS message payload is an Avro object with the following schema ([ep-metadata-updated.avsc](./ep-metadata-updated.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.metadata",
    "name":"MetadataUpdatedEvent",
    "type":"record",
    "doc":"Endpoint metadata updated event message",
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
            "name":"appVersionName",
            "type":"string",
            "doc":"Application version name the endpoint registered with"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Identifier of the endpoint that registered with the server"
        },
        {
            "name":"removed",
            "type":{
                "type":"map",
                "values":[
                    "null",
                    "string"
                ]
            },
            "doc":"Map of removed metadata key names to their former values. Value null is used to represent a null JSON."
        },
        {
            "name":"added",
            "type":{
                "type":"map",
                "values":[
                    "null",
                    "string"
                ]
            },
            "doc":"Map of added metadata key names to their values. Value null is used to represent a null JSON."
        },
        {
            "name":"updated",
            "type":{
                "type":"map",
                "values":{
                    "type":"record",
                    "namespace":"org.kaaproject.ipc.event.gen.v1.endpoint.metadata",
                    "name":"MetadataKeyUpdatedDto",
                    "fields":[
                        {
                            "name":"newValue",
                            "type":[
                                "null",
                                "string"
                            ],
                            "doc":"New JSON value of the updated metadata key. Value null is used to represent a null JSON."
                        },
                        {
                            "name":"oldValue",
                            "type":[
                                "null",
                                "string"
                            ],
                            "doc":"Former JSON value of the updated metadata key. Value null is used to represent a null JSON."
                        }
                    ]
                }
            },
            "doc":"Map of updated metadata key names to their old and new values"
        },
        {
            "name":"originatorReplicaId",
            "type":"string",
            "doc":"Identifier of the service replica that generated the event"
        }
    ]
}
```
