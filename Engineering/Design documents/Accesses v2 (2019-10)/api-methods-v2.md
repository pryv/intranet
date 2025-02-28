# API methods - v2

This document decribes the required v2 accesses - permission(s) as well as the scope - for each Pryv.io API call.

## Events

### events.get

events.read

event.streamId <= scope.streamId

event.tag = scope.tag

event.time within range

### events.getOne

same as events.get

### events.create

events.create

event.streamId <= scope.streamId

event.tag = scope.tag

event.time within range

### events.update

events.update

event.streamId <= scope.streamId

event.tag = scope.tag

event.time within range

### events.delete

events.delete

event.streamId <= scope.streamId

event.tag = scope.tag

event.time within range

### events.start

events.create, events.update

event.streamId <= scope.streamId

event.tag = scope.tag**???**

event.time within range

### events.stop

same as events.update

### events.addAttachment

same as events.update

### events.getAttachment

same as events.getOne

### events.deleteAttachment

same as events.update

## Streams

### streams.get

streams.read

stream.id <= scope.streamId**???**

stream.tag = scope.tag

### streams.create

streams.create

stream.id < scope.streamId

stream.tag = scope.tag **???**

### streams.update

streams.update

stream.id <= scope.streamId

stream.tag = scope.tag **???**

### streams.delete

streams.delete

stream.id <= scope.streamId

stream.tag = scope.tag **???**

## Accesses

### accesses.get

- accesses.read

### accesses.create

- accesses.create

### accesses.update

- accesses.update

### accesses.delete

- accesses.delete



