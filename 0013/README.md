---
name: Data Sample Transmission Protocol
shortname: 13/DSTP
status: draft
editor: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


# Introduction

Data Sample Transmission Protocol (DSTP) is designed to communicate endpoint data samples from transmitter to receiver services in the Kaa platform.

DSTP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint data sample (data sample)**: block of endpoint-related data of arbitrary structure.

- **Data transmitter service (transmitter)**: service that sends endpoint data samples.

- **Data receiver service (receiver)**: service that receives endpoint data samples for storage, processing, etc.


# Design

## Endpoint data sample transmission to receivers

In order to send endpoint data samples to receivers, transmitters MUST [broadcast](/0003/README.md#broadcast-messaging) `EndpointDataSamplesEvent` messages to the following NATS subject:
```
kaa.v1.events.<transmitter-service-instance-name>.endpoint.data-collection.data-samples-received
```
where `<transmitter-service-instance-name>` is the instance name of the transmitter service.
Allows listeners to subscribe to events from a specific transmitter.

In case a transmitter expects a processing result message from receivers, it MUST set `replyTo` NATS subject in the broadcast message according to the [session affinity design](/0003/README.md#session-affinity).

The NATS message payload is an Avro object with the following schema ([0013-ep-data-samples-event.avsc](./0013-ep-data-samples-event.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.dstp.gen.v1",
    "name":"DataSamplesEvent",
    "type":"record",
    "doc": "Broadcast event with endpoint data samples to data receivers",
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
            "doc":"Application version name the data samples are sent for"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Identifier of the endpoint that originated the data samples"
        },
        {
            "name":"contentType",
            "type":"string",
            "default":"application/json",
            "doc":"Type of the data sample payload, e.g.: application/json, application/x-protobuf, etc."
        },
        {
            "name":"dataSamples",
            "type":{
                "type":"array",
                "items":"bytes"
            },
            "doc":"Array of endpoint data samples encoded according to the content type"
        }
    ]
}
```

Example:
```json
{
    "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
    "timestamp":1490262793349,
    "timeout":3600000,
    "appVersionName":"humidity-sensor-v3",
    "endpointId":"7ad263ec-3347-4c7d-af89-50c67061367a",
    "contentType":"application/json",
    "dataSamples":[
        "eyAiaHVtaWRpdHkiOiA4OCB9"
    ]
}
 ``` 


## Endpoint data response from receivers

In case a `replyTo` subject is set in the endpoint data sample message, receivers MUST respond to that subject with a `EndpointDataSampleProcessed` message according to the [session affinity design](/0003/README.md#session-affinity).

The NATS message payload is an Avro object with the following schema ([0013-ep-data-samples-processed.avsc](./0013-ep-data-samples-processed.avsc)):
```json
{
    "namespace":"org.kaaproject.ipc.dstp.gen.v1",
    "name":"DataSamplesProcessed",
    "type":"record",
    "doc": "Message from data receiver to indicate the result of processing endpoint data",
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
            "name":"receiverServiceInstanceName",
            "type":"string",
            "doc":"Service instance name of the data receiver that originated the message"
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

Example:
```json
{
    "correlationId":"07d78e95-2c4d-4899-957c-b9e5a3701fbb",
    "timestamp":1490262793349,
    "timeout":3600000,
    "receiverServiceInstanceName":"humidity-sensor-epts-1",
    "statusCode":200,
    "reasonPhrase":"OK"
}
```
