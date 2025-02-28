|         |                       |
| ------- | --------------------- |
| Author  | Ilia Kebets 		  |
| Date    | 30.11.2018            |
| Reviewer  | Thiébaud Modoux (Version 1) |
| Version | 2                     |

Fetch accesses deletions
{: .doc_title} 

and related fixes
{: .doc_subtitle} 

# Summary

For demos, it is necessary to retrieve deleted accesses. As they are kept within Pryv.IO, it is an easy tasks that we take.

# Overview

When the API method `accesses.delete` is called, it simply adds a `deleted: NOW_TIMESTAMP` property to the document, which is not accessible because by default accesses retrieval ignores deleted data.  

The goal of this task is to add a `includeDeletions` parameter to the API method which will return the deleted accesses in the `accessDeletions` field.

# Design

We will make an additional call to the MongoDB because it will not be a function that is often called and we can currently affoard the debt, especially since we do this in `events.get`.

Override the [#BaseStorage.findDeletions](https://github.com/pryv/service-core/blob/release-1.3/components/storage/src/user/BaseStorage.js#L120) by implementing its own in [storage/user/Accesses](https://github.com/pryv/service-core/blob/release-1.3/components/storage/src/user/Accesses.js) which retrieves deleted accesses exclusively.

We will add `includeDeletions` (as well as the missing `includeExpired`) in the accesses schema.

We will also replace the [#storage/user/accesses.delete hack](https://github.com/pryv/service-core/blob/release-1.3/components/storage/src/user/Accesses.js#L63) with a [partial index](https://docs.mongodb.com/v3.4/core/index-partial/) on the token key and {type, name, deviceName} tuple depending on the existence of the `deleted` field.  
This change requires a database migration that replaces the accesses collections indices, sets `null` to undeleted accesses and reverses the previous hack renaming the deleted accesses fields `_token`, `_name`, `_type`, `_deviceName` without the `_` prefix, as it is now possible to query them through the API. 
Newly created accesses have a `deleted = null` property, which is not returned by API methods as before.

We will add `includeDeletions` to the API reference.

### Note on personal accesses

They are never deleted. The `accesses.get()` method returns an increasing number of personal tokens.  
As they are always associated with a session. It is the session that handles if the personal token is valid or not as seen [here](https://github.com/pryv/service-core/blob/release-1.3/components/model/src/MethodContext.js#L202).
Sessions leave no trace as they are deleted when expired: [code link](https://github.com/pryv/service-core/blob/release-1.3/components/storage/src/Sessions.js#L51).

# Test / Quality Control

## Automated

GET /accesses

- if `includeDeletions` is set to true, an array `accessDeletions` must be included in the response body

POST /accesses

- upon access creation, the API response should not contain a `access.deleted` field, but it should be stored in MongoDB
- upon access creation, setting the `deleted` field is forbidden

PUT /accesses

- upon access update, setting the `deleted` field is forbidden

Database migration

# Plan (time estimate)

We intend to modify the following artefacts:

- Write tests (3h)
- Replace index (8h)
- implement route (1h)
- update documentation (2h)
- review (3h)
- make release? (1h)

