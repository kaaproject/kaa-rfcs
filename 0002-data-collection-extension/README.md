---
name: Data Collection protocol
shortname: 2/DCX
status: draft
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>
---

<!-- toc -->

# Introduction
Data Collection protocol is an endpoint-aware extension of [Kaa protocol](/0001-kaa-protocol/README.md).

It is designed to collect data from endpoints and transfer it to services/extensions for storage and/or processing.

# Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Record**: a single data point that user is interested in.
- **Batch**: a collection of records uploaded within a single request.
- **Processing confirmation**: an acknowledgment designating that the server has finished processing a request.

# Requirements and constraints
Data Collection protocol requirements:

- Record processing is asynchronous.
Processing might require a significant period of time and may be performed simultaneously or in different order.

- The client must know if a record has been processed.
This is needed for client to guarantee that the data has been processed and to free allocated resources.

  Possible solutions:
  - Use QoS levels for message delivery confirmation.
  - Use QoS levels for message processing confirmation.
    MQTT requires all PUBACK messages appear in the same order as received PUBLISH messages.
    This creates a synchronization point, which conflicts with asynchronous record processing requirement.
  - Use separate response messages for processing confirmation.

    For CoAP, this should be supported by [Separate Response (RFC 7252, Section 5.2.2)](https://tools.ietf.org/html/rfc7252#section-5.2.2).

    For MQTT, this is achieved by publishing a response into the status topic. See request/response pattern defined in 1/KP.

- Server should handle different types of data.
  Different endpoints may send data in a variety of formats. Server should know the format to parse the payload.

  Solutions:
  - Use `<format_specifier>` embedded into resource path.

- The server should know the endpoint that generates the data.

  Solutions:
  - Using `<endpoint_token>` embedded into resource path.
    Therefore, 2/DCX is an endpoint-aware extension as defined per 1/KP.

- Little network usage.
  Internet connection may be slow, unstable, or costly, so the extension should send as little data as possible.

  Solutions:
  - Upload data in batches.
    A batch is a number of records uploaded in one network packet.

- Device can be unable to generate timestamps.

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
To reduce network usage, all records are uploaded in *batches*.
All records in a batch are processed as one record and have a single response status.

## No built-in timestamp handling
Due to the fact that different applications might need timestamps in different formats and precision, Data Collection protocol does not provide any special handling for timestamps.
There is no special field for a timestamp — it is the responsibility of higher layers to interpret any field as a timestamp.

Recommended fallback solution for cases when there is no timestamp: save server timestamp upon receiving a network message and pass it along the parsed data, so the upper layers can use that timestamp if needed.

## Request/response
2/DCX uses client-initiated request/response pattern defined in [1/KP](/0001-kaa-protocol/#requestresponse-pattern).

## Formats
### Schemeless JSON
#### Request
JSON Schema for requests is defined in the [request.schema.json](./request.schema.json) file.

The server MUST handle requests at the following resourse path:
```
/<endpoint_token>/json
```

The payload MUST be a UTF-8 encoded JSON object with the following structure:

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "2/DCX request schema",

  "type": "array"
}
```
where each element of an array represents a single record.

Example:
```json
[
  { "key": "value" },
  15,
  [ "an", "array", 13 ]
]
```

#### Response
A processing confirmation response MUST follow the error response format as defined per 1/KP.

Succesful response means the bucket has been sucessfully delivered and processed.
