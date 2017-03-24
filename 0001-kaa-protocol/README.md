---
name: Kaa Protocol
shortname: 1/KP
status: draft
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>
---

## Introduction
The document describes the general requirements, principles, design and guidelines for the **Kaa Protocol** (KP).
KP is a standard protocol designed for connecting clients and endpoints (colloquially - devices) to a Kaa server.

## Requirements and constraints
- Clients must be able to connect to Kaa server without using an SDK and with minimum custom implementation.
- Encrypted and unencrypted channels must be supported.
- KP should work over MQTT and CoAP.
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
All devices share the same firmware, and there are no unique credentials that can be loaded of set per device.
However, they have distinct embedded hardware identifiers (e.g., MAC addresses).

### UC3: actor gateway
Multiple constrained devices communicate with Kaa server through a gateway.
Gateway is a Kaa client that uses KP to communicate with the server and represent the end devices.
On the other hand, end devices use a custom (proprietary) protocol to communicate with the gateway.

### UC4: forward proxy
One or more devices communicate to the Kaa server by using KP and connecting via a forward proxy.
Forward proxies between the end device and the Kaa server may be chained.

## Design

### Client vs endpoint
We introduce two following terms.

- _Endpoint_ -- an entity that is managed by the Kaa platform.
Platform users are interested in differentiating all endpoints.
Endpoints may be either physical or virtual.

- _Client_ -- an application that uses a single "connection" to the server.
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
`kaa/<appversion_name>/<extension_instance_name>`

All KP resource paths start with `kaa`.
This helps routing all KP-related traffic though MQTT brokers, as all messages can be matched with a single topic filter: `kaa/#`.

`<appversion_name>` is a unique name that identifies application and its version within a Kaa solution.

`<extension_instance_name>` is a name that uniquely identifies an extension instance within the application.

Extension-specific resource path part for endpoint-aware extensions **must** start with the endpoint token.
Thus, for them the resource paths start with:
`kaa/<appversion_name>/<extension_instance_name>/<endpoint_token>`

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
Many extensions require request/response style communication, which is not supported by MQTT natively.
That is usually overcome by introducing a separate topic name for responses.

We introduce `/status` topic for response messages.
In case original payload contains `packet_id`, response should be published in that topic.

**Note:** this only applies to client-originating requests.

### Security
We separate _client authentication_ and _endpoint identification_.
During the client authentication, a client and a server should verify each other: client verifies server identity and server verifies that the client is authorized to access the server.
Note that multiple clients may share the same credentials, and thus server can't identify clients -- it merely verifies the client knows a shared secret.

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
_That’s more secure and allows using short non-secure tokens._

On the other hand, using a predefined token is a simpler but less secure alternative.
In that case, the endpoint is manually added to the server, the server allocates a unique token and provides it to the user, the user embeds the token in the application.

## Open questions

### Topic name aliases
It's good to keep topic names short.
Maybe we can introduce topic aliases, so user won't need to use full-length topic name.

### Endpoint migration
An endpoint might migrate between different clients.
How possible this use case is?

### KP versioning
Need to explicitly define how the future versions of KP will be differentiated.

### Reporting errors
In the current KP design it is not possible to respond with an error to a specific message.
Errors may be:
- appversion does not exit;
- wrong extension instance name;
- unauthorized (EP token not found);
- etc.

### Security
- Which authentication combinations should the KP implementations support?
- Is it possible to send unencrypted, yet signed data?

## Glossary
- _Endpoint_ -- an entity that is managed by the Kaa platform.
Platform users are interested in differentiating all endpoints.
Endpoints may be either physical or virtual.

- _Client_ -- an application that uses a single "connection" to the server.
One client may represent multiple endpoints.

- _Resource path_ -- a unique resource identifier included in each request.
For MQTT that is Topic Name, for CoAP: URI-Paths.

- _Extension_ -- a piece of functionality offered to the client by the server.
It is usually represented by a separate resource.
Examples of extensions are data collection, configuration management, endpoint metadata synchronization.
