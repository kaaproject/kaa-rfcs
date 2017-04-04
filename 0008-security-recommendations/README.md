---
name: KP Security Recommendations
shortname: 8/Security
status: raw
editor: Alexey Shmalko <ashmalko@cybervisiontech.com>
---
## Introduction
This document describes security recommendations for systems using 1/KP. This is only a recommendation and is not mandatory to implement.

## Model
We start with the generalization of the system to better understand the attack surface, threats, and possible protection techniques.

The general system consists of a server, clients, and end devices. End devices are devices that produce data, and the client only serves as a proxy for them. End devices are represented as endpoints within 1/KP.

So the connection scheme looks like this:
```
server -- clients -- end devices
```

All data flows between the server and end devices via clients.

As you see, there are two links involved: server-client and client-device. These links are purely logical. (e.g., there is no physical link if the client acts as an end device and produces data on its own.)

## Goals
The goal of this document is to ensure _confidentiality_, _integrity_, and _authenticity_ of data flowing between the server and end devices.

_Confidentiality_ "is the property, that information is not made available or disclosed to unauthorized individuals, entities, or processes" (Excerpt ISO27000).

_Data integrity_ means maintaining and assuring the accuracy and completeness of data over its entire life-cycle. This means that data cannot be modified in an unauthorized or undetected manner.

_Authenticity_ is the property that ensures that the identity of a subject or resource is the identity claimed.

_Note: in general, you want to authenticate both servers, clients, and devices._

The whole system is secure when all three properties are ensured for both links.

Note also that not all properties might be required for the system at the same time, but authenticity usually requires data integrity, and often relies on confidentiality to be effective.

## Possible solutions
This section describes possible solutions for ensuring necessary properties for both links.

Note that the easiest way to ensure all properties at the same time is making the link purely virtual. That could be achieved by placing both ends of a link on the same software or hardware system.

Examples are running both client and end devices on the same hardware machine, or running both client and server on the same host and communicate over loop interface. (While that is not extremely practical in IoT solution it well may happen during development.)

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

When using a bi-directional certificate authentication, it is recommended to issue separate certificates for each client instance, to limit the effect of private key stealing. Unfortunately, that is not always possible (e.g., mass production device); in that case, it is recommended to pass some unique client identifier using MQTT username/password (and using whitelists).

### Client-Device
The Client-Device link is usually running a different protocol than 1/KP, so responsibility of ensuring confidentiality, integrity, and authenticity can be delegated to the client.

_Note: that can only be delegated when the client itself is trusted. That usually requires integrity and authenticity of Server-Client link._

If Client-Device link is running 1/KP, the Server-Client link protection techniques can be used (replacing server with client, and client with device). See Server-Client section for details.

Recommendations for ensuring required properties for different protocols is out of scope of this document.

However, in case clients cannot ensure required properties, the server can do that by considering Server-Device link, which is running over a Server-Client one.

### Server-Device

Confidentiality:

- MQTT payload encryption to server/endpoint key

Integrity:

- payload signatures with server/endpoint keys

Authenticity:

- pre-allocated endpoint tokens
- dynamic authentication via 1/KP extension (issuing tokens at runtime)

#### Tokens
If end devices cannot migrate between clients, the token scope can be limited to a client or even a single client session. In that case, the same token used by two different clients identifies two different endpoints. That technique limits the effect of token stealing.

## Notes
### Note on TLS
You should check TLS setting for both server and client. There are many attacks that make cilent and server negotiate on the least secure option.

### Note on private key protection
It is hard to protect private keys when people have hardware access to the device.
