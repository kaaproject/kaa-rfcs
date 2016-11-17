---
name: Kaa Protocol
shortname: 2/KP
status: raw
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
contributors: Alexey Gamov <agamov@cybervisiontech.com>
---

## Introduction

The document describes general guidelines, principles, requirements for **Kaa protocol**.

## Requirements and constraints

- The protocol should work over both **TCP** and **UDP** internet layers.
- The protocol should allow clients to connect to a server without using any SDKs and with minimum custom implementation.
- The protocol should allow clients to have the ability to establish either secure or non-secure connection. The non-secure connection can be used in case if client performance decreases gratefully while encrypting data.
- The protocol should work over MQTT and CoAP. Generally, it should work over any of application level transport protocols.
- The protocol should be extensible enough to be supplemented via users extensions.
- The protocol should allow clients to work through any gateways e.g. **MQTT** gateways or **MQTT-SN** gateways.


## Use cases

### Use case 1

Mass production. All devices share the same firmware, and there is no means to modify it. However, they all have distinct embedded hardware identifiers (e.g., MAC addresses).

### Use case 1

A gateway. Multiple constrained devices communicate with Kaa server by means of a gateway. Gateway is a traditional Kaa client and uses Kaa protocol to communicate with the server. On the other hand, end devices use a custom (proprietary) protocol to communicate with the gateway.


## Design

### Client vs endpoint

According to glossary, in Kaa 0.x series, client is:

_A client-side entity that implements the endpoint functionality. Kaa client typically uses Kaa endpoint SDK to communicate to Kaa server._

An endpoint is:

_An independently managed client-side entity within a Kaa deployment. Kaa represents every managed entity – device, sensor, mobile phone, etc. – as an endpoint._

Starting from Kaa 1.0, we differentiate between clients and endpoints.

### Extensions

The protocol must be extensible to allow adding more features in the future. We introduce a notion of _extensions_. Extension is a piece of functionality offered to the endpoints by the server. There are different extensions and we don’t know all of them ahead, so the protocol must be extension-agnostic and don’t assume extension payload format.

### Authentication

We separate _client authentication_ and _endpoint identification_. During the client authentication, a client and a server should verify each other: client verifies server identity  and server verifies that client is authorized to access the server. Note that multiple clients may share the same credentials, and thus server can’t identify clients—it merely verifies the client knows a shared secret.

The client authentication and _session-wide encryption_ are usually handled simultaneously by separate encapsulating protocols (TLS, DTLS, IPSec, none, etc.), so it is not described in this document. However, a possible authentication method is using MQTT CONNECT packet with name and password. (It is also possible to disable client authentication at all, but that’s insecure and not recommended.)

After client authentication is done, the client may issue commands on behalf of endpoints. To do so, it needs endpoint token. A token is sent along with each request as part of the resource path.

_Token_ is an opaque data blob that uniquely identifies an endpoint. Each endpoint has exactly one token assigned by the server. In the simplest form, a client always uses the same token, thus representing a single endpoint.

There are two possibilities for a client to get an endpoint token: it is either pre-provisioned into the application, or requested at run-time via an extension _(endpoint registration)_. Using an extension allows implementing different registration schemes as well as changing and allocating endpoint tokens dynamically.

**NOTE:** the server may require endpoint registration before using by specific client. That way we can prevent endpoint stealing to some degree.

**NOTE:** we can assign endpoint tokens per client session. That’s more secure and allows using short non-secure tokens.

On the other hand, using a predefined token is a simpler but less secure alternative. In that case, the endpoint is manually added to the server, the server allocates a unique token and provides it to the user, the user hard-codes the token in the application.

### Resource path format

Both MQTT and CoAP have some kind of _resource path_: MQTT has hierarchical topic names, and CoAP has URI-Path. It is wise to use them to differentiate requests.

We divide resource path into two parts. The first part of the path identifies a specific extension instance, and the second part is extension-specific. That allows extensions to have their own resource hierarchies. This document only describes the first part and gives general recommendation for the second.

Note that MQTT is sensitive to trailing/leading slashes: **a/b/c** is not same as **a/b/c/** or **/a/b/c**. The common path has the next form:

```
kaa/<application_token>/<extension_token>
```

All paths have a **kaa** prefix. (It’s likely to be configurable in the future.) The prefix helps routing all Kaa-related traffic though MQTT brokers, as all data can be matched with a single topic filter `kaa/#`.

`<application_token>` is a unique token that identifies application within a Kaa server instance.

`<extension_token>` is a unique token that identifies an extension instance within Kaa application.

// TODO(Alexey Shmalko): not using tokens is more user-friendly, we should discuss that.

The rest of the path is extension-specific and is described in separate documents. Note that endpoint token is not part of the first part of resource path as there are extensions that don’t require endpoint identity (e.g., endpoint registration extension).

**NOTE:** it might be good to restrict extension usage. That’s best handled by filtering clients, not endpoints (as we don’t really authenticate endpoints).


## Extension design guidelines

While extensions have all the freedom to define their own resource hierarchies, payload format, and communication template, they all need a set of rules to make them uniform.

The current solution suggest the following format:

```
<endpoint>/<format>[/<schema>]
```

### Endpoint-specific and nonspecific extensions

There are two large classes of extensions: those that are endpoint-specific and require an endpoint to operate on, and those that do not. While most extensions are endpoint-specific (data collection, profiling, configuration, etc.), there are extensions that are not (endpoint registration).

Endpoint-specific extensions should start path with endpoint token. Nonspecific extension should not.

### Payload format specifier

Extensions may use multiple payload formats. In that case, it is wise to add payload format specifier.

In example, use `/json` for JSON-formatted payload, and `/protobuf/<scheme_id>` for protocol buffers.

// TODO(Alexey Shmalko): CoAP has Content-Format option for that. It already has json and cbor, but not protobuf.

### Request/response pattern

Many extensions require request/response style communication, which is not supported by MQTT natively. That is usually overcome by introducing a separate topic name for responses.

We introduce `/status` topic for response messages. In case original payload contains “packet_id” response should be published in that topic.

### Payload fields

```
{
    <extension specific payload>,
    ["timestamp": timestamp],
    ["packet_id": packet_id]
}
```

`timestamp` field is optional. If it’s not present, server uses packet delivery time.

`packet_id` field is required for request/response operations. If the field is present, response containing proper packet_id should be published under `<request_topic>/status`.

**NOTE:** we might have used MQTT packet id, but in that case we lose ability to work via gateways as MQTT packet id is different for client and server.
