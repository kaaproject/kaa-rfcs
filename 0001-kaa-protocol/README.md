---
name: Kaa Protocol
shortname: 1/KP
status: draft
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>, Andrew Kokhanovskyi <akokhanovskyi@cybervisiontech.com>
---

## Introduction
The document describes the general requirements, principles, design and guidelines for the **Kaa Protocol** (KP) version 1.
KP is a standard protocol designed for connecting clients and endpoints (colloquially - devices) to a Kaa server.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## Requirements and constraints
- KP must support asynchronous messaging.
- Clients must be able to connect to a server without using an SDK and with minimum custom implementation.
- Encrypted and unencrypted channels must be supported.
- KP must be able to work over MQTT v3.1 and later.
- KP should be able to work over CoAP.
- KP must be extensible to support future extensions.
- Clients should be able to work through MQTT gateways. That also includes MQTT-SN gateways.
- KP must support client authentication.
- KP must support endpoint identification (but not necessarily authentication).

## Use cases
KP is designed to support the following key use cases (and combinations thereof).

### UC1: individually authenticated device
Every device has an individual set of credentials that they use to authenticate with the server.
The credentials may be loaded into the device at the factory or set by the device user.

### UC2: mass production device
All devices share the same firmware, and there are no per-device unique credentials that can be loaded.
However, they have distinct embedded hardware identifiers (e.g., MAC addresses).

### UC3: actor gateway
Multiple constrained devices communicate with a server through a gateway.
Gateway is a client that uses KP to communicate with the server and represent the end devices.
On the other hand, end devices use a custom (proprietary) protocol to communicate with the gateway.

### UC4: forward proxy
One or more devices communicate to a server by using KP and connecting via a forward proxy.
Forward proxies between the end device and a server may be chained.

## Design

### Client vs endpoint
We introduce two following terms.

- _Endpoint_ – an entity that is managed by a server.
Platform users are interested in differentiating all endpoints.
Endpoints may be either physical or virtual.

- _Client_ – an application that uses a single "connection" to the server.
One client may represent multiple endpoints.

### Extensions
To support various platform features we introduce a notion of _extensions_.
A Kaa extension is a piece of functionality offered to the endpoints by the server: such as data collection, configuration management, metadata synchronization, etc.
To enable data exchange between endpoints and Kaa server extensions, KP itself must be extensible (with _KP extensions_).
Various extensions may require different formats for the data exchanged on top of KP.
Since the data formats for all possible extensions are not known upfront, KP must be agnostic to that and avoid assuming extension payload format.

There are two large classes of extensions:
- _endpoint-aware extensions_ operate against a defined endpoint.
For example, data collection extension requires the source endpoint identification.

- _endpoint-unaware extensions_ do not require the endpoint identity to function.
For example, endpoint registration extension may function before there is an established endpoint identity in the server.

The same extension may be configured in a Kaa server more than once.
For example, there may be several instances of the data collection extension set up for collecting data of different nature.
_Extension instance names_ are used in Kaa to distinguish between _extension instances_.

### Resource path format
Both MQTT and CoAP support some kind of a _resource path_: MQTT has hierarchical topic names, and CoAP has URI-Path.
In KP they are used to differentiate types of requests and responses.

Resource path is separated into two parts.
The first part is common for all extensions, and the second is extension-specific.
That allows extensions define their own resource path hierarchies.
This document only describes the first part and provides requirements and recommendations for the second one.

**Note:** MQTT is sensitive to trailing/leading slashes: `a/b/c` is not the same as `a/b/c/` or `/a/b/c`.

The common resource path part has the following format:
`kp1/<appversion_name>/<extension_instance_name>`

All KP resource paths start with `kp1`, which is the reserved prefix for KP version 1.
Future versions of KP will have prefixes such as `kp2`, `kp3`, and so on.
Having a predefined prefix helps routing all KP-related traffic through MQTT brokers, as all messages can be matched with a single topic filter (`kp1/#` for KP version 1).

`<appversion_name>` is a unique name that identifies application and its version within a Kaa server.

`<extension_instance_name>` is a name that uniquely identifies an extension instance within the application.

Extension-specific resource path part for endpoint-aware extensions MUST start with the [endpoint token](#Glossary).
Thus, for them the resource paths start with:
`kp1/<appversion_name>/<extension_instance_name>/<endpoint_token>`.

The rest of the resource path is extension-specific and is described in separate documents that define KP protocol extensions.

## Extension design guidelines
While extensions have all the freedom to define their own resource hierarchies, payload format, and communication template, they all need a set of rules to make them uniform.

Examples of extension specific paths are:
```
/<endpoint_token>/json
/protobuf/<scheme_id>
/json/status
```

### Payload format specifier
Extensions may support multiple payload formats.
In that case, it is wise to add payload format specifier.

For example, use `/json` for JSON-formatted payload, and `/protobuf/<scheme_id>` for protocol buffers.

// TODO: CoAP has Content-Format option for that. It already has json and cbor, but not protobuf.

### Request/response pattern
Many extensions require request/response style communication, which is natively supported by CoAP, but not MQTT.
That is overcome by introducing a separate topic (resource path) for responses over MQTT.

Responses over MQTT MUST be published to the topic constructed by appending `/status` suffix to the request topic.
This applies to both server and client responses.

Response MUST be published with the same QoS level as the corresponding request.

### Security
We separate _client authentication_ and _endpoint identification_.
During the client authentication, a client and a server should verify each other: client verifies server identity and server verifies that the client is authorized to access the server.
Note that multiple clients may share the same credentials, and thus server can't identify clients – it merely verifies the client knows a shared secret.

The client authentication and _session-wide encryption_ are usually handled simultaneously by separate encapsulating protocols (TLS, DTLS, IPSec, none, etc.), so it is not described in this document.
However, a possible authentication method is using MQTT CONNECT packet with name and password.
(It is also possible to disable client authentication at all, but that's insecure and is not recommended.)

After client authentication is done, the client may issue commands on behalf of endpoints.
To do so, it needs an endpoint token.
A token is sent along with each request as part of the [resource path](#Glossary).

_Endpoint token_ is an opaque data blob that uniquely identifies an endpoint.
Each endpoint has exactly one token assigned by the server.
In the simplest form, a client always uses the same token, thus representing a single endpoint.

There are two possibilities for a client to get an endpoint token: it is either pre-provisioned into the application, or requested at run-time via an extension _(endpoint registration)_.
Using an extension allows implementing different registration schemes as well as changing and allocating endpoint tokens dynamically.

_Note: the server may require endpoint registration before using by specific client._
_That way we can prevent endpoint stealing to some degree._

_Note: we can assign endpoint tokens per client session._
_That's more secure and allows using short non-secure tokens._

On the other hand, using a predefined token is a simpler but less secure alternative.
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
In the current KP design it is not possible to respond with an error to a specific message.
Errors may be:
- appversion does not exit;
- wrong extension instance name;
- unauthorized (EP token not found);
- etc.

With MQTT, the only option seems to be responding to the `.../status` topic using the MQTT request packet id. That does not play well with gateways/proxies as they do not preserve packet id.

With CoAP, the response can be direct.

### Security
Which authentication combinations should the KP implementations support?

## Glossary
- _Endpoint_ – an entity that is managed by the Kaa platform.
Platform users are interested in differentiating all endpoints.
Endpoints may be either physical or virtual.

- _Client_ – an application that uses a single "connection" to the server.
One client may represent multiple endpoints.

- _Endpoint token_ is an opaque data blob that uniquely identifies an endpoint.
Each endpoint has exactly one token assigned by the server.

- _Resource path_ – a unique resource identifier included in each request.
For MQTT that is Topic Name, for CoAP: URI-Paths.

- _Extension_ – a piece of functionality offered to the client by the server.
It is usually represented by a separate resource.
Examples of extensions are data collection, configuration management, endpoint metadata synchronization.
