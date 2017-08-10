---
name: Data Collection protocol
shortname: 2/DCP
status: draft
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>
---

<!-- toc -->


# Introduction

Data Collection protocol is an endpoint-aware extension of [Kaa protocol](/0001-kaa-protocol/README.md).

It is designed to collect data from endpoints and transfer it to extension services for storage and/or processing.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Data sample**: an independent data record collected from an endpoint.
- **Batch**: a collection of data samples uploaded in a single request message.
- **Processing confirmation**: an acknowledgment designating that the server has finished processing a request.


# Requirements and constraints

- Data sample processing is asynchronous.
Processing data samples might require a significant period of time and may be performed simultaneously or in a different order than sent.

- The client may need to know if a data sample has been processed.
This is needed for the client to guarantee that the data has been processed and to free allocated resources.

  Possible solutions:
  - Use QoS levels for message delivery confirmation.
  - Use QoS levels for message processing confirmation.
    MQTT requires all PUBACK messages appear in the same order as received PUBLISH messages.
    This creates a synchronization point, which conflicts with asynchronous processing requirement.
  - Use separate response messages for processing confirmation.
    This is achieved by request/response pattern as defined per 1/KP.

- The server should be able to handle different data formats.
  Different endpoints may send data in a variety of formats. The server should know the format to parse the payload.

  Solutions:
  - Use `<format_specifier>` embedded into the resource path.

- The server should know the endpoint that generates the data.

  Solutions:
  - Make 2/DCP an endpoint-aware extension as defined per 1/KP.

- Minimize network usage.
  Internet connection may be slow, unstable, or costly, so the extension should send as little data as possible.

  Solutions:
  - Upload data in [batches](#Language).

- Device may not be able to generate timestamps.

  Solutions:
  - On the server side, use network packet arrival time as a timestamp if no timestamp is present.


# Use cases


## UC1: Device shadow

The user only wants to know the current status of the endpoint parameters.
The endpoint updates them periodically.


## UC2: Historical data collection

The user wants to store all collected data with timestamps for further processing and for visualizing historical trends.


# Design


## Batch uploads

To reduce network usage, data sample are uploaded in *batches*.
All records in a batch are processed as one record and have a single response status.


## No built-in timestamp handling

Due to the fact that different applications might need timestamps in different formats and precision, Data Collection protocol does not provide any special handling for timestamps.
There is no special field for a timestamp — it is the responsibility of higher layers to interpret any field as a timestamp.

Recommended fallback solution for cases when there is no timestamp: save server timestamp upon receiving a network message and pass it along with the parsed data, so the upper layers can use that timestamp if needed.


## Request/response

2/DCX uses client-initiated request/response pattern defined in [1/KP](/0001-kaa-protocol/#requestresponse-pattern).


### Request

The client MUST send data collection requests to the following extension-specific Resource Path:
```
/<endpoint_token>/json
```

The request payload MUST be a UTF-8 encoded JSON object with the following [JSON schema](http://json-schema.org/) ([request.schema.json](./request.schema.json))
```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "2/DCX request schema",
  "type": "array"
}
```
where each element of the array represents a single data sample.
Data samples can be of any valid JSON type.

Example:
```json
[
  { "key": "value" },
  15,
  [ "an", "array", 13 ]
]
```


### Response

A successful processing confirmation response MUST have zero-length payload.

A successful response indicates the bucket was successfully delivered and processed.
