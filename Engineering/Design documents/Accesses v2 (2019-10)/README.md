|         |                       |
| ------- | --------------------- |
| Authors | Pierre-Mikael Legris, Ilia Kebets |
| Reviewers | Simon Goumaz (v1,2) |
| Date    | October 4th 2019     |
| Version | 3                   |


# Accesses v2

## Motivation

1. Extend access rights functionality
2. Refactor accesses logic into a scope structure, based on API methods, associated to streams, tags & API method params sets.

## Vocabulary

- End user: app developer  
- Currently: v1.4 implementation

## Access creation methods

There should be **at least** 2 access creation methods:  

- Obtained through **`auth.login`** providing a valid username/password pair  
- Obtained through an **`accesses.create`** call providing a valid access token  

We don't want to have the concept of access type anymore. The difference can be handled by accesses expiration and methods scope.  

#### Current implementation

`auth.login`: gives a *personal* access - which is a fixed type  

`accesses.create` gives *app* and *shared* accesses - which are extensible types  

## Proposed functionality

An access is defined by a set of **scopes**, each associated to a list of **permissions**. There is a logical **OR** between 2 scopes.  

### Access scopes

A scope is:  
- at least 1 streamId, the default stream is *root* (still no agreement on its name)  
- optional tags  
- optional parameters, for now: fromTime, toTime  

There is a logical **AND** in between each of these scope parameters.

### Permissions

Permissions are defined by a **list of allowed or disallowed resource-action** pairs:    
- **Resources** are events, streams, accesses, webhooks, ...  
- **Actions** are `read`, `create`, `update`, `delete`   

The documentation will describe what level(s) are required for each API method usage. For example:  

-  `events.getOne` requires `events.read`  
- `events.start` requires `events.create ` and `events.update`.  

By default, the only allowed API method is `getAccessInfo`, but we will probably force to have at least 1 method in the set.

#### Negation

- If no permissions are defined for the scope or parent scope, access is denied.  
- Permissions recursively apply to child scopes unless specific permissions are defined for them.  
- For a given scope, permissions can be granted (`events.read: true`) or denied (`events.delete: false`); unspecified permissions are assumed to be denied.  

### Reasonning

#### Multiple scopes

There are clear use cases when we wish to define different permissions for different scopes. See the example where the external algorithm requires read permissions on heart and stress sensor and a write permission on processed event.

#### At least 1 stream

If the stream is *root*, it needs to be explicited

### Example

~~~yml
- scope: stream1 & stream2 & tag1 & (event time within dateA…dateB)
    - events:
        - read: true
    - streams:
        - read: true
- scope: stream3 & (event time within dateC…dateD)
    - events:
        - read: false // optional, No need to explicit negation
        - create: true
    - streams:
        - read: true
~~~

### Handling of negations on a stream tree

When considering if an action is doable:

if tag: is it tagged as such? -> Yes/No

if stream:   
- build stream tree  
- to each stream that is part of a scope, assign the scope-associated permissions: canEventsRead(), cannotEventsCreate() (or deal with undefined)  
- travel from the stream where you are upstream until you find an answer  

##### Concrete use case

Streams tree:  

~~~
  A
 / \
B   C
 \
  D
~~~

The scope is the following:   
A,D: events.create   
B: !events.create    

If we wish to create an event on D, we get the answer when we ask it directly -> Yes    
If we wish to create an event on B, we get the answer when we ask it directly -> No    
If we wish to create an event on A, we get the answer when we ask it directly -> Yes    

## Access delegation limit

As there is no use case where we might want to limit the access delegation beyond 1 level, this is simply handled by mentioning accesses.create in the permissions.

## Out of scope

### Access creation methods

- ? obtained through a successfull oAuth2 process - No because we use `auth.login`.  
- events.create of a defined type: for later versions  
- validating data: update an event with a certain tag. I propose to create a *validation* event referencing the validated event. Perki proposes to limit the tagging/detagging to the available scope(s)  
