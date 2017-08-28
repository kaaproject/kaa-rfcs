---
name: Kaa Protocol
shortname: 1/KP
status: draft
editor: Alexey Shmalko <ashmalko@kaaiot.io>
contributors: Alexey Gamov <agamov@kaaiot.io>, Andrew Kokhanovskyi <ak@kaaiot.io>
---

<!-- toc -->


## Introduction

This document describes general requirements, principles, design, and guidelines for Kaa Protocol (KP) version 1.
KP is a standard protocol designed to connect client applications and endpoints to a Kaa server.


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint**: a primary entity managed by the Kaa platform that represents either a physical or a virtual device (thing).
Platform users are interested in differentiating between endpoints.

- **Client**: an application that represents one or multiple endpoints to Kaa server.

- **Endpoint token**: an opaque data blob that uniquely identifies an endpoint.
Each token is assigned to exactly one endpoint.

- **Resource path**: a unique resource identifier included in each request.
For MQTT, that is Topic Name, for CoAP — URI-Paths.

- **Extension**: a coherent set of platform functionality offered to clients/endpoints by Kaa server.
An extension implements a function-specific communication pattern (protocol) and is usually represented by one or more separate KP resources.
Examples of extension protocols are [Data Collection Protocol](/0002/README.md), [Configuration Management Protocol](/0007/README.md), [Endpoint Metadata Protocol](0010/README.md), etc.


## Requirements and constraints

- Asynchronous messaging.
- Encrypted and unencrypted channels.
- Binding to [MQTT](http://mqtt.org/) version 3.1 and newer.
Compatibility with MQTT/MQTT-SN gateways.
- Binding to [CoAP](http://coap.technology/).
- Extensibility, support for future extensions.
- Client authentication.
- Endpoint identification (but not necessarily authentication).

KP should enable connecting clients to a server without using an SDK and with minimum custom implementation.


## Use cases

KP is designed to accommodate for the following key use cases and combinations thereof.


### UC1: Individually authenticated device

Every device has an individual set of credentials that it uses to authenticate with the server.
Credentials may be loaded into device at the factory or set by the device user.


### UC2: Mass production device

All devices share the same firmware, and there are no per-device unique credentials that can be loaded.
However, they have distinct embedded hardware identifiers (e.g. MAC addresses).


### UC3: Actor gateway

Multiple constrained devices communicate with a server through a gateway.
Gateway is a client that uses KP to communicate with the server and represent the end devices.
On the other hand, end devices use a custom (proprietary) protocol to communicate with the gateway.


### UC4: Forward proxy

One or more devices communicate to a server by using KP and connecting through a forward proxy.
Forward proxies between the end device and a server may be chained.


## Design

### Extensions

Extensions are used to support various platform features offered by the server to the clients/endpoints, such as data collection, configuration management, metadata synchronization, etc.
In order to allow implementing new extension-specific communication patterns (protocols), KP itself must be extensible.
Various extensions may require different formats for the data exchanged over KP.
Since the data formats for all possible extensions are not known upfront, KP must be agnostic to that and avoid assuming extension payload format.

There are two large classes of extensions:
- *Endpoint-aware extensions* operate against a defined endpoint.
For example, Data Collection extension requires identification of the source endpoint.

- *Endpoint-unaware extensions* do not require endpoint identity to function.
For example, [Endpoint Registration Extension]()<!--TODO--> may function before there is an established endpoint identity on the server.

The same extension may be configured on a server more than once.
For example, there may be several instances of Data Collection Extension that are set up to collect data of different nature.
*Extension instance names* are used in Kaa to distinguish *extension instances*.


### Resource path format

Both MQTT and CoAP support some kind of a *Resource Path*: MQTT has hierarchical topic names and CoAP has URI-Path.
In KP they are used to differentiate types of requests and responses.

Resource Path is separated into two parts.
The first part is common for all extensions, and the second is extension-specific.
This allows extensions to define their own resource path hierarchies.
This RFC only describes the first part and provides requirements and recommendations for the second one.

>**NOTE:** MQTT is sensitive to trailing/leading slashes: `a/b/c` is not the same as `a/b/c/` or `/a/b/c`.

The common Resource Path part has the following format:
```
kp1/<appversion_name>/<extension_instance_name>
```

All KP resource paths MUST start with `kp1`, which is the reserved prefix for KP version 1.
Future versions of KP will have prefixes such as `kp2`, `kp3`, and so on.
Having a predefined prefix helps routing all KP-related traffic through MQTT brokers, as all messages can be matched with a single topic filter (`kp1/#` for KP version 1).

`<appversion_name>` is a unique name that identifies application and its version within a server.

`<extension_instance_name>` is a name that uniquely identifies an extension instance within an application.

Extension-specific resource path part for endpoint-aware extensions MUST start with the [endpoint token](#language).
Thus, the resource path for them starts with `kp1/<appversion_name>/<extension_instance_name>/<endpoint_token>`.

The rest of the resource path is extension-specific and is described in other RFCs that define KP protocol extensions.

Examples of extension-specific Resource Paths:
```
/<endpoint_token>/json
/protobuf/<scheme_id>
/json
```


## Request/response pattern

Many extensions require request/response style communication, which is natively supported by CoAP, but not MQTT.
The following section describes request/response communication pattern for MQTT and CoAP.

Note that some requests do not require a full response but only a receipt acknowledgement.
In that case, KP over MQTT SHOULD use PUBLISH acknowledgement only (PUBACK, PUBREC, PUBREL packets), as defined in the MQTT standard.


### Request direction

For CoAP, server-initiated requests require implementing a server on client nodes.
That also implies there must be an open path from server to client, which may require UDP hole punching.

Given that, server-initiated requests are NOT RECOMMENDED.


### Request ID

CoAP message has a built-in Token used to match responses to requests.

MQTT PUBLISH packet includes a Packet Identifier.
However, Packet Identifiers are not stable.
This means, when a PUBLISH packet is re-transmitted by MQTT broker or MQTT gateway, the Packet Identifier may be changed.
Thus, MQTT Packet Identifiers are not suitable for matching responses to requests.

KP MUST NOT use Packet Identifiers over MQTT to match responses to requests.

KP cannot make assumptions about payload format, thus it cannot define the format for passing Request IDs within message payload.
Thus, Request IDs are embedded within MQTT topic name.

In case the last topic level in an MQTT topic is a positive integer number, it MUST be interpreted as a Request ID.
Request ID in the topic name is MQTT-specific and is not counted toward Resource Path.
For example, a request sent to `<endpoint_token>/update/42` topic is a request to `<endpoint_token>/update` Resource Path with Request ID `42`.

If Request ID is not present in MQTT topic, response SHOULD NOT be published.


### Response topic

For CoAP, request and response semantics are carried in CoAP messages.

MQTT does not have a request/response notion, so a separate MQTT topic is introduced to differentiate responses.

Successful responses over MQTT MUST be published to the topic constructed by appending `/status` suffix to the request topic.

Error responses over MQTT MUST be published to the topic constructed by appending `/error` suffix to the request topic.

For example, request sent to `<endpoint_token>/json/42` has the following success response topic: `<endpoint_token>/json/42/status`; the error response topic is `<endpoint_token>/json/42/error`.

>**NOTE:** Response type suffix is MQTT-specific and is not counted toward Resource Path.


### Error response format

Error responses published over MQTT MUST be UTF-8 encoded JSON object with the following [JSON schema](http://json-schema.org/) ([error-response.schema.json](./error-response.schema.json)):

```json
{
    "$schema":"http://json-schema.org/draft-04/schema#",
    "description":"Kaa Protocol error response",

    "type":"object",
    "properties":{
        "statusCode":{
            "type":"integer",
            "description":"HTTP status code of the request processing"
        },
        "reasonPhrase":{
            "type":"string",
            "description":"Human-readable status reason phrase"
        }
    },
    "required":[
        "statusCode",
        "reasonPhrase"
    ],
    "additionalProperties":false
}
```

For CoAP, error responses SHOULD leverage built-in CoAP capabilities:
- Status Code SHOULD map to CoAP Response Codes ([RFC 7252, section 5.9](https://tools.ietf.org/html/rfc7252#section-5.9)).
- Reason Phrase SHOULD be transferred as CoAP Diagnostic Payload ([RFC 7252, section 5.5.2](https://tools.ietf.org/html/rfc7252#section-5.5.2)).

Extensions MAY reuse the same format for successful responses.


### Response QoS level

It is RECOMMENDED that responses are published with the same QoS level as the corresponding request.


## Extension design guidelines

While extensions have all the freedom to define their own resource hierarchies, payload format, and communication template, they need a set of rules to make them uniform.


### Payload format specifier

Extensions may support multiple payload formats.
In such case, it is RECOMMENDED to add a payload format specifier.

For example, use `/json` for JSON-formatted payload, and `/protobuf/<scheme_id>` for protocol buffers.

<!--TODO: CoAP has Content-Format option for that. It already has json and cbor, but not protobuf.-->


### Security

We separate *client authentication* and *endpoint identification*.
During client authentication, client and server should verify each other: client verifies server identity and server verifies that client is authorized to access server.
Note that multiple clients MAY share the same credentials, in which case server can not identify clients: it merely verifies that client knows a shared secret.

Client authentication and *session-wide encryption* are usually handled simultaneously by separate encapsulating protocols (TLS, DTLS, IPSec, none, etc.), and are therefore not described in this RFC.
However, a possible authentication method would use an MQTT CONNECT packet with name and password.
You can also completely disable client authentication, but that's insecure and is not recommended.

Upon successful client authentication, the client may issue commands on behalf of the endpoints.
To do so, the client needs an [endpoint token](#language).
A token is sent along with each request as a part of the Resource Path.

In the simplest form, client always uses the same token, thus representing a single endpoint.
In case of a gateway client, multiple EP tokens may be used to represent different endpoints behind the gateway.

There are two possibilities for client to get an endpoint token: it is either pre-provisioned into the client-side application or firmware, or requested at run-time through an extension *(endpoint registration, endpoint authentication)*.
Using an extension allows implementing different registration schemes as well as changing and allocating endpoint tokens dynamically.

>**NOTE:**
>- Server may require endpoint registration before permitting EP token use by a client.
>This is to prevent endpoint stealing to some degree.
>- Endpoint tokens MAY be assigned per client session.
>This provides better security and allows using short non-secure tokens.

For more recommendations on KP security, refer to [8/KPSR](/0008/README.md).


## Open questions

### Topic name aliases

It's good to keep topic names short.
Maybe we can introduce topic aliases, so users won't need to use full-length topic names.


### Endpoints roaming

Endpoints might roam between different clients.

#### UC1
An endpoint is a wearable that communicates through stationary gateways.
The endpoint thus can be seen through any of the gateways.
Also, it can be seen through two or more gateways at the same time.

#### UC2
Endpoint has channel redundancy through two (or more) different gateways (or forward proxies).
