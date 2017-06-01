---
name: Data Collection protocol
shortname: 2/DCX
status: draft
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>
---

- [Introduction](#introduction)
- [Language](#language)
- [Requirements and constraints](#requirements-and-constraints)
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
Data Collection protocol is an endpoint-aware extension of [Kaa protocol](/0001-kaa-protocol/README.md).

It is designed to collect data from endpoints and transfer it to services/extensions for storage and/or processing.

## Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Record**: a single data point user is interested in.
- **Batch**: a collection of records uploaded within a single transaction.
- **Processing confirmation**: an acknowledgment designating that the server has finished processing a request.

## Requirements and constraints

Data Collection protocol requirements:

- Asynchronous record processing.
- QoS levels to confirm message delivery and processing.
- Separate response messages for processing confirmation.
- Data type identification: distinct resource paths for different formats, `<format_specifier>` embedded into resource path.
- Endpoint identification: `<endpoint_token>` embedded into resource path.
- Minimum network usage: Upload data in batches.
A batch is a number of records uploaded in one network packet.
- Timestamp generation: on the server side, using network packet arrival time as a timestamp if no timestamp is provided by device.

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
Due to the fact that different applications might need timestamps in different formats and precision, Data Collection protocol does not provide any special handling for timestamps.
There is no special field for a timestamp — it is the responsibility of higher layers to interpret any field as a timestamp.

Recommended fallback solution for cases when there is no timestamp: save server timestamp upon receiving a network message and pass it along the parsed data, so the upper layers can use that timestamp if needed.

### Request/response
2/DCX uses client-initiated request/response pattern defined in [1/KP](/0001-kaa-protocol/#requestresponse-pattern).
A response is sent if a bucket requires processing confirmation.

### Formats
#### Schemeless JSON
##### Request

JSON Schema for requests is defined in the [request.schema.json](./request.schema.json) file.

Since 2/DCX is an endpoint-aware protocol, the server SHOULD support uploading arbitrary JSON records as data points to the following resource path:
```
<endpoint_token>/json
```

The payload MUST be a UTF-8 encoded JSON object with the following structure:

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "2/DCX request schema",

  "oneOf": [
    {
      "type": "object",
      "properties": {
        "id": {
          "type": "number",
          "multipleOf": 1.0
        },
        "entries": {
          "type": "array"
        }
      },
      "required": [ "entries" ],
      "additionalProperties": false
    },
    {
      "type": "array"
    }
  ]
}
```
where:
- `id` (optional) is the batch ID.
Should be integer number.
- `entries` (required) is an array of data entries.
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
>If used, it would not allow working via gateways as a gateway can change MQTT packet ID.

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

##### Response

JSON Schema for responses is defined in the [response.schema.json](./response.schema.json) file.

For MQTT, processing confirmation responses are published to the following resource path.
```
<endpoint_token>/json
```

A processing confirmation response MUST be a UTF-8 encoded JSON record with the following structure:

```json
{
  "$schema": "http://json-schema.org/schema#",
  "title": "2/DCX response schema",

  "type": "object",
  "properties": {
    "id": {
      "type": "number",
      "multipleOf": 1.0
    },
    "status": {
      "type": "string"
    }
  },
  "required": [ "id", "status" ],
  "additionalProperties": false
}
```
where:
- `id` is a copy of the `id` field from the corresponding request.
- `status` is a human-readable string explaining the cause of an error (if any).
If the processing is successful, the value MUST be `"ok"`.