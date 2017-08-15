---
name: Endpoint Metadata Protocol
shortname: 10/EPMP
status: draft
editor: Volodymyr Tkhir <vtkhir@kaaiot.com>
contributors: Alexey Shmalko <ashmalko@kaaiot.com>
---

<!-- toc -->


# Introduction

The Endpoint Metadata Protocol (EPMP) is an endpoint-aware [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

EPMP is intended to manage endpoint's metadata.
The endpoint metadata contains information about the endpoint and is a collection of key-value pairs.
Example endpoint's metadata keys are `name`, `description`, `location`, `vendor`, `deviceModel`, `firmwareVersion`, etc.


# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Endpoint metadata (metadata)**: key-value data associated with a specific endpoint.


# Requirements and constraints

- Client MUST be able to retrieve the list of EP metadata keys available in the server.
- Client MUST be able to retrieve complete or partial EP metadata from the server.
In case of a partial request client SHOULD specify metadata keys it is interested in receiving.
- Client MUST be able to completely or partially update the EP metadata.
- Client MUST be able to delete EP metadata fields by their key names from the server.
- Server MAY restrict client read and/or write access to some or all EP metadata keys.
In case of a client's read or write request to restricted EP metadata keys, server MUST respond with an error.


# Design

## Metadata keys

Metadata keys MUST be case-sensitive non-empty alphanumeric strings with no whitespace, i.e., MUST match the following [regular expression](https://en.wikipedia.org/wiki/Regular_expression) pattern: `^[a-zA-Z0-9]+$`.

No duplicate metadata keys are allowed.

The restrictions to the key format are dictated by the considerations of compatibility with various data stores and computer systems in general.


## Metadata values

Metadata field values MUST be of any JSON type.


## JSON object representation

Complete or partial endpoint metadata (set of key-value pairs) MAY be represented as a JSON object.
For example:

```json
{
    "name":"Sensor 1",
    "OSName":"Linux",
    "OSVersion":"4.2.9",
    "cores":2,
    "ssd":true,
    "location":{
        "latitude":27.664827,
        "longitude":-81.515754
    },
    "supportedFirmwareVersions":[
        "2.0.0",
        "2.0.1"
    ]
}
```
where `name`, `OSName`, `OSVersion`, `cores`, `ssd`, `location`, and `supportedFirmwareVersions` are metadata keys.


## Request/response

10/EPMP follows client-initiated request/response pattern defined in [1/KP](/0001-kaa-protocol/#requestresponse-pattern).


### Get metadata keys

#### Get metadata keys request

To retrieve the list of EP metadata keys, client MUST send requests to the following extension-specific resource path:
```
/<endpoint_token>/get/keys
```
where `<endpoint_token>` identifies the endpoint.

The request payload SHOULD be zero-byte empty payload.
The server MUST ignore payload if any.


#### Get metadata keys response

The server response payload MUST be a UTF-8 encoded JSON array with the the following [JSON schema](http://json-schema.org/) ([get-metadata-keys-response.schema.json](./schemas/get-metadata-keys-response.schema.json)):

```json
{
    "$schema":"http://json-schema.org/draft-04/schema#",
    "description":"10/EPMP get metadata keys response: a set of endpoint metadata keys",

    "type":"array",
    "items":{
        "type":"string",
        "pattern":"^[a-zA-Z0-9]+$",
        "description":"Endpoint metadata key name"
    },
    "uniqueItems":true
}
```

Example:

```json
[
    "name",
    "location",
    "deviceModel"
]

```

In its response, the server MUST filter out EP metadata key names, to which the client does not have neither read nor write access.
Some of the returned keys may only be readable or writable by the client.


### Get metadata

#### Get metadata request

To retrieve complete or partial EP metadata, client MUST send requests to the following extension-specific resource path:
```
/<endpoint_token>/get
```
where `<endpoint_token>` identifies the endpoint.

The request payload MUST be a UTF-8 encoded JSON object with the following JSON Schema ([get-metadata-request.schema.json](./schemas/get-metadata-request.schema.json)):

```json
{
    "$schema":"http://json-schema.org/draft-04/schema#",
    "description":"10/EPMP get metadata request",

    "type":"object",
    "properties":{
        "keys":{
            "type":"array",
            "items":{
                "type":"string",
                "pattern":"^[a-zA-Z0-9]+$",
                "description":"Endpoint metadata key name"
            },
            "uniqueItems":true,
            "description":"EP metadata keys to be returned. Optional. If not specified, all EP metadata keys will be returned."
        }
    },
    "additionalProperties":false
}
```

Examples:
- get all metadata:

  ```json
  {
  }
  ```
  In its response, the server MUST filter out EP metadata keys, to which the client does not have neither read nor write access.

- get only `name` and `location` metadata key-value pairs:

  ```json
  {
      "keys":[
          "name",
          "location"
      ]
  }
  ```

Client MAY send a get metadata request with a zero-byte empty payload.
In such event, server MUST process the request as if no metadata key names were specified in the request.

The server MUST NOT return the metadata keys that are not readable by the client.
In case the client explicitly requests a restricted EP metadata key, the server MUST return an error.


#### Get metadata response

The server response payload is a set of EP metadata key-value pairs in [JSON object representation](#JSON-object-representation).
The payload MUST be a UTF-8 encoded JSON object with the following JSON Schema ([get-metadata-response.schema.json](./schemas/get-metadata-response.schema.json)):

```json
{
    "$schema":"http://json-schema.org/draft-04/schema#",
    "description":"10/EPMP get metadata response: a set of EP metadata key-value pairs in JSON object representation",

    "type":"object",
    "patternProperties":{
        "^[a-zA-Z0-9]+$":{

        }
    },
    "additionalProperties":false
}
```

Example:
```json
{
    "name":"Sensor 1",
    "OSName":"Linux",
    "OSVersion":"4.2.9"
}
```


### Full metadata update

#### Full metadata update request

Full metadata update request overwrites the EP metadata in server with a new set of key-values received from client.

To update the EP metadata, client MUST send requests to the following extension-specific resource path:
```
/<endpoint_token>/update
```
where `<endpoint_token>` identifies the endpoint.

The request payload MUST be a UTF-8 encoded JSON object with the following JSON Schema ([update-metadata-request.schema.json](./schemas/update-metadata-request.schema.json)):

```json
{
    "$schema":"http://json-schema.org/draft-04/schema#",
    "description":"10/EPMP update metadata request: a set of EP metadata key-value pairs in JSON object representation",

    "type":"object",
    "minProperties":1,
    "patternProperties":{
        "^[a-zA-Z0-9]+$":{

        }
    },
    "additionalProperties":false
}
```

Example:
```json
{
    "deviceModel":"example model",
    "name":"Sensor 1"
}
```

The request payload is the desired new state of the EP metadata in [JSON object representation](#JSON-object-representation).
When processing a full metadata update request, the server MUST:
- create keys that were not present in the server
- update key values that were already present in the server
- remove keys that were present in the server, but absent in the request payload.
  To avoid removing the existing key-values, use [partial update operation](#partial-metadata-update).
The server MUST NOT modify or delete the metadata keys that are not writable by the client.
In case the client requests to create or update a restricted EP metadata key, the server MUST return an error.

For example, the EP metadata before the update looks like:
```json
{
    "name":"Device 1",
    "description":"The first sensor",
    "location":{
        "latitude":27.664827,
        "longitude":-81.515754
    }
}
```

The request payload:
```json
{
    "name":"Device 1",
    "location":{
        "latitude":27.112167,
        "longitude":-81.023434
    },
    "vendorId":2
}
```

As a result, value of the `location` key must be updated, value of the `vendorId` key must be added, and the `description` key must be removed in the server.


#### Full metadata update response

A successful processing confirmation response MUST have zero-length payload.


### Partial metadata update

#### Partial metadata update request

Partial metadata update request updates or creates only endpoint metadata key-value pairs present in the request payload.
Compared with the [full metadata update](#full-metadata-update), this request does not remove the existing EP metadata keys that are not present in the request payload.

To perform a partial update of the EP metadata, client MUST send requests to the following extension-specific resource path:
```
/<endpoint_token>/update/keys
```
where `<endpoint_token>` identifies the endpoint.

The request payload MUST be a JSON-encoded object with the same JSON schema (http://json-schema.org/) as in [full metadata update request](#full-metadata-update-request).

Example:
```json
{
    "deviceModel":"example model",
    "name":"Sensor 1"
}
```

The server MUST NOT modify the metadata keys that are not writable by the client.
In case the client requests to create or update a restricted EP metadata key, the server MUST return an error.


#### Partial metadata update response

A successful processing confirmation response MUST have zero-length payload.


### Delete metadata keys

To delete EP metadata keys, client MUST send requests to the following extension-specific resource path:
```
/<endpoint_token>/delete/keys
```
where `<endpoint_token>` identifies the endpoint.


#### Delete metadata keys request

The request payload MUST be a UTF-8 encoded JSON object with the following JSON Schema ([delete-metadata-keys-request.schema.json](./schemas/delete-metadata-keys-request.schema.json)):

```json
{
    "$schema":"http://json-schema.org/draft-04/schema#",
    "description":"10/EPMP delete metadata keys request",

    "type":"array",
    "items":{
        "type":"string",
        "pattern":"^[a-zA-Z0-9]+$",
        "description":"Endpoint metadata key name"
    },
    "minItems":1,
    "uniqueItems":true
}
```

Example:
```json
[
    "location",
    "areaId"
]
```

The server MUST NOT delete the metadata keys that are not writable by the client.
In case the client requests to delete a restricted EP metadata key, the server MUST return an error.


#### Delete metadata keys response

A successful processing confirmation response MUST have zero-length payload.
