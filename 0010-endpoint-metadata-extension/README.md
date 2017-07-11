---
name: Endpoint Metadata Extension
shortname: 10/EPMDX
status: draft
editor: Volodymyr Tkhir <vtkhir@kaaiot.com>
contributors: Alexey Shmalko <ashmalko@kaaiot.com>
---

<!-- toc -->

# Introduction

The Endpoint Metadata Extension (EPMDX) protocol is an endpoint-aware [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

The Endpoint Metadata Extension is intended to manage endpoint's metadata.
The metadata provides information about the endpoint and is a collection of key-value pairs.
Example endpoint's metadata keys are `name`, `description`, `location`, `vendor`, `deviceModel`, `firmwareVersion` etc.

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

The following terms and definitions are used in this RFC.

- **Metadata**: key-value data associated with a specific endpoint.

# Requirements and constraints

EPMDX protocol must support the following operations:
- Get metadata key-value pairs (all or for individual keys)
- Get a list of metadata keys
- Update metadata key-value pairs (all or partial update)
- Delete metadata fields by key names

# Design

## Metadata keys

The metadata keys MUST be case-sensitive non-empty alphanumeric strings with no embedded whitespace, i.e., MUST match the following [regular expression](https://en.wikipedia.org/wiki/Regular_expression) pattern:
```
^[a-zA-Z0-9]+$
```

No duplicate metadata keys are allowed.

This format is the most friendliest identifier format for all possible data stores and all kinds of computer systems in general.

## Metadata values
Metadata field values may be of any JSON type.

Example endpoint metadata:

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

## Full metadata update

Update endpoint metadata operation.

The extension-specific resource path is:
```
/<endpoint_token>/update
```
where `<endpoint_token>` identifies the endpoint.

The metadata payload represents the new state on metadata key-values.
For example, the endpoint's metadata before this operation looks like:

```json
{
  "name": "Device 1",
  "description": "The first sensor",
  "location":{
    "latitude":27.664827,
    "longitude":-81.515754
  }
}
```

The request payload:

```json
{
  "name": "Device 1",
  "location":{
    "latitude":27.112167,
    "longitude":-81.023434
  },
  "vendorId": 2
}
```

As a result, a `description` key is removed, value of `location` key is updated, and value for `vendorId` key is added.

Use [partial update operation](#partial-metadata-update) to avoid removing existing values.

### Full metadata update request

The request payload MUST be a UTF-8 encoded JSON object with the [JSON Schema](http://json-schema.org/) defined in  [update-metadata-request.schema.json](./schemas/update-metadata-request.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX update metadata request. The endpoint metadata object. The property names of this object are metadata keys and values of these properties are metadata values.",

  "type": "object",
  "minProperties": 1,
  "patternProperties": {
    "^[a-zA-Z0-9]+$": {}
  },

  "additionalProperties": false
}
```

Example:
```json
{
  "deviceModel":"example model",
  "name":"Sensor 1"
}
```

### Full metadata update response

The server MUST respond to the metadata update according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).

The response MUST follow the error response format as defined per 1/KP.

## Partial metadata update

Updates only endpoint metadata key-value pairs present in the request payload.
In comparison with [full metadata update operation](#full-metadata-update), this request will not remove the existing keys that are not present in the request payload.

Extension-specific resource path is:
```
/<endpoint_token>/update/keys
```
where `<endpoint_token>` identifies the endpoint.

### Partial metadata update request

The request payload MUST be a JSON-encoded object with the same [JSON schema](http://json-schema.org/) as in [full metadata update request](#full-metadata-update-request).
`metadata` JSON property specifies only key-value pairs that need to be updated.

Example:
```json
{
  "deviceModel":"example model",
  "name":"Sensor 1"
}
```

### Partial metadata update response

The server MUST respond to the metadata update according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).

The response MUST follow the error response format as defined per 1/KP.

## Get metadata

Extension-specific resource path is:
```
/<endpoint_token>/get
```
where `<endpoint_token>` identifies the endpoint.

### Get metadata request

The request payload MUST be a UTF-8 encoded JSON object with the JSON Schema defined in [get-metadata-request.schema.json](./schemas/get-metadata-request.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX get metadata request",
  "type": "object",
  "properties": {
    "keys": {
      "type": "array",
      "items": {
        "type": "string",
        "pattern": "^[a-zA-Z0-9]+$",
        "description": "The metadata key"
      },
      "uniqueItems": true,
      "description": "The metadata keys to be returned. Optional, if not specified all metadata keys will be returned"
    }
  },
  "additionalProperties": false
}
```

Examples:
- get all metadata

  ```json
  {
  }
  ```
- get only certain metadata key-value pairs

  ```json
  {
    "keys": [
      "name",
      "location"
    ]
  }
  ```

### Get metadata response

The response payload MUST be a UTF-8 encoded JSON object with the JSON Schema defined in [get-metadata-response.schema.json](./schemas/get-metadata-response.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX get metadata response. The endpoint metadata object. The property names of this object are metadata keys and values of these properties are values.",

  "type": "object",
  "patternProperties": {
    "^[a-zA-Z0-9]+$": {}
  },
  "additionalProperties": false
}
```

Examples:

- Endpoint metadata successfully retrieved

  ```json
  {
    "name":"Sensor 1",
    "OSName":"Linux",
    "OSVersion":"4.2.9"
  }
  ```

## Get metadata keys

Extension-specific resource path:
```
/<endpoint_token>/get/keys
```
where `<endpoint_token>` identifies the endpoint.

### Get metadata keys request

The client SHOULD send zero-byte empty payload. The server MUST ignore payload if any.

### Get metadata keys response

The response payload MUST be a UTF-8 encoded JSON array with the JSON Schema defined in [get-metadata-keys-response.schema.json](./schemas/get-metadata-keys-response.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX get metadata keys response. The set of endpoint metadata keys.",

  "type": "array",
  "items": {
    "type": "string",
    "pattern": "^[a-zA-Z0-9]+$",
    "description": "The endpoint metadata key"
  },
  "uniqueItems": true
}
```

Examples:

- Endpoint metadata keys successfully retrieved

  ```json
  [
    "name",
    "location",
    "deviceModel"
  ]

  ```

## Delete metadata keys

Extension-specific resource path is:
```
/<endpoint_token>/delete/keys
```
where `<endpoint_token>` identifies the endpoint.

### Delete metadata keys request

The request payload MUST be a UTF-8 encoded JSON array with the JSON Schema defined in [delete-metadata-keys-request.schema.json](./schemas/delete-metadata-keys-request.schema.json) file.

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX delete metadata keys request",

  "type": "array",
  "items": {
    "type": "string",
    "pattern": "^[a-zA-Z0-9]+$",
    "description": "The endpoint metadata key"
  },
  "minItems": 1,
  "uniqueItems": true
}
```

Example:
```json
["location", "areaId"]
```

### Delete metadata keys response

The server MUST respond to the metadata keys delete according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).

The response MUST follow the error response format as definer per 1/KP.
