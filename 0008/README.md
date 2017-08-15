---
name: KP Security Recommendations
shortname: 8/KPSR
status: raw
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
---

<!-- toc -->

## Introduction
This document describes security recommendations for systems using 1/KP. This is only a recommendation and is not mandatory to implement.

## Model
We start with the generalization of the _physical_ system to better understand the attack surface, threats, and possible protection techniques.

The general system consists of a server, clients, and end devices. End devices are devices that produce data, and the client only serves as a proxy for them. End devices might be represented as endpoints within 1/KP.

Note that end device is always a real device, so end device is not always an endpoint. One end device might contain multiple endpoints.

So the connection scheme looks like this:
```
server -- clients -- end devices
```

All data flows between the server and end devices via clients.

As you see, there are two links involved: server-client and client-device. These links are logical. (e.g., there is no physical link if the client acts as an end device and produces data on its own.)

### Proxy model
There could be multiple clients between the server and end devices, each acting as a proxy:
```
server -- proxy [-- proxy]* -- clients -- end devices
```

This case could be decomposed as follows.
```
server -- proxy  -- clients     -- end devices
=
server -- client -- end devices
+
          server -- clients     -- end devices
```

That is, the proxy acts as a server for the client and acts as a client for the server.

This allows analyzing this communication profile with the same set of rules.

## Goals
The goal of this document is to ensure _confidentiality_, _integrity_, and _authenticity_ of data flowing between the server and end devices.

_Confidentiality_ "is the property, that information is not made available or disclosed to unauthorized individuals, entities, or processes" (Excerpt ISO27000).

_Data integrity_ means maintaining and assuring the accuracy and completeness of data over its entire life-cycle. This means that data cannot be modified in an unauthorized or undetected manner.

_Authenticity_ is the property that ensures that the identity of a subject or resource is the identity claimed.

_Note: in general, all communicating parties should be authenticated (server, client, device)._

The whole system is secure when all three properties are ensured for both links.

Note also that not all properties might be required for the system at the same time (e.g., the system might not require confidentiality if the data could be safely disclosed), though authenticity usually requires data integrity, and often relies on confidentiality to be effective.

## Possible solutions
This section describes possible solutions for ensuring necessary properties for both links.

Most proposed solutions are not excluding (e.g., it is possible to use both certificate authentication and MQTT username/password).

Note that the easiest way to ensure all properties at the same time is making the link local. That could be achieved by placing both ends of a link on the same software or hardware system.

Examples are running both client and end devices on the same hardware machine, or running both client and server on the same host and communicate over loop interface. (While that is not extremely practical in IoT solution, it well may happen during development.)

### Server-Client
Confidentiality:

- channel encryption (provided by TLS)
- MQTT payload encryption to server/client key

Integrity:

- Message Authentication Code (MAC). e.g., HMAC (provided by TLS)
- payload signatures with server/client key

Authenticity:

- certificate authentication (provided by TLS)
- MQTT username/password (this does not authenticate the server)

_Note: when using certificates for client authentication, it is recommended to issue separate certificates for each client instance to limit the effect of private key compromising. Unfortunately, that is not always possible (e.g., mass production device); in that case, it is recommended to pass some unique client identifier using MQTT username/password and use whitelists. Note that this still not provides good security and is merely an identification technique; especially in case if unique client identifier does not have enough entropy. In any case, when a key is compromised, it must be revoked as soon as possible._

### Client-Device
The Client-Device link usually runs a different protocol than 1/KP, so responsibility of ensuring confidentiality, integrity, and authenticity can be delegated to the client.

_Note: that can only be delegated when the client itself is trusted. That usually requires integrity and authenticity of Server-Client link._

If Client-Device link is running 1/KP, the Server-Client link protection techniques can be used (replacing server with client, and client with device). See Server-Client section for details.

Recommendations for ensuring required properties for different protocols is out of scope of this document.

However, in case clients cannot ensure required properties, the server can do that by considering Server-Device link, which is running over a Server-Client one.

### Server-Device

Confidentiality:

- MQTT payload encryption to server/endpoint key. (This assumes the end device has a distinct set of keys not known to the client.)

Integrity:

- payload signatures with server/endpoint keys

Authenticity:

- pre-allocated endpoint tokens
- dynamic authentication via 1/KP extension (issuing tokens at runtime)

#### Tokens
If end devices cannot migrate between clients, the token scope can be limited to a client or even a single client session. In that case, the same token used by two different clients identifies two different endpoints. That technique limits the effect of token stealing.

## Examples
### Mobile device
A mobile device is controlled by a user, and represents both a client and an end device. It commuticates with the server directly over 1/KP over MQTT.

It that case, the client-device link is local, therefore should not be protected.

It is recommended to protect server-client link with bi-directional TLS (to provide authenticity, confidentiality, and integrity). An out-of-band registration process can be used. (e.g., a user must register on a website and provide their public key.)

In case that issuing certificates for every client is not viable, the next recommended solution is using unidirectional TLS (to provide server authenticity, confidentiality, and integrity) and MQTT username/password for user authentication.

### Mass production device
A factory produces devices and flashes them with the same firmware.

It is not possible to provision different private keys, so the recommended solution is provisioning the same key and use bi-directional TLS (provides authentication, confidentiality, and integrity) and MQTT username/password to provide unique device identifier (provides identity).

### Gateway
Multiple constrained devices communicate with a server through a gateway. Gateway is a client that uses KP over MQTT to communicate with the server and represents end devices. End devices use a custom (potentially proprietary) protocol to communicate with the gateway, and can not be reprogrammed.

As client-device link uses a custom protocol, securing it is not covered by this document.

The server-client link could be secured as either "Mobile device" or "Mass production device" example.

### Forward proxy
One or more devices communicate to a server by using KP over MQTT and connecting via a forward proxy. Forward proxies between the end device and a server may be chained.

To simplify, let's assume there are only two proxies; it is trivial to deduce solution for other number of proxies. The scheme looks as follows.
```
server -- proxy1 -- proxy2 -- client -- device
```

Whether client-device link is local or not is not relevant for this example.

If a proxy is trusted, it could authenticate adjacent parties (`proxy1` should authenticate `server` and `proxy2`; `proxy2` should authenticate `proxy1` and `client`).

For example, a proxy may trust server-provided certificate authority to verify a party identity, or issue calls to the server to authenticate MQTT username/password.

A proxy is also allowed to use completely different authentication mechanisms.

Establishing confidentiality and integrity is similar to previous examples.

In a case the proxy could not be trusted or is not able to authenticate next parties, establishing a server-device link should be considered. It is recommended to use payload encryption and payload signing with server and endpoint keys. Message authentication code is also recommended to avoid replay attacks.

Either dynamic or pre-allocated endpoint tokens are recommended.

## Notes
### Note on TLS
TLS setting must be checked for both server and client. There are many attacks that make cilent and server negotiate on the least secure option.

### Note on private key protection
It is hard to protect private keys when people have hardware access to the device.
