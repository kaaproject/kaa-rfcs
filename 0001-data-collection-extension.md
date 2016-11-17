---
name: Data Collection Extension
editor: Alexey Shmalko
status: raw
contributors: Alexey Gamov
---

## Introduction

This document describes data collection extension. For general statements and principles refer to the [protocol design reference]().

The feature is intended to solve the problem of delivering collected data to a server for further processing e.g. storing the data, perform analytics and so on.

## Requirements and constraints

This section describes requirements for *Data Collection Extension*, basing on the problems which appear during data supplying from endpoints to a server.

### Problems and possible solutions

1. TCP or UDP can be used as a network layer, so it's unknown whether the message sent by an endpoint has reached a server. As for UDP, it is clear that there is no delivery guarantee and this issue should be handled on the higher layer - application one. As for TCP things are not so obvious: though TCP guarantees delivery, an acknowledgment can be lost (in case if a connection has been lost), so the endpoint re-transmits the message and the server receives a duplicate.

  - As for _UDP_ case, _QoS_ can be used for delivery confirmation. As for _TCP_ case, [log batch id](#Glossary) can be used for *a processing confirmation*. *Log batch id* is the token which represents at least one [log record](#Glossary). Log records are sent via [batches](#Glossary). All log records inside the batch have the same *log batch id*.

2. A server should handle different types of data. At one time point the endpoint can send one type of data and at another time point it can send different one. The variety of data types are represented via multiple data formats (json, protobuf) and different data itself.

  - The issue can be resolved using separate address tokens for all the combinations of data formats with data set. The mean for data sets descriptions is data schemes. A data schemes set and a data formats set should be restricted by an [application version]() e.g. if we have the application v.1.0.0, it supports json data format and data schema v.1 then the application can not send data in protobuf format or use another data schema version. For this purpose another application should be created, with another application version. Possible format for address token is:

```
<endpoint>/<format>[/<schema>]
```

**NOTE:** The current solution suggests using no scheme in the address token.

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

## Glossary

- Log record — a single data point user is interested in.
- Log batch — a collection of records that are uploaded within a single transaction.
- Log batch id - a token which identifies a batch of log records.
- Application version - //TODO: Discuss what it is.
- A processing confirmation - an acknowledgment which is sent by a server, which designates that the server has received and processed a message.
