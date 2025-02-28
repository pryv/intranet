# Completion plan for Java-lib
This document presents a plan of required efforts to have a finalized Java-lib for Pryv.

This is version 6 (06.12.2017) of the document and the author is Thiébaud Modoux.

## Missing API features

### [Batch calls](https://github.com/pryv/lib-java/issues/30)

Time estimation: 5 days

Priority: high

#### Motivation

Batch calls are of primary importance since they could be used instead of any other missing API feature. However, this does not address the need of implementing data Model for each of the missing API feature (Accesses, Profile, Account, FollowedSlice).

In conclusion, we still have to decide if we prioritize on completing the missing API features or implement Batch calls and rely on them to fill this gap.

#### Design proposition

A [Batch call](http://pryv.github.io/reference/#call-batch) is  defined by a set of API calls that will be executed in a serial and asynchronous way, and will produce a list containing the result of each call (error or success).

On the implementation side, we first need to implement a Model for Batches (model/BatchCall.java). A BatchCall object will basically represent an array of API calls.

Each elements of this array will contain a 'method' property which corresponds to any of the CRUD operations of the API (create, get, update, delete for events, streams, accesses, account, followedSlices, profile).

The second property of a BatchCall element will be the 'body' that can contain an Event, a Stream, an Access, an Account, a FollowedSlice, a Profile, depending on the choice of 'method'. This is why we need to complete the Models with missing ones (model/Access.java, model/Account.java, model/FollowedSlice.java, model/FollowedSlice.java, model/Profile.java).

BatchCall objects will also have a function to add a new element to the BatchCall, taking as arguments the 'method' choice and the 'body' object.

Then, a connection/ConnectionBatchcall.java class will handle the sending of this BatchCall and process the results. Thus, it will have a 'send' function that takes the BatchCall and a BatchCallCallback as arguments and execute the batch call against the API.

Another function 'handleResponse' will process the answer from API by iterating through the array of results and call the BatchCallCallback at the end, which will be added as new interface (interfaces/BatchCallCallback.java).

We still have to decide if this 'handleResponse' will be a totally new function or if we will augment the one already implemented in the ApiResponseHandler in api/OnlineEventsAndStreamsManager.java.

### [Profile and access-info](https://github.com/pryv/lib-java/issues/12): get (public/app profile, access-info) [2 days]

### [Start/stop for Event](https://github.com/pryv/lib-java/issues/24) [2 day]

### [GetOne Event](http://pryv.github.io/reference/#get-one-event) [1 day]

## Incomplete API features

### [Filters](https://github.com/pryv/lib-java/issues/40) [3 days]

### [Attachments handling](https://github.com/pryv/lib-java/issues/10) [4 days]
Attachments handling is incomplete, in particular, we currently only handle the first attachment of an Event.

### [Fix Stream.merge](https://github.com/pryv/lib-java/issues/9) [3 days]

### Handle event/stream/accessDeletions in GET response
The deletions object (containing all deleted resources if includeDeletions parameter is set) should be forwarded up to the user, alongside with the usual list of retrieved resources.

Since this feature was incomplete, we chose to remove it until we feel a need from users.

## Improvements

### [Improve server time usage](https://github.com/pryv/lib-java/issues/34) [2 days]
To ensure that client and server are based on the same time referential, we have to maintain a delta, recalculated with each server response:

> deltaTime = serverTime - System.currentTimeMillis() / millisToSeconds;

Then, all API request including a time parameter will be adjusted with this delta:

> DateTime(time / millisToSeconds + deltaTime);

We chose to remove this feature for now, since it was only partially implemented in the library and thus not used at all.

## Documentation

### [Improve documentation](https://github.com/pryv/lib-java/issues/7) [1 days]

### Simplify the JSON (de)serialization

The handling of the API responses in currently implemented in a custom JSONConverter that rely on a JSON serialization library but also need to do some prework. Moreover, this prework is absolutely not generic (bunch of different functions for Events, Streams...). 

We are convinced that we can find a better solution that will delegate the most of the work to the JSON serialization library and only requires some generic tasks on our side.

Some research around JSON (de)serialization could be undertaken and we consider to change the JSON serialization library used if the current one does not fit our needs.

Finally, we want to improve the current method of providing fields keys, either by avoiding this, or by storing them in resources Models.

Examples of libraries:
* [Flexjson](http://flexjson.sourceforge.net/)
* [Owlike](http://owlike.github.io/genson/)
* [Mkyong](http://www.mkyong.com/java/how-do-convert-java-object-to-from-json-format-gson-api/)

### Simplify/improve the test suite

The current test suite seems a bit messy and give not enough or unclear information (which exact feature is failing? what are we testing?).

We plan to have a more clear and intuitive hierarchy in our tests by following a describe/it scheme and by splitting our big test case in small subtests (each test should reflect one precise expected behavior).
For example, when testing an update of an event, we want to have a test for each updatable field of the event.

Moreover, the test suite should also be simplified at the same time that we are improving the library design. For example, simplifying the JSON (de)serialization (see previous point) will also simplify the way we test it.

### CreateOrReuse on self object

CreateOrReuse function in Event/Access are currently statically called but we can maybe make them callable from the actual Event/Access.

## DONE!

### Unification of the javalib package

We would like to drop the current split of the javalib in 3 modules (Commons/Android/Java) and instead propose one core package that will work independantly on Java and Android. This will be possible by having/using only cross-platform features/libraries.

Right now, only the database part differs between Java and Android module, so since we will probably drop the cache feature, the new implementation will be easily packaged in one module.

Moreover, packing everything in one core package will result in an easier release process and also allow to plug eventual future external modules (cache, async, ...).

### Drop cache feature

We realize that the cache feature is something really complicated to implement and that it is currently in an incomplete state. 

Moreover, it seems that this feature is currently not something that clients use or need. Thus, we decide to alleviate the core package of the lib from this feature and consider to later bring it back again as an optional package if needed.

### Drop callbacks and async scheme

Callbacks scheme seems not to be the way to go with Java and using async http calls make the code more complex (both in library and for the developer using the library).

We will prefer sync http calls (Okhttp does that) and function that return directly a response to the user or throw Exceptions. This will be the task of the user to handle these Exceptions (try/catch), as it is usually done in the Java community (some research could be undertaken to prove this).

[Research on how to do concurrency in Android.](https://www.linkedin.com/learning/android-app-development-restful-web-services/concurrent-programming-in-android)

### [Accesses](https://github.com/pryv/lib-java/issues/11)

Time estimation: 5 days

Priority: high

#### Motivation

Accesses are of primary importance since it is a missing main API feature and some customers start to explore this feature.
It will also simplify the Android project with LSI.

#### Design proposition

[Accesses](http://pryv.github.io/reference/#access) will be represented by a new Model (model/Access.java) containing the following properties: 'id', 'token', 'type', 'name', 'deviceName', 'permissions', 'lastUsed', 'created', 'createdBy', 'modified', 'modifiedBy', with the corresponding setters/getters.

Moreover, we will complete the existing (but empty) connection/ConnectionAccesses.java class by implementing all the CRUD methods (get, create, update, delete). This part will be very similar to the one for Events so we can reproduce the same structure as in connection/ConnectionEvents.java.

Since all these CRUD methods make use of OnlineEventsAndStreamsManager, we will have to augment (and rename/split) the OnlineEventsAndStreamsManager with the corresponding request and response handling functions.

Finally, we also need to add some interfaces (similarly to Events: interfaces/AccessesCallback.java, interfaces/GetAccessesCallback.java, interfaces/AccessesManager.java).

### [Connection methods without callback](https://github.com/pryv/lib-java/issues/8) [1 day]

### [Improve release procedure](https://github.com/pryv/lib-java/issues/41) [1 day]

### [Complete Cache](https://github.com/pryv/lib-java/issues/5) [1 week]

### [Cache synchronization](https://github.com/pryv/lib-java/issues/13) [1 week]

### [Cache cleaning](https://github.com/pryv/lib-java/issues/27) [3 days]

### [Query Generator](https://github.com/pryv/lib-java/issues/26) [1 day]

### Drop OnlineManager

The new implementation tends to delegate most of the work of OnlineManager to Connection objects, so the OnlineManager will be, in the long term, almost empty. We will then drop this class as soon as all its code is split among Connection classes.

### Update methods should not try to update read-only fields
Currently, all update methods take a resource in parameter. Thus, the update call will try to update all fields contained in the provided resource, including read-only fields such as id, created, createdBy, modified, modifiedBy...
We have to strip these parameters so that only alterable fields are included in the update call (white list).
To do so, we define a list of alterable properties (in model/Event,Stream,Access) according to Api specifications and when calling update methods (in ConnectionEvents/Accesses/Streams), we apply a function on the resource passed in parameter that will return a new resource with only alterable fields set, then we delete this temporary new resource. This function will be implemented in the models (model/Event,Stream,Access).

### Throw specific and custom Exceptions
We currently throw only IOExeption, which is not really correct and not specific enough. We have to differentiate between APIExceptions (API response with erroneous status), lib Exceptions and standard Exeptions (IO, JSON). We want the APIExceptions to contain all information returned by the API (id, message, data, subErrors). We want the user of the library to be able to have only one general catch block.

* Exception
  * ApiException
  * IOExceptions

### Review parameters and return type for delete
We have to clarify the design for the delete call, both for Events and Streams. Do we provide an id of the resource to delete or the actual resource? Then, this method should first trash the resource and then delete it if the call is repeated. Should we return both time with a trash/deletion id? Or should we return each time with a resource (the trashed resource, then the deleted resource, how to define a deleted resource?)?

Current idea is to provide the resource id as parameter and always return the resource, with trashed to true after the first delete call, and with deleted to true after the second delete call.
  