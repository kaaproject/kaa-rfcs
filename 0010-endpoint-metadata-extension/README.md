---
name: Endpoint Metadata Extension
shortname: 10/EPMDX
status: draft
editor: Volodymyr Tkhir <vtkhir@cybervisiontech.com>
---

- [Introduction](#introduction)
- [Requirements and constraints](#requirements-and-constraints)
  - [Metadata keys](#metadata-keys)
  - [Metadata values](#metadata-values)
  - [Supported operations](#supported-operations)
- [Design](#design)
  - [Full metadata update](#full-metadata-update)
    - [Full metadata update request](#full-metadata-update-request)
    - [Full metadata update response](#full-metadata-update-response)
  - [Partial metadata update](#partial-metadata-update)
    - [Partial metadata update request](#partial-metadata-request)
    - [Partial metadata update response](#partial-metadata-response)
  - [Get metadata](#get-metadata)
    - [Get metadata request](#get-metadata-request)
    - [Get metadata response](#get-metadata-response)
  - [Get metadata keys](#get-metadata-keys)
    - [Get metadata keys request](#get-metadata-keys-request)
    - [Get metadata keys response](#get-metadata-keys-response)


## Introduction

The Endpoint Metadata Extension (EPMDX) protocol is an endpoint-aware [Kaa Protocol](/0001-kaa-protocol/README.md) extension.

It is intended to manage endpoint metadata.

## Requirements and constraints

### Metadata keys
EPMDX protocol supports only string type of metadata keys.
All UTF-8 characters are supported in metadata key names.
Metadata key must contain at least one non-whitespace character.
For example, `longitude`, `street`, `name`, `$address 2` are valid metadata keys.
No duplicate metadata keys allowed.
### Metadata values
Valid metadata value type is any JSON type:
- `boolean`
- `string`
- `number`
- `null`
- `object`
- `array`

Example endpoint metadata:

```json
{
  "name":"Sensor 1",
  "OS name":"Linux",
  "OS version":"4.2.9",
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
### Supported operations

EPMDX protocol must support the following operations:
- Get metadata key-value pairs (all or for a certain keys)
- Get a list of metadata keys
- Update metadata key-value pairs (all or partial update)
- Delete a single metadata key value pair by key name

## Design

### Full metadata update

Create or update endpoint metadata operation.
In case of update the provided metadata will override all existing metadata key-value pairs.
Use [partial update operation](#partial-metadata-update) to avoid removing an existing values.

#### Full metadata update request

Client MUST send requests with the following extension-specific [resource path](/0001-kaa-protocol/README.md#resource-path-format) part in order to receive the metadata key-value pairs:

  `<endpoint_token>/update`

  where `endpoint_token` identifies the endpoint.

The request payload MUST be a JSON-encoded object with the following [JSON schema](http://json-schema.org/) ([file](./schemas/update-metadata-request.schema.json)):

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX update metadata request",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "description": "The message identifier used to match server response to the request"
    },
    "metadata": {
      "type": "object",
      "minProperties": 1,
      "patternProperties": {
        "\\S+": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "number"
            },
            {
              "type": "boolean"
            },
            {
              "type": "null"
            },
            {
              "type": "object"
            },
            {
              "type": "array"
            }
          ]
        }
      },
      "additionalProperties": false,
      "description": "The endpoint metadata object. The property names of this object are metadata keys and values of these properties are values"
    }
  },
  "additionalProperties": false,
  "required": [
    "metadata"
  ]
}
```
Examples:
- update metadata with message identifier

```json
{
  "id":42,
  "metadata":{
    "deviceModel":"example model",
    "name":"Sensor 1"
  }
}
```
- update metadata without message identifier

```json
{
  "metadata":{
    "deviceModel":"example model",
    "name":"Sensor 1"
  }
}
```

#### Full metadata update response

The server MUST respond to the metadata get request by publishing the response message according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).
The extension-specific resource path part format is:

  `<endpoint_token>/update/status`

The response payload MUST be a JSON-encoded object with the following JSON schema ([file](./schemas/update-metadata-response.schema.json)):

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX update metadata response",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "description": "The message identifier used to match server response to the request"
    },
    "statusCode": {
      "type": "integer",
      "minimum": 1,
      "description": "Status code based on HTTP status codes"
    },
    "reasonPhrase": {
      "type": "string",
      "minLength": 1,
      "description": "A human-readable string explaining the cause of an error (if any). `OK` if request was successful"
    }
  },
  "required": [
    "statusCode",
    "reasonPhrase"
  ],
  "additionalProperties": false
}
```

  where
  - `id` MUST match that in the request

Examples:

- Endpoint metadata successfully updated

```json
{
  "id":42,
  "statusCode":200,
  "reasonPhrase":"OK"
}
```
- Endpoint metadata successfully created

```json
{
  "id":42,
  "statusCode":201,
  "reasonPhrase":"Created"
}
```
### Partial metadata update

Create or update endpoint metadata key-value pairs.
In case of update the provided metadata will override all existing metadata key-value pairs.
In comparison with [full metadata update operation](#full-metadata-update) this operation will not override all metadata key-value pairs.
It will only create or update the metadata keys specified in the request.

#### Partial metadata request

Client MUST send requests with the following extension-specific [resource path](/0001-kaa-protocol/README.md#resource-path-format) part in order to receive the metadata key-value pairs:

  `<endpoint_token>/update/keys`

  where `endpoint_token` identifies the endpoint.

The request payload MUST be a JSON-encoded object with the same [JSON schema](http://json-schema.org/) as in [full metadata update request](#full-metadata-update-request).
In the `metadata` JSON property specify only key-value pairs that need to be updated.

#### Partial metadata response

The server MUST respond to the metadata get request by publishing the response message according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).
The extension-specific resource path part format is:

  `<endpoint_token>/update/keys/status`

The response payload MUST be a JSON-encoded object with the same [JSON schema](http://json-schema.org/) as in [full metadata update request](#full-metadata-update-response).

### Get metadata

#### Get metadata request

Client MUST send requests with the following extension-specific [resource path](/0001-kaa-protocol/README.md#resource-path-format) part in order to receive the metadata key-value pairs:

  `<endpoint_token>/get`

  where `endpoint_token` identifies the endpoint.

The request payload MUST be a JSON-encoded object with the following [JSON schema](http://json-schema.org/) ([file](./schemas/get-metadata-request.schema.json)):

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX get metadata request",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "description": "The message identifier used to match server response to the request"
    },
    "keys": {
      "type": "array",
      "items": {
        "type": "string",
        "minLength": 1,
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
- get metadata without message identifier

```json
{}
```
- get metadata with message identifier

```json
{
  "id": 42
}
```
- get only certain metadata key-value pairs

```json
{
  "id": 42,
  "keys": [
    "name",
    "location"
  ]
}
```

#### Get metadata response

The server MUST respond to the metadata get request by publishing the response message according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).
The extension-specific resource path part format is:

  `<endpoint_token>/get/status`

The response payload MUST be a JSON-encoded object with the following JSON schema ([file](./schemas/get-metadata-response.schema.json)):

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX get metadata response",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "description": "The message identifier used to match server response to the request"
    },
    "metadata": {
      "type": "object",
      "patternProperties": {
        "\\S+": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "number"
            },
            {
              "type": "boolean"
            },
            {
              "type": "null"
            },
            {
              "type": "object"
            },
            {
              "type": "array"
            }
          ]
        }
      },
      "additionalProperties": false,
      "description": "The endpoint metadata object. The property names of this object are metadata keys and values of these properties are values. May not present on the error status response"
    },
    "statusCode": {
      "type": "integer",
      "minimum": 200,
      "description": "Status code based on HTTP status codes"
    },
    "reasonPhrase": {
      "type": "string",
      "minLength": 1,
      "description": "A human-readable string explaining the cause of an error (if any). `OK` if request was successful"
    }
  },
  "required": [
    "statusCode",
    "reasonPhrase"
  ],
  "additionalProperties": false
}
```

  where
  - `id` MUST match that in the request

Examples:

- Endpoint metadata successfully retrieved

```json
{
  "id":42,
  "statusCode":200,
  "reasonPhrase":"OK",
  "metadata":{
    "name":"Sensor 1",
    "OS name":"Linux",
    "OS version":"4.2.9"
  }
}
```
- No endpoint metadata found

```json
{
  "id":42,
  "statusCode":404,
  "reasonPhrase":"Endpoint metadata not found"
}
```

### Get metadata keys

#### Get metadata keys request


Client MUST send requests with the following extension-specific [resource path](/0001-kaa-protocol/README.md#resource-path-format) part in order to receive the metadata key-value pairs:

  `<endpoint_token>/get/keys`

  where `endpoint_token` identifies the endpoint.

The request payload MUST be a JSON-encoded object with the following [JSON schema](http://json-schema.org/) ([file](./schemas/get-metadata-keys-request.schema.json)):

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX get metadata keys request",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "description": "The message identifier used to match server response to the request"
    }
  },
  "additionalProperties": false
}
```
Examples:
- get metadata without message identifier

```json
{}
```
- get metadata with message identifier

```json
{
  "id": 42
}
```

#### Get metadata keys response

The server MUST respond to the metadata get request by publishing the response message according to the [request/response design pattern defined in 1/KP](/0001-kaa-protocol/README.md#requestresponse-pattern).
The extension-specific resource path part format is:

  `<endpoint_token>/get/keys`

The response payload MUST be a JSON-encoded object with the following JSON schema ([file](./schemas/get-metadata-keys-response.schema.json)):

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "10/EPMDX get metadata keys response",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "description": "The message identifier used to match server response to the request"
    },
    "keys": {
      "type": "array",
      "items": {
        "type": "string",
        "minLength": 1,
        "description": "The endpoint metadata key"
      },
      "uniqueItems": true,
      "description": "The set of endpoint metadata keys"
    },
    "statusCode": {
      "type": "integer",
      "minimum": 1,
      "description": "Status code based on HTTP status codes"
    },
    "reasonPhrase": {
      "type": "string",
      "minLength": 1,
      "description": "A human-readable string explaining the cause of an error (if any). `OK` if request was successful"
    }
  },
  "required": [
    "statusCode",
    "reasonPhrase"
  ],
  "additionalProperties": false
}

```

  where
  - `id` MUST match that in the request

Examples:

- Endpoint metadata successfully retrieved

```json
{
  "id":42,
  "statusCode":200,
  "reasonPhrase":"OK",
  "keys":[
    "name",
    "location",
    "deviceModel"
  ]
}
```
- No endpoint metadata found

```json
{
  "id":42,
  "statusCode":404,
  "reasonPhrase":"Endpoint metadata not found"
}
```

