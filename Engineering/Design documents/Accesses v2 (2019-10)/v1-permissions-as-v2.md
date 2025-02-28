# Accesses v1

This document expresses the current access permissions mechanism using the accesses v2 mechanism.

### Tests

API methods on `/accesses`:

- api-server/test/accesses-app.test.js
- api-server/test/accesses-personal.test.js

**API methods on other resources testing permissions mechanism:**

- api-server/test/permissions.test.js

API methods on events fetching and creation covering mainly deletion and expiration mechanics - not relevant except their fixtures: 

- api-server/test/acceptance/accesses.test.js

### Implementation

- model/src/accessLogic.js

# Current implementation

The goal is to define the current permissions using v2 accesses parameters (scope + permissions).

In order to do so we will: 

1. define the scope of each current API method
2. define v1 permissions (stream+level only) by the API methods they allow
3. from 1 & 2, derive v1 permissions per v2 permissions

We assume that this is for app & shared access, we will specify when it only applies to one.  

**Resources**: events, streams, accesses, notifications/webhooks
**Actions**: read, create, update, delete

Additionnally to a simple yes/no, we will specify for events & streams if it applies to the current stream level or its children only.  

**Accesses:** we'll see that later.



## Permissions v1 per API method

Maybe we need to specify the access type

*is events.getAttachement only valid for what*

### streamId+read

#### API methods

- events.get/getOne/getAttachments
- streams.get

#### v2 accesses

event.streamId <= scope.streamId

events.read

streams.read

#### streamId+contribute

#### API methods

streamId+read

- events.create
- events.update
- events.start
- events.stop
- events.delete
- events.addAttachment
- events.deleteAttachment

#### v2 accesses

event.streamId <= scope.streamId

- events.*
- streams.read

### streamId+manage

#### API methods

streamId+contribute

- streams.create
- streams.update
- streams.delete

#### v2 accesses

event.streamId <= scope.streamId

- events.*

- streams.*

### tag+read

- events.get/getOne
- streams.get

### tag+contribute

- 























##Permissions v1 as v2

### streamId+read

- events.read (=<)

- streams.read (<) - should be =


- webhooks/notifications.register (app OK, shared?)
- webhooks.create (app)

- streamId+contribute

- events.create
- events.update
- events.delete

- accesses.read (app access - for shared only with level<=this)

### streamId+manage

- streams.create (children)
- streams.update (=)
- streams.delete (=)

- accesses.read (app access - for shared only with level<=this)

### tag+read

- 

### tag+contribute

### tag+manage



### Webhooks

- can only be created by app accesses

### Audit

- Personal tokens can audit all accesses
- shared accesses can only self-audit
- app accesses can audit themselves and subaccesses (returned by accesses.get - `accessLogic.canManageAccess()`)

### Profile

TBD

## Issues

streamId/level
- accesses.read (app access - for shared only with level<=this)
	- HOW DOES IT SEE ACCESSES WITH PERMISSIONS OF STREAMS OUTSIDE OF ITS EXPANDED STREAMS??











