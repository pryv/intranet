
|         |                       |
| ------- | --------------------- |
| Authors | Ilia Kebets |
| Reviewers | Pierre-Mikael Legris |
| Date    | April 28th 2021 |
| Version | 1                 |

# Dynamic mapping (dynamic stores)

## Motivation

Expose a legacy storage through the Pryv.io API.

## Proposition

We propose to define an internal store interface that our customers will have to implement in order to access their legacy storage through the Pryv.io API. This will complete the default stores which are:

- Local storage (MongoDB)
- System streams (accessed as local)
- Audit

HF series is accessed through its own API.

## Iterations

1. Define the API for dynamic stores, by doing it for audit v2 storage
2. Define the internal API for dynamic stores, by doing it for audit storage and system streams
3. Develop an example dynamic store such as for the file system

## API

Dynamic stores are not platform-wide, but per user.

They need to implement at least [events.get](https://api.pryv.com/reference/#get-events) with eventual limitations on the method parameters. A stores API support will be defined in its config.

### Parameters

Not all parameters may be supported by all stores, for example sorting.  
API method parameters will be applied to each store. Which means that limit=10 will return the 10 items from each available store.

#### Sorting & limit

not available on multiple stores?

#### Stream queries

[Stream query](https://api.pryv.com/reference/#streams-query) blocks cannot mix stores.

Example: `{ any: ['.file-system:/users/bob/', 'diary']}` will return an error

[**ExpandStream**](https://github.com/pryv/service-core/blob/1.7.0-rc01/components/api-server/src/methods/events.js#L222) might be passed to the store. [TBC with](https://github.com/pryv/docs-pryv/blob/master/pryv.io/stores/notes-perki.md#event-association-to-stores--eventsget).

### Event.streamId

An event's streamId defines what store it belongs to through its [prefix](#store-prefix).

An event can't be part of multiple stores. Except for system streams and local storage, for legacy reasons.

#### Store prefix

As we wish to prevent collisions between IDs of different stores, we will prefix them with the name of the store. The prefix must contain characters that are forbidden in usual streamIds to not collide with existing streamIds. Forbidden characters are those that are escaped by [slug](https://www.npmjs.com/package/slug), most special characters are such as: `.`, `#`, `$`, `/`.

Examples:

- .audit:

There is a risk of collision with customers [system streams](https://api.pryv.com/customer-resources/system-streams/), therefore, we might use another character than `.`.

###### prefix end

Do we require another character to indicate the end of the prefix such as `:`, which is currently escaped? Or do we simply end it with a `-`, forbidding the use of dashes in store prefixes?

**End with another sign:**  
pros:  
- readability  
- as there can be limitations of API operations to streams of different, the dev can prevent such actions without trying the API.  
cons:  
- extra sign management in doc  

**End with a dash:**  
pros:  
- no need of extra sign  
- end-user does not need to see that they are addressing different storages (Perki's idea, I'm not convinced.)  
cons:  
- can't use dashes  

##### Local-specific

Local storage streamIds wont be prefix with `.local:` to ensure retro-compatibility.

##### Audit-specific

In audit stream prefixes, what part is the store prefix, and what defines the id type?

Accesses:

- store: `.audit:`
- type: `access-`
- accessId: `c1234`
- full: `.audit:access-c1234`

Actions:

- store: `.audit:`
- type: `action-`
- action: `events.get`
- full: `.audit:action-events.get

### Event.id

They will need to be prefixed in the same way as [streamIds](#store-prefix).

### Streams

Depending on the store, streams structure discovery can be costly. In this case, returned streams have a `hiddenChildren: true` parameter.

Each store will have a root stream which will contain the store parameters.

### Undefined properties

For some stores, properties such as `modifiedAt` can be unavailable. We propose to set them to an **absurd** value in the same type to **not break** type dependent services.

Dates: 

- 06:15 November 5, 1955 (back to the future reference)

## Deliverables

## Discussion

- [undefined properties](#undefined-properties): should they be close to normal values in order to not distort visual tools? I think not. They should be obviously wrong to notify the UX designers ASAP that they should handle these values.

## Test cases



## Out of scope


