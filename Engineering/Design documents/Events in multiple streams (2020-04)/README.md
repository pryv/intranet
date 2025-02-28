
|         |                       |
| ------- | --------------------- |
| Authors | Pierre-Mikael Legris, Ilia Kebets |
| Reviewers | Simon Goumaz (v1) |
| Date    | April 21th 2020 |
| Version | 3                  |


# Events part of multiple streams

## Motivation

Allow Events to belong to multiple streams (contexts).  
Currently, adding another context requires to tag all events that are to be included in the new context **as well as the current one**, which is tedious and redundant.

## Proposition

We propose to **allow events to be part of multiple streams** without preventing them to be part of a streams lineage.

This will allow us to **remove tags** from our API as their functionality can be achieved using streams that are part of another tree.

We also introduce an **aggregate update method for events** to add or remove a context.

We will **remove Pryv.io's timetracking capabilities** in this release as they are unused by our current customers and are expensive to maintain. They have been marked as *deprecated* on [api.pryv.com](https://api.pryv.com) after asking all our customers if they are using it.

## Deliverables

### API

- Support for multiple streams per event
- Deleted streams won't delete events if they have more than 1 streamId, but update its streamIds
- events.aggregateUpdate API method
- Deprecate streams.delete mergeEventsWithParent parameter
- Remove events.start
- Remove events.stop
- Remove stream.singleActivity

### Docs

- API reference

  - event.streamIds
  - event.streamId: mark as deprecated
  - event.tags deprecated
  - events.aggregateUpdate
  - streams.delete mergeEventsWithParent parameter deprecated
  - Remove events.start
  - Remove events.stop
  - Remove stream.singleActivity
  - add new errors

- Multiple streams guide

  - Why
  - How to use

  

## Discussion

### same-lineage streamIds

There is an argument against allowing an event to be part of streams that have a lineage relationship, i.e. an event can't be part of streams A & B if B is a descendant of A or vice-versa.

The argument to prevent this is that it does not make any sense when considering streams as concepts for classification.

However, enforcing it induces that Pryv.io either:

1. prevents moving a stream because at least one of its events is also part of a destination parent stream - which the user or app developer must handle

2. Allows it, while removing the parent stream from the conflicting events (decision: keep the youngest child, aka "most specialized stream")

Mainly this induces a **loss of information** which prevents an easy rollback.

#### Use case

For example if a picture was in a stream "dogs" and "labrador", which aren't related. Moving the "labrador" stream under "dogs" would involve removing the stream "dogs" from all its events. If we move the "labrador" stream outside of its parent "dog", we will have removed all events previously in stream "dogs", possibly **preventing access** to it.

#### Tradeoff

Allowing events to be part of a streams lineage does let end users and app developers to build some **redundant** classifications, but prevents loss of information.

`[...] trop de complications pour un gain plus que discutable dans le cas d’une API qui se veut générique.`

### mergeEventsWithParent

We propose to remove this parameter whose functionality will be replaced by the aggregate event update.  

This feature makes sense in the case where `streamId` exists, it should therefore be kept as long as we support it. We'll therefore deprecate it, linking to the events.aggregateUpdate function.

### Modifying/Deleting events

Modifying an event is allowed if you have a contribute permission to at least 1 streamId. Same for deletion.

Modifying its streamIds works as following:

- add a streamId: requires contribue-level on the new streamId
- Remove a streamId: requires contribute-level permission on it

### Reading events

When returning events, should the API filter out the streamIds for which the access has no permission?

Since we propose to use detached streamIds as base for sharings, we might not want to include this information.

Additionnally, it is convenient if a client wishes to update streamIds, he will not have to work around the streamIds he doesn't have permissions for.

## API

### Events

Currently, an event has a field `streamId` which is a string and is mandatory.

We propose to change it by a `streamIds` field which is an array of strings, although there will be a *transition phase*.

#### Transition

Since we have customers in production that are using the *single* `streamId` field, we don't want to break their implementation. Therefore we propose to temporarily **allow both properties**:

```json
{
  "id": "ck5cahsj4000nnkpvf9ovg82k",
  "time": 1578910414,
  "streamId": "insurance",
  "streamIds": ["insurance","tax"],
  ...
}
```

Both fields are **allowed on creation and update, but exclusively**. The API returns both, the first element of `streamIds` in `streamId` if there are more than one.

This will allow us to deprecate the single streamId usage smoothly.

## Test Cases

### events.get

- must return streamIds & streamId containing the first one (if many) [x] - 1GR9
- must return only the streamIds you have a read access to

### events.create

- must not be able to provide both streamId and streamIds [x]- POIZ


#### when using streamId

- must return streamIds & streamId [x] 

#### when using streamIds

- must return streamIds & streamId containing the first one [x] - 1GR9
- [1GZ9] must clean duplicate streamIds
- [5NEZ] must forbid providing an unknown streamId
- [1G19] must forbid creating an event in multiple streams, if a contribute permission is missing on at least one stream

### events.update

- must return streamIds & streamId containing the first one (if many) [x] - 4QRX
- must allow modification, if you have a contribute permission on at least 1 streamId
- must return only the streamIds you have a read access to

#### when modifying streamIds

- must forbid to provide both streamId and streamIds
- [01BZ] must forbid providing an unknown streamId
- [4QRX] must allow streamId addition, if you have a contribute permission for it
- [4QZU] must forbid streamId addition, if you don't have a contribute permission for it
- must allow streamId deletion, if you have a contribute permission for it
- must forbid streamId deletion, if you don't have contribute permission for it
- **TODO**: add check that these don't influence streamIds that are non readable

### events.start

- [5C8J] must not allow a running period event with multiple streamIds
  - to change: remove
- must return a 410 (Gone)
- remove all other tests

### events.stop

- must return a 410 (Gone)
- remove all other tests

### events.delete (trashed)

- must return streamIds & streamId containing the first one (if many) [x]
- must return only the streamIds you have a read access to
- [AU5U] must forbid deletion of trashed event, when no write access on all streams -
  - To change to : must allow deletion, if you have a contribute permission on at least 1 streamId
- [XT5U] must flag the multiple stream event as trashed, when write access on all streams
  - To change to : must allow trashing, if you have a contribute permission on at least 1 streamId

### streams.create

- must forbid setting the "singleActivity" field

### streams.update

- must forbid setting the "singleActivity" field

### streams.delete

#### When the event is part of at least another stream outside of its descendants

- ​	[J6H8] must not delete events, but remove the deleted streamId from their streamIds [x]

#### When the event is part of the stream and its children

##### when mergeEventsWithParent=false

- [J6H9] must delete the events
- ????[J6H1] Multiple streams events attached should be deleted if all streams they belong are deleted

##### when mergeEventsWithParent=true

- [J7H8] must not delete events, but remove all streamIds and add its parentId

### events.aggregateUpdate

TBD

- (Not ok) must allow adding a streamId to multiple events
  - getEvents
  - events.forEach(e=>e.streamIds.push('newStreamId'))
  - aggregateUpdateEvents(events)
- (not ok) must allow removing a streamId of multiple events
  - getEvents
  - events.forEach(e=>e.streamIds.remove('newStreamId'))
  - aggregateUpdateEvents(events)

### Migrations

- must migrate from 1.4.0 to 1.5.0 (transform event.streamId to event.streamIds and remove stream.singleActivity)

## Storage

This change will introduce a database migration. - to be completed on how streamIds will be stored [x]

- change all `streamId: 'ABC'` fields into `streamIds: ['ABC']`
- remove singleActivity from streams

## Out of scope

TBC
