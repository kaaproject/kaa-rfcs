---
name: Data Collection extension protocol
shortname: 2/DCX
status: draft
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>
---

- [Introduction](#introduction)
- [Language](#language)
- [Requirements and constraints](#requirements-and-constraints)
  - [Problems and possible solutions](#problems-and-possible-solutions)
- [Use cases](#use-cases)
  - [UC1: individually authenticated device](#uc1-individually-authenticated-device)
  - [UC2: mass production device](#uc2-mass-production-device)
- [Design](#Design)
  - [Batch uploads](#batch-uploads)
  - [No timestamp handling](#no-timestamp-handling)
  - [Request/response](#request-response)
  - [Formats](#formats)
    - [Schemeless JSON](#schemeless-json)
      - [Request](#request)
      - [Response](#response)

## Introduction
The Data Collection extension protocol is an endpoint-aware [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

It is designed to collect data from other services/extensions and transfer it to a server for storage and further processing.

## Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Record** — a single data point user is interested in.
- **Batch** — a collection of records uploaded within a single transaction.
- **Processing confirmation** — an acknowledgment designating that the server has finished processing a request.

## Requirements and constraints
### Problems and possible solutions

1. *Record processing is asynchronous.*
Processing might require a significant period of time and may be performed simultaneously or in different order.

2. *The client should know if a record has been delivered and/or processed.*
This is needed for client to guarantee that the data has been processed and to free allocated resources.

   Solutions:
   - Use QoS levels for message delivery confirmation.
   - Use QoS levels for message processing confirmation.

     MQTT requires all PUBACK messages appear in the same order as received PUBLISH messages.
     This creates a synchronization point, which conflicts with asynchronous record processing.
   - Use separate response messages for processing confirmation.

     For CoAP, this is supported by [Separate Response (RFC 7252, Section 5.2.2)](https://tools.ietf.org/html/rfc7252#section-5.2.2).

     For MQTT, this is achieved by publishing a response into the status topic. (See [Kaa Protocol](/0001-kaa-protocol/README.md).)

3. *Server should handle different types of data.*
Different endpoints may send data in a variety of formats.
Server should know the format to parse the payload.

   Solutions:
   - Use distinct resource paths for different formats. Embed `<format_specifier>` into the resource path.

4. *The server should know the endpoint that generates the data.*

   Solutions:
   - Embed `<endpoint_token>` into the resource path.

5. *Little network usage.*
Internet connection may be slow, unstable, or costly, so the extension should send as little data as possible.

   Solutions:
   - Upload data in *batches*.
   A batch is a number of records uploaded in one network packet.

6. *Device may be unable to generate timestamps.*

   Solutions:
   - On the server side, use network packet arrival time as a timestamp if no timestamp is present.

## Use cases

### UC1
Device shadow.
The user only wants to know the current status of the endpoint parameters.
The endpoint updates them periodically.

### UC2
Historical data collection.
The user wants to store all collected data with timestamps for further processing and for visualizing historical trends.

## Design

### Batch uploads
To reduce network usage, all records are uploaded in *batches*.
All records in a batch are processed as one record and have a single response status.

### No timestamp handling
Due to the fact that different applications might need timestamps in different formats and precision, Data Collection extension does not provide any special handling for timestamps.
There is no special field for a timestamp — it is the responsibility of higher layers to interpret any field as a timestamp.

Recommended solution: save server timestamp upon receiving a network message and pass it along the parsed data, so the upper layers can use that timestamp if needed.

### Request/response
The extension uses client-initiated request/response pattern defined in [1/KP](/0001-kaa-protocol).
A response is sent if a bucket requires processing confirmation.

For MQTT, responses MUST be published at `<request_path>/status`.
Each response MUST be published with the same QoS as the corresponding request.

### Formats
#### Schemeless JSON
##### Request
The server SHOULD support uploading arbitrary JSON records as data points to the following resource path:
```
<endpoint_token>/json
```

The payload MUST be a UTF-8 encoded JSON object with the following fields:
- `id` (optional) — ID of the batch.
Should be integer number.
- `entries` (required) — an array of data entries.
Each entry can be of any JSON type.

Example:
```json
{
  "id": 42,
  "entries": [
    { "key": "value" },
    15,
    [ "an", "array", 13 ]
  ]
}
```

If the `id` field is present, the server MUST respond with a processing confirmation response.

>**NOTE:** MQTT packet ID cannot be used.
>If used, we would be unable to work via gateways as a gateway is allowed to change MQTT packet ID.

If a batch does not specify the ID, the request degrades into a single-field JSON object.
```json
{
  "entries": [ 15 ]
}
```

In that case, the client MAY omit the `{"entries": }` part.
```json
[
  { "key": "value" },
  15,
  [ "an", "array", 13 ]
]
```

JSON schema for requests is defined in the [request.schema.json](./request.schema.json) file.

##### Response

For MQTT, processing confirmation responses are published to the following resource path.
```
<endpoint_token>/json
```

A processing confirmation response MUST be a UFT-8 encoded JSON record with the following fields:
- `id` — a copy of the `id` field from the corresponding request.
- `status` — a human-readable string explaining the cause of an error (if any).
If the processing is successful, the value MUST be `"ok"`.

JSON schema for responses is defined in the [response.schema.json](./response.schema.json) file.

---