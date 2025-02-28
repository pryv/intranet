# Pryv.io data store REST API

A brief documentation for implementing a custom (read-only) Pryv.io data store via a REST service.


## Overview

The service must implement two endpoints, responding to **streams** and **events** queries respectively.


## Endpoints

### `GET /{userId}/streams`

Retrieve the “root” streams for this user.

#### Parameters

- **`query.childrenDepth`**: integer (optional)
   The desired depth of the stream structure (i.e. how many levels of sub-streams must be expanded). `0` (default) will return only the root level; `-1` will return the entire streams structure.
- **`query.excludedIds`**: array of strings (optional)
   The list of stream ids to exclude from the results. The children of excluded streams will be excluded as well. Default: none.
- **`query.includeTrashed`**: boolean (optional)
   Whether to include trashed items. Default: `false`.
- *CURRENTLY UNUSED* **`options.*`**

#### Result

`HTTP 200 OK`

- **`streams`**: array of streams
  Stream objects have the [structure defined in the Pryv.io API documentation](https://api.pryv.com/reference/#stream).

#### Example

Result with query `/{userId}/streams/?childrenDepth=1`:

```json
[{
    "id": "root1",
    "name": "Root Stream 1"
    "created": 1615248868.0888026,
    "createdBy": "system",
    "modified": 1615248977.154165,
    "modifiedBy": "system"
    "children": [{
        "id": "child1",
        "name": "Children 1 of Stream 1"
        "parentId": "root1",
        "created": 1615248868.0888026,
        "createdBy": "system",
        "modified": 1615248977.154165,
        "modifiedBy": "system"
        "children": [],
        "childrenHidden": true 
    }]
}, {
    "id": "root1",
    "name": "Root Stream 1"
    "created": 1615248868.0888026,
    "createdBy": "system",
    "modified": 1615248977.154165,
    "modifiedBy": "system"
    "children": [],
    "clientData": {}
}]
```

### `GET /{userId}/streams/{id}`

Retrieve a specific stream for this user.

#### Parameters

- **`query.childrenDepth`**: integer (optional)
   The desired depth of the stream structure (i.e. how many levels of sub-streams must be expanded). `0` (default) will return only the root level; `-1` will return the entire streams structure.
- **`query.excludedIds`**: array of strings (optional)
   The list of stream ids to exclude from the results. The children of excluded streams will be excluded as well. Default: none.
- **`query.includeTrashed`**: boolean (optional)
   Whether to include trashed items. Default: `false`.
- ==CURRENTLY UNUSED== **`options.*`**

#### Result

`HTTP 200 OK`

- **`stream`**: stream
  Stream objects have the [structure defined in the Pryv.io API documentation](https://api.pryv.com/reference/#stream).

#### Example


Result with query `/{userId}/streams/child1?childrenDepth=1`:

```json
{
    "id": "child1",
    "name": "Children 1 of Stream 1"
    "parentId": "root1",
    "created": 1615248868.0888026,
    "createdBy": "system",
    "modified": 1615248977.154165,
    "modifiedBy": "system"
    "children": [],
    "childrenHidden": true 
}
```


### `GET /{userId}/events`

Retrieve events for this user.

#### Parameters

- **`query.fromTime`**: timestamp (optional)
  The start time of the timeframe to retrieve events for (i.e. `event.time` >= `query.fromTime` or `event.time` + `event.duration` >= `query.fromTime`)
- **`query.toTime`**: timestamp (optional)
  The end time of the timeframe to retrieve events for (i.e. `event.time` <= `query.toTime`)
- **`query.streams`**: array of stream condition sets (optional) – _see example below_
  The stream(s) to retrieve events in, defined as an array of one or more stream condition set(s), with the result events being those that match at least one of the condition sets (i.e. logical OR).
  - A **condition set** is an array of condition objects
    Events that match a particular condition set are those that match all of the conditions therein (i.e. logical AND). Each **condition** is an object with an `any` or `none` property:
    - **`any`**|**`none`**: array of stream ids
      To match, events must belong to any (`any` condition) / none (`none` condition) of the referenced streams.
- **`query.types`**: array of string (optional)
  If set, only events of any of the listed types will be returned.
- **`query.running`**: boolean (optional)
  If `true`, only running period events will be returned.
- **`query.state`**: `default`|`trashed`|`all` (optional)
  Indicates what items to return depending on their state. By default, only items that are not in the trash are returned; `trashed` returns only items in the trash, while `all` return all items regardless of their state.
- **`query.modifiedSince`**: timestamp (optional)
  If specified, only events modified since that time will be returned.
- **`options.sortAscending`**: boolean (optional)
  If `true`, events will be sorted from oldest to newest. Default: `false` (sort descending).
- **`options.skip`**: number (optional)
  The number of items to skip in the results.
- **`options.limit`**: number (optional)
  The number of items to return in the results.

#### Result

Note: To handle large set of events and avoid high memory consumption, we recommend to send the result with the header `Transfer-Encoding: chunked`, it will be processed accordingly and streamed throught Pryv.io. 

`HTTP 200 OK`

- **`events`**: array of events
  Event objects have the [structure defined in the Pryv.io API documentation](https://api.pryv.com/reference/#event).

#### Example

To retrieve all "important" events that are neither "private" nor "secret" in either "A", "B", "C" (all those being stream ids), the parameters should look like:

```json
{
    "query": {
        "streams": [
            [
                { any: ["important"] },
                { not: ["private", "secret"] }, 
                { any: ["A", "B", "C"] },
            ]
        ]
    }
}
```

### `GET /{userId}/events/{id}`

Retrieve a specific event for this user.

#### Parameters

None.

#### Result

`HTTP 200 OK`

- **`event`**: event
  Event objects have the [structure defined in the Pryv.io API documentation](https://api.pryv.com/reference/#event).


## Errors 

When an error occurs, the endpoints return a response with HTTP status `4xx` or `5xx` and an `error` object detailing the cause. Error objects have at least `id` and `message` properties – [as defined in the Pryv.io API documentation](https://api.pryv.com/reference/#error).

The expected errors are, as designated by their `id` values:

- **`invalid-request-structure`**: HTTP code `400`
  The request’s structure is not that expected; for example: invalid JSON syntax.
- **`unknown-resource`**: HTTP code `404`
  The resource can’t be found.
- **`unexpected-error`**: HTTP code `500`
  An unexpected error occurred on the server.


## Pryv.io client configuration

This configuration is set on Pryv.io.

- **`url`**: string
  The base URL of the data store REST API.
- **`headers`**: object (optional)
  HTTP headers to include with each call, defined as the object's keys (header names) and values (corresponding header values). Example: `{ "Authorization": "Bearer-Token" }`
- **`tls`**: object (optional)
  TLS configuration used if `url` starts with `https://`
    - **`ca`**: string (optional)
      The CA certificate(s) to trust in PEM format.
    - **`cert`**: string (optional)
      The client certificate chain(s) in PEM format.
    - **`key`**: string (optional)
      The client private key(s) in PEM format.
      

