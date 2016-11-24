---
name: Data Collection Extension
shortname: 1/DCE
status: raw
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>
---

// TODO(Alexey Shmalko): remove all terms related to logging. (i.e., rename "log records", "log buckets", etc.)

## Introduction

This document describes data collection extension. For general statements and principles refer to the [protocol design reference](/0002-kaa-protocol/README.md).

The feature is intended to solve the problem of delivering collected data to a server for further processing (e.g. storing the data, perform analytics).

## Requirements and constraints
### Problems and possible solutions

1. TCP or UDP can be used as a network layer, so it's unknown if the message sent by an endpoint has reached a server. As for UDP, it is clear that there is no delivery guarantee and this issue should be handled on the higher layer - application one. As for TCP, things are not so obvious: though TCP guarantees delivery, an acknowledgment can be lost (in case a connection has been lost), so the endpoint re-transmits the message and the server receives a duplicate.

  - As for UDP case, QoS can be used for delivery confirmation. As for TCP case, [log batch id](#Glossary) can be used for *a processing confirmation*. *Log batch id* is the token which represents at least one [log record](#Glossary). Log records are sent via [batches](#Glossary). All log records inside the batch have the same *log batch id*.

2. A server should handle different types of data. At one time point the endpoint can send one type of data and at another time point it can send different one. The variety of data types are represented via multiple data formats (JSON, Protocol Buffers) and different data itself.

  - The issue can be resolved using separate address tokens for all the combinations of data formats with data set. The mean for data sets descriptions is data schemes. A data schemes set and a data formats set should be restricted by an [application version]() e.g. if we have the application v.1.0.0, it supports JSON data format and data schema v.1 then the application can not send data in protobuf format or use another data schema version. For this purpose another application should be created, with another application version. Possible format for address token is:

    ```
    <endpoint>/<format>[/<schema>]
    ```

3. Sometimes data from different endpoint should be handled the different way and a server has no notion of which endpoint sends data to it.

  - The solution is simple: we can use `<endpoint>` part in the address token to uniquely identify an endpoint.

4. Internet connection can be slow or unstable or really expensive to waste the traffic.

  - There is a need to send as little data as possible. Because of the fact that a logs amount can be sufficient, even in the case with stable connection, adding some excessive meta-data increases overhead in a significant degree. So, possible solutions are:

    - Using *batches*. In this case meta-data is added to batches instead of log records and log records are uploaded batch-by-batch, so we save a lot of internet traffic.
    - Using *delta compression*.

5. Devices can have no ability to generate timestamps.

  - Timestamps can be appended on a server. Trade-off here is that timestamps can be quite inaccurate.

6. A current server may go down.

  - A client should have ability to migrate to another server without stopping the client. //TODO: Design an implementation.

7. A client (gateway) may go down.

  - If another client (gateway) is accessible, there is to be the ability for an endpoint to work through the accessible client (gateway). //TODO: Design an implementation.

8. An endpoint can have no persistent storage.

  - The necessary metadata, such as *batch id* should be stored on the server.

## Use cases

### Use case 1
Device shadow. The user is only interested in the current status of the endpoint parameters. The endpoint updates them periodically.

### Use case 2
Historical data collection. The user is interested in storing all collected data with timestamps for further processing and visualizing historical trends.

## Design

### Batch uploads
To reduce network usage, all records are uploaded in _batches_. All records in a batch are processed as one and have a single response status.

### Timestamps
Due to the fact different application might need timestamps in different formats and precision, data collection extension does not provide any special handling for timestamps. That's responsibility of higher layers to interpret any field as a timestamp.

#### Absent timestamp handling
As there is no special field for a timestamp, that's higher level's issue how to handle absent timestamps.

The recommended solution is to save server timestamp upon network message receiving and pass it along the parsed data, so upper layers can use receiving timestamp if needed.

### Formats
#### Schemeless JSON
Data collection extension should support uploading arbitrary JSON records as data points at the following resource path:
```
<endpoint_token>/json
```

The payload should be a JSON-encoded object with the following fields:
- `id` (optional) - id of the batch. Should be either string or number.
- `entries` (required) - an array of data entries. Entry can be of any type.

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
  "entries": [ 0 ]
}
```

If `id` field is present, the server will post a processing confirmation response at the following resource path:
```
<endpoint_token>/json/status
```

Processing confirmation response is a JSON record with the following fields:
- `id` a copy of the `id` field from the corresponding request.
- `status` a human-readable string explaining the cause of an error (if any). In case processing was sucessful, it is `"ok"`.

## Open questions
### Batches without id
If batch does not have an id, the request degrades into a single-field JSON object. It might be good to drop `{"entries": }` part in that case.

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

- Log record — a single data point user is interested in.
- Log batch — a collection of records that are uploaded within a single transaction.
- Log batch id - a token which identifies a batch of log records.
- Application version - //TODO: Discuss what it is.
- A processing confirmation - an acknowledgment which is sent by a server, which designates that the server has received and processed a message.
