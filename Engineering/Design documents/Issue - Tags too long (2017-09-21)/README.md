|         |                 |
| ------- | --------------- |
| Author  | Thiébaud Modoux |
| Version | 1 (21.9.2017)   |

# Tags length limitation

This document relates an [issue](https://github.com/pryv/service-core/issues/69) in service-core about Pryv tags
and describes our decisions on how to fix it.

This is version 1 of the document and the author is Thiébaud Modoux.

## Issue:

We encountered several mongo crash in service-core due to some events with very long tags created from IFTTT. Since these tags are stored as indexes in mongo, they can break the [mongo index limitation](https://docs.mongodb.com/manual/reference/limits/#indexes) if they contain more than 1011 characters.

This issue is related to the creation and update of events, more precisely to the part of code handling with event tags.
Our original solution operates some tags cleanup but does not address too long tags.

## Fix:

In order to prevent the creation of events with too long tags, we chose to patch both our API (service-core) and the service generating the errors ([bridge-ifttt](https://github.com/pryv/bridge-ifttt)).

Since the ifttt bridge is handling with automated requests, we chose not to block the event creations but strip the tags that exceed a certain length limit.

On the other hand, in service-core, we chose to reject the event creations/updates and throw an explicit error message, so that developers can understand and correct their requests.

In both patches, we defined a tag as too long as soon as it exceeds the limit of 500 characters (half of the breaking limit).
This limit should be documented in our API. 