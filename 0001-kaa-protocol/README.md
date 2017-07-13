---
name: Kaa protocol
shortname: 1/KP
status: draft
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>, Andrew Kokhanovskyi <akokhanovskyi@cybervisiontech.com>
---

<!-- toc -->

## Introduction
This document describes general requirements, principles, design, and guidelines for **Kaa protocol** (KP) version 1.
KP is a standard protocol designed to connect client applications and endpoints to a Kaa server.

## Language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint**: an entity managed by Kaa platform. <!-- TODO(ashmalko): We need to define this more specifically. There are many entities managed by Kaa, most of which are not endpoints. -->
Platform users are interested in differentiating all endpoints.
Endpoints may be either physical or virtual.

- **Client**: an application that uses a single "connection" to the server.
One client may represent multiple endpoints.

- **Endpoint token**: an opaque data blob that uniquely identifies an endpoint.
Each endpoint has exactly one token assigned by the server.

- **Resource path**: a unique resource identifier included in each request.
For MQTT, that is Topic Name, for CoAP — URI-Paths.

- **Extension**: a piece of functionality offered to the client by the server.
Extension is usually represented by a separate resource.
Examples of extensions are [Data Collection]()<!--TODO-->, [Configuration Management]()<!--TODO-->, [Endpoint Metadata Synchronization]()<!--TODO-->.

## Requirements and constraints

KP requirements:

- Asynchronous messaging.
- Encrypted and unencrypted channels.
- Working over [MQTT](http://mqtt.org/) version 3.1 and newer.
- Working over [CoAP](http://coap.technology/).
- Extensibility, support for future extensions.
- Client authentication.
- Endpoint identification (but not necessarily authentication).

KP should enable client to:

- Connect to a server without using an SDK and with minimum custom implementation.
- Work through MQTT gateways.
This also includes MQTT-SN gateways.

## Use cases
KP is designed to accommodate for the following key use cases (and combinations thereof).

### UC1: individually authenticated device
Every device has an individual set of credentials that they use to authenticate with the server.
Credentials may be loaded into device at the factory or set by the device user.

### UC2: mass production device
All devices share the same firmware, and there are no per-device unique credentials that can be loaded.
However, they have distinct embedded hardware identifiers (e.g. MAC addresses).

### UC3: actor gateway
Multiple constrained devices communicate with a server through a gateway.
Gateway is a client that uses KP to communicate with the server and represent the end devices.
On the other hand, end devices use a custom (proprietary) protocol to communicate with the gateway.

### UC4: forward proxy
One or more devices communicate to a server by using KP and connecting via a forward proxy.
Forward proxies between the end device and a server may be chained.

## Design

### Extensions
Extensions are used to support various platform features offered by the server to the endpoints, such as data collection, configuration management, metadata synchronization, etc.
To enable data exchange between endpoints and Kaa server extensions, KP itself must be extensible (with *KP extensions*). <!-- TODO(ashmalko): server extensions is an implementation detail for 1/KP (protocols extensions can be easily implemented within the server core), so the reasoning is a bit vague. -->
Various extensions may require different formats for the data exchanged over KP.
Since the data formats for all possible extensions are not known upfront, KP must be agnostic to that and avoid assuming extension payload format.

There are two large classes of extensions:
- *endpoint-aware extensions* operate against a defined endpoint.
For example, Data Collection extension requires identification of the source endpoint.

- *endpoint-unaware extensions* do not require endpoint identity to function.
For example, [Endpoint Register extension]()<!--TODO--> may function before there is an established endpoint identity on the server.

The same extension may be configured more than once.
For example, there may be several instances of Data Collection extension that are set up to collect data of different nature.
*Extension instance names* are used in Kaa to distinguish *extension instances*.

### Resource path format
Both MQTT and CoAP support some kind of a *Resource Path*: MQTT has hierarchical topic names, and CoAP has URI-Path.
In KP they are used to differentiate types of requests and responses.

Resource Path is separated into two parts.
The first part is common for all extensions, and the second is extension-specific.
This allows extensions define their own resource path hierarchies.
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
Thus, the resource paths for them will start with `kp1/<appversion_name>/<extension_instance_name>/<endpoint_token>`.

The rest of the resource path is extension-specific and is described in other RFCs defining KP protocol extensions.

Examples of extension-specific Resource Paths:
```
/<endpoint_token>/json
/protobuf/<scheme_id>
/json
```

## Request/response pattern
Many extensions require request/response style communication, which is natively supported by CoAP, but not MQTT.
The following section describes request/response communication pattern for MQTT and CoAP.

Note that some requests do not require a full response but only an acknowledgement of reception.
In that case, KP over MQTT SHOULD use PUBLISH acknowledgement only (PUBACK, PUBREC, PUBREL packets), as defined per MQTT standard.

### Request direction
For CoAP, server-initiated requests require implementing a server on the client nodes.
That also implies there must be an open path from server to client, which may require UDP hole punching.

Given that, server-initiated requests are NOT RECOMMENDED.

### Request ID
CoAP message has a built-in Token which is used to match responses to requests.

MQTT PUBLISH packet includes a Packet Identifier.
However, Packet Identifiers are not stable; that means, when a PUBLISH packet is re-transmitted by MQTT broker or MQTT gateway, the Packet Identifier may be changed.
Thus, MQTT Packet Identifiers are not suitable for matching responses to requests.

KP over MQTT MUST NOT use Packet Identifiers to match responses to requests.

KP can not make assumptions about payload format, thus it cannot define the format for passing Request IDs within the message payload.
Thus, Request IDs are embedded within the topic name.

In case the last topic level in MQTT topic is a positive integer number, it MUST be interpreted as a Request ID.

For example, the request sent to `<endpoint_token>/update/42` topic is a request to `<endpoint_token>/update` Resource Path with Request ID `42`.

If Request ID is not present in the MQTT topic, the response SHOULD NOT be published.

> Note: Request ID in the topic name is MQTT-specific and is not counted toward Resource Path.

### Response topic
For CoAP, request and response semantics are carried in CoAP messages.

MQTT does not have a request/response notion, so a separate MQTT topic is introduced to differentiate responses.

Successful responses over MQTT MUST be published to the topic constructed by appending `/status` suffix to the request topic.

Error responses over MQTT MUST be published to the topic constructed by appending `/error` suffix to the request topic.

For example, request sent to `<endpoint_token>/json/42` has the following success response topic: `<endpoint_token>/json/42/status`; the error response topic is `<endpoint_token>/json/42/error`.

> Note: response type suffix is MQTT-specific and is not counted toward Resource Path.

### Error response format
Error responses published over MQTT MUST be UTF-8 encoded JSON object with the scheme defined in [error-response.schema.json](./error-response.schema.json) file:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "Kaa Protocol error response",
  "type": "object",
  "properties": {
    "statusCode": {
      "type": "integer",
      "description": "Status code based on HTTP status codes"
    },
    "reasonPhrase": {
      "type": "string",
      "description": "A human-readable string explaining the cause of an error"
    }
  },
  "required": [
    "statusCode",
    "reasonPhrase"
  ],
  "additionalProperties": false
}
```

For CoAP, error responses should leverage built-in CoAP capabilities:
- Status Code should map to CoAP Response Codes ([RFC 7252, section 5.9](https://tools.ietf.org/html/rfc7252#section-5.9)).
- Reason Phrase should be transfered as CoAP Diagnostic Payload ([RFC 7252, section 5.5.2](https://tools.ietf.org/html/rfc7252#section-5.5.2)).

Extensions MAY reuse the same format for successful responses.

### Response QoS level
The response is RECOMMENDED to have same QoS level as the corresponding request.

## Extension design guidelines
While extensions have all the freedom to define their own resource hierarchies, payload format, and communication template, they need a set of rules to make them uniform.

### Payload format specifier
Extensions may support multiple payload formats.
In that case, it is wise to add payload format specifier.

For example, use `/json` for JSON-formatted payload, and `/protobuf/<scheme_id>` for protocol buffers.

<!--TODO: CoAP has Content-Format option for that. It already has json and cbor, but not protobuf.-->


### Security
We separate *client authentication* and *endpoint identification*.
During client authentication, a client and a server should verify each other: client verifies server identity and server verifies that the client is authorized to access the server.
Note that multiple clients may share the same credentials, and thus server can't identify clients — it merely verifies the client knows a shared secret.

Client authentication and *session-wide encryption* are usually handled simultaneously by separate encapsulating protocols (TLS, DTLS, IPSec, none, etc.), so it is not described in this RFC.
However, a possible authentication method is using MQTT CONNECT packet with name and password.
It is also possible to disable client authentication at all, but that's insecure and is not recommended.

After client authentication is done, the client may issue commands on behalf of the endpoints.
To do so, the client needs an [endpoint token](#language).
A token is sent along with each request as a part of the [resource path](#language).

In the simplest form, a client always uses the same token, thus representing a single endpoint.

There are two possibilities for a client to get an endpoint token: it is either pre-provisioned into the application, or requested at run-time via an extension *(endpoint registration)*.
Using an extension allows implementing different registration schemes as well as changing and allocating endpoint tokens dynamically.

>**NOTE:**
>- The server may require endpoint registration before using by specific client.
>This is to prevent endpoint stealing to some degree.
>- Endpoint tokens can be assigned per client session.
>This provides better security and allows using short non-secure tokens.

A predefined token can also be used, which is a simpler but less secure alternative.
In that case, the endpoint is manually added to the server, the server allocates a unique token and provides it to the user, the user embeds the token in the application.

## Open questions

### Topic name aliases
It's good to keep topic names short.
Maybe we can introduce topic aliases, so user won't need to use full-length topic name.

### Status topics
Would it make sense to move `/status` suffix to some place in the middle of the MQTT path (like between the common and extension-specific parts) instead of the end?
This would make it easier for clients to subscribe to responses with a wildcard.

If it is, it should be placed right after the endpoint token.
In that case, a wildcard to subscribe to all endpoint responses is `kp1/+/+/<endpoint_token>/status/#`.

Downsides are:
- The rule seems to be more complex.
When it is "whatever the request topic is, just append /status", it is really easy to follow.

- Status topics have different directions.
(Some are client->server, some are server->client.)
That means that subscribing to all status topics isn't always a good idea.

### Endpoint migration
An endpoint might migrate between different clients.

#### UC1
Endpoint is a wearable that communicates through stationary gateways.
The endpoint thus can be seen through any of the gateways.
Also, it can be seen through two or more gateways at the same time.

#### UC2
Endpoint has channel redundancy through two (or more) different gateways (or forward proxies).

### Reporting errors
In the current KP design, it is not possible to respond with an error to a specific message.

Errors may be:
- Application version does not exist.
- Wrong extension instance name.
- Unauthorized (EP token not found).
- etc.

With MQTT, the only option seems to be responding to the `.../status` topic using the MQTT request packet ID.
That does not play well with gateways/proxies as they do not preserve packet ID.

With CoAP, the response can be direct.

### Security
Which authentication combinations should the KP implementations support?
