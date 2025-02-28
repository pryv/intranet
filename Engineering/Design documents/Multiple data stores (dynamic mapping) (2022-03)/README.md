|           |                                    |
| --------- | ---------------------------------- |
| Authors   | Pierre-Mikaël Legris, Simon Goumaz |
| Reviewers |                                    |
| Date      | 2021, 2022                         |
| Version   | ?                                  |

# Pryv support for multiple data stores (a.k.a. dynamic mapping)

## Goals

(Not restating the high-level goals already mentioned in the [public annoncement](https://www.pryv.com/2022/01/19/pryv-personal-data-mapping-enables-automatic-integration-with-existing-warehouses-bridging-the-gap-for-privacy-compliance-and-managing-data-subject-requests-on-personal-data-access-and-proce/))

- Implementing a data store must be easy for customers, i.e. as little work and cognitive load as possible.
- The data store API should ideally close to the Pryv.io API, to make it easy to grasp for devs
- The whole thing must be consistent with – or at least not take us further from – the design envisioned for v2 of Pryv.io


## Implementation

### Overview

- The scope is **events** and **streams** data (attachments, user accounts, accesses, everything else remains as-is).
- `DataStore` interface that all data stores (both internal and customers') must implement. To be published as npm module.
- `Mall` (==name TBC==) component, acting as the intermediary between API methods and `DataStore` implementations: delegating requests & handling results to/from stores, possibly translating parameters format, etc.
- Which store a given piece of data belongs to is encoded in its identifier; format: `:{store id}:{in-store object id}`
  - Stores must not care about "identifier internals": the mall strips the `:{store id}:` bit from items' ids before they are handed to their store; the store id only remains in references (to allow cross-store references – see following point)
  - ==Is there a need to also strip the store id from references within the store?== There might be a case with stores enforcing stuff like SQL's foreign keys… but then referencing anything outside the store would break…)
- Events can reference streams (and other events, supposing we have relationships) across multiple stores, but streams hierarchies must remain in one store (no multi-store stream hierarchies mess)
- Event attachments storage remain separate ==Q: don't we need to also support alternate stores for attachments?==
- Current events and streams storage (the default) is accessed via `LocalDataStore` implementation of `DataStore`.
- Audit data is accessed via `AuditDataStore` implementation of DataStore (events only, read only; internal writing of audit data remains as-is).

### Data store API

- Basic structure: each implementation exposes a singleton implementing `DataStore`, with `events` and `streams` properties exposing the data methods
- Handling different level of support: each store must explicitly declare its capabilities via a property or method (one reason for ruling out "implicit" discovery is the need to support versioning, which can't be easily discovered by calling a method – cf. below)
- `DataStore` should expose only the bare minimum to cover initial customer use cases. Our own internal implementations include the whole thing (such as versioning).
- Data structure:
  - ==`duration` and/or :point_right: `endTime` property for events?== (question kept open until we are confident enough with a solution for events queries, see below)
  - Stores that support versioning must implement `events.headId` (that should initially only be our internal stores, cf. comment above)
- Methods:
  - Event queries (get): ==WIP… How to specify and handle event queries? Any way to ease the work of data store implementations so that they don't have to interpret/translate the whole thing according to their structure? ⇒ 1st step: keep querying as-is, possibly adding support for a proper query language in a second step==
  - Update is actually "replace", unlike the "diff-only" updates in the Pryv.io API
- Transactions: ==see related question for mall below==

### Mall

- Basic structure mirroring `DataStore`'s: singleton exposing `events` and `streams` properties
- ==Name might need a change for clarity… `DataMall`? maybe best to just name it `[User][Data]Storage` once fully implemented, and to move it inside the `storage` component==
- ==Q: how to handle transactions within and across stores? knowing that 1) some store implementations won't support them and 2) even if all did support them, we wouldn't be able to do a "transaction of transactions" unless stores also supported an "undo" action (a probable headache)==
- Aim to keep the "converters" pattern originally defined in `BaseStorage` rather than ad-hoc equivalents

### Pryv.io API

- Generally:
  - Item ids that don't specify the store (i.e. no `:{store id}:` prefix) are assumed to point to the default (local) store
  - Keep the API itself unchanged
- Discovering stores/streams in different stores:
  - ==Current state [SG to review – search code for `hiddenChildren` or similar]:== When no parent id is provided, `streams.get`'s response includes "virtual" root streams (with no children) for each non-default streams store (Q: for backwards compatibility we must keep returning all local root streams on the same level – could that be inconsistent?)
  - ==Other options would probably involve changing the API (while staying backward-compatible)==
    - With a `storeId` param? How to discover stores themselves then?
    - Should we actually move toward the multiple "ontologies" proposal for v2, each streams store being one?
- `events.get`: Query is multiplexed to all stores on the back-end; the sets of results are merged and trimmed into the response
- `events|streams.create`:
  - ==How to specify the store to send the new item to?== (Not a problem when the caller is setting the event/stream id, which must then be `:{store id}:{event/stream id}`)
    - Accept "partial" id `:store id:`, letting the server generate the in-store id?
    - Add an optional `storeId` parameter? But how (parameters are literally the event/stream, so we'd have to change the API)? Again perhaps thinking of the v2 proposal could help
- No change with other methods
- HF events are always kept in the default (local) store
- Accesses: permissions refer to streams across stores using id format `:{store id}:{event/stream id}`


## Use cases collected so far

==**Perki: contribution needed**==

**Addmin**: integrating E-Post (service to digitise postal mail) data into Pryv.io

Hypothetical: access rights managed in external service like Active Directory, exposed as a read-only streams store, whose streams (i.e. user groups) are then used to set permissions for events in other (e.g. default/local) stores.


---

## Temp stuff for discussion follows

```
/**
 * A generic query for events.get, events.updateMany, events.delete
 * @typedef {Object} EventsGetQuery
 * @property {string} [id] - an event id (inconpatible with headId)
 * @property {string} [headId] - for history querying the id of the event to get the history from (incompatible with id))
 * @property {Array<StreamQuery>} [streams] - an array of stream queries (see StreamQuery)
 * @property {('trashed'|'all'|null)} [state=null] - get only trashed, all document or non-trashed events (default is non-trashed)
 * @property {boolean} [includeDeletions=false] - also returns deleted events (default is false) !! used by tests only !!
 * @property {timestamp} [deletedSince] - return deleted events since this timestamp
 * @property {boolean} [includeHistory] - if true, returns the history of the event and the event if "id" is given - Otherwise all events, including their history (use by tests only)
 * @property {Array<EventType>} [types] - reduce scope of events to a set of types
 * @property {timestamp} [fromTime] - events with a time of endTime after this timestamp
 * @property {timestamp} [toTime] - events with a time of endTime before this timestamp
 * @property {timestamp} [modifiedSince] - events modified after this timestamp
 * @property {boolean} [running] - events with an EndTime "null"
 * @property {Object} [NOT] - events without the given field value
 * @property {string} [NOT.id] - events not matching this id (incompatible with id)
 */

```

## Time Frame selection recap

Note:
- if (toTime != null && fromTime == null) => toTime = endTime - 24h
- if (running) ==> query (endTime == null)


| Coerced  |          | DB params                                     |
| -------- | -------- | --------------------------------------------- |
| fromTime | toTime   |                                               |
| -        | -        | -                                             |
| F >= now | -        | ==A:== endTime >= F <br> ==B:== time >= F                                 |
| F <= now | -        | ==A:== endTime >= F or endTime = null   <br> ==B:== time >= F or duration = null              |
| F <= now | T <= now | ==A:== time <= T and endTime >= F  <br> ==B:== time <= T and time + duration >= F                   |
| F <= now | T >= now | ==A:== time <= T and (endTime >= F or endTime = null) <br> ==B:== time <= T and (time + duration >= F or duration = null) |
| F >= now | T >= now | ==A:== time <= T and endTime >= F <br> ==B:== time <= T and time + duration >= F                     |


```javascript=
// mongodb
if (T) {
    query.time = {$lte: T}
}

// running
if (running) {
    query.endTime = null;
} else if (F) {
    if ( F >= now() && ( T == null || T >= now)) {
        query.$and.push({$or: [{endTime: {$gte: F}}, {endTime: null}]});
    } else {
        query.endTime = {$gte : F};
    }
}



```

```javascript=
// sql
if (T) {
    query += ` AND time <= ${T}`;
}

// running
if (running) {
    query += ` AND endTime = null`;
} else if (F) {
    if ( F >= now() && ( T == null || T >= now)) {
        query += ` AND (endTime >= ${F} OR endTime = null)`;
    } else {
        query += ` AND endTime >= ${F}`;
    }
}

```

### Proposal

Discard current params: `running`, `fromTime`, `toTime`

Add following params:
 - `timeBefore` - should be translated to 'time <= {value}'
 - (optional) `endTimeNull` - should be Translated to 'endTime = null'
 - `endTimeSince` - should be translated to 'endTime >= {value}'
 - `endTimeNullOrSince` - should be translated to '(endTime >= {value} OR endTime = null)'

`timeBefore`, `endTimeSince` and `endTimeNullOrSince` being mutualy exclusive

*alternative*
 - `endTimeQuery` - `{since: NUMBER , null: boolean}`
