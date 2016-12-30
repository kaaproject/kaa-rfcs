---
name: Data Collection Extension
shortname: 1/DCE
status: raw
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>
---

## Introduction

The Data Collection Extension is a [Kaa Protocol](/0002-kaa-protocol/README.md) extension.

It is intended to solve the problem of transfering collected data to a server for further storing or processing.

## Requirements and constraints
### Problems and possible solutions

1. _Record processing is asynchronous._ Processing might require a significant period of time and may be performed simultaneously or in different order.

2. _The client should know if a record has been delivered and/or processed._ This is needed for client to guarantee data has been processed and free resources.

   Solutions:
   - Use QoS levels for message delivery confirmation.
   - Use QoS levels for message processing confirmation.

     MQTT requires all PUBACK messages appear in the same order as received PUBLISH messages. This creates a synchronization point, which conflicts with asynchronous record processing.
   - Use separate response messages for processing confirmation

     For CoAP, this is supported by [Separate Response (RFC 7252, Section 5.2.2)](https://tools.ietf.org/html/rfc7252#section-5.2.2).

     For MQTT, this is achieved by publishing a response into the status topic. (See [Kaa Protocol](/0002-kaa-protocol/README.md).)

3. _A server should handle different types of data._ Different endpoints may send data in a variety of formats; the server should know the format to parse the payload.

   Solutions:
   - Use distinct resource paths for different formats. Embed `<format_specifier>` into the resource path.

4. _The server should know the endpoint that generates the data._

   Solutions:
   - Embed `<endpoint_token>` into the resource path.

5. _Little network usage._ Internet connection may be slow, unstable, or costly, so the extension should send as little data as possible.

   Solutions:
   - Upload data in _batches_. A batch is a number of records, which are uploaded in one network packet.

6. _A device may have no ability to generate timestamps._

   Solutions:
   - On the server side, use network packet arrival time as a timestamp.

## Use cases

### UC1
Device shadow. The user is only interested in the current status of the endpoint parameters. The endpoint updates them periodically.

### UC2
Historical data collection. The user is interested in storing all collected data with timestamps for further processing and visualizing historical trends.

## Design

### Batch uploads
To reduce network usage, all records are uploaded in _batches_. All records in a batch are processed as one and have a single response status.

### Timestamps
Due to the fact that different application might need timestamps in different formats and precision, data collection extension does not provide any special handling for timestamps. That's responsibility of higher layers to interpret any field as a timestamp.

#### Absent timestamp handling
As there is no special field for a timestamp, that's higher level's issue how to handle absent timestamps.

The recommended solution is to save server timestamp upon network message receiving and pass it along the parsed data, so upper layers can use receiving timestamp if needed.

### Request/response
The extension uses request/response pattern. A response is sent if a bucket requires a processing confirmation.

For MQTT, responses MUST be published at `<request_path>/status`. Each response MUST be published with the same QoS as the corresponding request.

### Formats
#### Schemeless JSON
The server should support uploading arbitrary JSON records as data points at the following resource path:
```
<endpoint_token>/json
```

The payload should be a JSON-encoded object with the following fields:
- `id` (optional) - id of the batch. Should be either string or number.
- `entries` (required) - an array of data entries. Each one of the entries can be of any JSON type.

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

```json
{
  "entries": [ 15 ]
}
```

If `id` field is present, the server MUST respond with a processing confirmation response.

_Note: we might have used MQTT packet id, but in that case we lose ability to work via gateways as a gateway may change MQTT packet id._

A processing confirmation response is a JSON record with the following fields:
- `id` a copy of the `id` field from the corresponding request.
- `status` a human-readable string explaining the cause of an error (if any). In case processing was sucessful, it is `"ok"`.

## Open questions
### Batches without id
If a batch does not have an id, the request degrades into a single-field JSON object. It might be good to drop `{"entries": }` part in that case.

Example request:
```json
[
  { "key": "value" },
  15,
  [ "an", "array", 13 ]
]
```

Pros:
- simpler for some cases
- saves network usage a little

Cons:
- that's a corner case, which means more handling on the server side

## Glossary

- Record — a single data point user is interested in.
- Batch — a collection of records that are uploaded within a single transaction.
- A processing confirmation - an acknowledgment, which designates that the server has finished processing a request.
