---
name: Time Series Transmission Protocol
shortname: 14/TSTP
status: draft
editor: Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


# Introduction

Time Series Transmission Protocol (TSTP) is designed to communicate endpoint time series data from transmitter to receiver services in the Kaa platform.

TSTP complies with the [Inter-Service Messaging](/0003/README.md) guidelines.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Data point**: data structure consisting of a timestamp, a value, and optional key-value tags.
Data point value and tags value can be of arbitrary primitive or composite data type.

- **Endpoint time series (time series)**: uniquely named series of endpoint-related data points.

- **Time series data transmitter service (transmitter)**: service that sends endpoint time series data.

- **Time series data receiver service (receiver)**: service that receives endpoint time series data for storage, processing, etc.


# Design

## Time series data transmission to receivers

In order to send endpoint data points to receivers, transmitters MUST [broadcast](/0003/README.md#broadcast-messaging) endpoint time series events to the following NATS subjects:
```
kaa.v1.events.<transmitter-service-instance-name>.endpoint.data-collection.data-points-received.<time-series-name>
```

where:
- `<transmitter-service-instance-name>` is the instance name of the transmitter service.
This allows listeners to subscribe to events from a specific transmitter.
- `<time-series-name>` is the name of the time series the transmitted data points belong to.

The NATS message payload is an Avro object with the following schema ([0014-ep-time-series-event.avsc](./0014-ep-time-series-event.avsc)):

```json
{
    "namespace":"org.kaaproject.ipc.tstp.gen.v1",
    "name":"TimeSeriesEvent",
    "type":"record",
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
            "doc":"Application version name the data is sent for"
        },
        {
            "name":"endpointId",
            "type":"string",
            "doc":"Identifier of the endpoint that data points belong to"
        },
        {
            "name":"timeSeriesName",
            "type":"string",
            "doc":"Name of the time series that data points belong to"
        },
        {
            "name":"dataPoints",
            "doc":"Array of endpoint time series data points",
            "type":{
                "type":"array",
                "items":{
                    "namespace":"org.kaaproject.ipc.tstp.gen.v1",
                    "name":"DataPoint",
                    "type":"record",
                    "fields":[
                        {
                            "name":"timestamp",
                            "type":"long",
                            "doc":"Data point UNIX timestamp in milliseconds"
                        },
                        {
                            "name":"contentType",
                            "type":[
                                "string",
                                "null"
                            ],
                            "default":"null",
                            "doc":"Type of the value in case of a composite type"
                        },
                        {
                            "name":"value",
                            "type":[
                                "boolean",
                                "int",
                                "long",
                                "float",
                                "double",
                                "string",
                                "bytes",
                                "null"
                            ],
                            "doc":"Data point value, can be any primitive or composite type. In case of a composite type, value should be encoded into string or bytes."
                        },
                        {
                            "name":"tags",
                            "doc":"Map of data point tags with tag names represented as map keys",
                            "type":{
                                "type":"map",
                                "values":{
                                    "namespace":"org.kaaproject.ipc.tstp.gen.v1",
                                    "name":"DataPointTagValue",
                                    "type":"record",
                                    "fields":[
                                        {
                                            "name":"contentType",
                                            "type":[
                                                "string",
                                                "null"
                                            ],
                                            "default":"null",
                                            "doc":"Type of the value in case of a composite type"
                                        },
                                        {
                                            "name":"value",
                                            "type":[
                                                "boolean",
                                                "int",
                                                "long",
                                                "float",
                                                "double",
                                                "string",
                                                "bytes",
                                                "null"
                                            ],
                                            "doc":"Data point tag value, can be any primitive or composite type. In case of a composite type, value should be encoded into string or bytes."
                                        }
                                    ]
                                }
                            }
                        }
                    ]
                }
            }
        }
    ]
}
```


## No receipt acknowledgement

Receivers MUST NOT acknowledge the receipt of time series events.


# Open questions

## Receipt acknowledgement

Is it worth acknowledging time series events?
In the current version of the protocol there is no delivery guarantee whatsoever.
This might pose a problem for receivers that rely on uninterrupted events delivery.
