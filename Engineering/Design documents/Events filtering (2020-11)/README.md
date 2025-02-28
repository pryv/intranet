|         |                       |
| ------- | --------------------- |
| Authors | Ilia Kebets |
| Reviewers |  |
| Date    | November 17th 2020 |
| Version | 1                  |

# Events.get filtering

## Motivation

We require more granular filtering for the events.get API method such as:

- boolean **AND** for streamIds

This will avoid to have to declare additional stream structures to match requests that would be answered with new filter parameters, as current customers intend to do.

## Proposition

We need to extend the current [events.get](https://api.pryv.com/reference/#get-events) API parameters to express boolean **AND** operations additionally to the current boolean **OR**.

We might include in this feature release other boolean operators for streams, such as **negation (NOT)**.

## Deliverables

1. API extension design
2. Feature implementation
3. Documentation

### API extension design

#### current implementation

Currently, the API accepts an **array of streamIds**, which are interpreted as a union:  

```
["streamA", "streamB"] => "streamA" || "streamB"
```

They are provided as query parameters as following:

```
?streams[]=streamA&streams[]=streamB
```

#### Proposed extension

We can keep the current list to indicate **OR** operations between multiple sets which will now accept some form of **AND** and **NOT** operators.
For example:

```
["streamA&&!streamB", "streamC"] => ("streamA" && ! "streamB") || "streamC"
```

This allows us to express all boolean expressions using [De Morgan's law](https://en.wikipedia.org/wiki/De_Morgan%27s_laws).

**Con:** in query parameter format, it is slightly counter intuitive as `&` are interpreted as **OR**:

```
?streams[]=streamA%26%26%21streamB&streams[]=streamC
```

### Feature implementation

Implementation of conditional **AND** requires to extend the boolean expression to the children streams.

### Documentation

update [events.get](https://api.pryv.com/reference/#get-events).

## Implementation as per 3rd Dec 2020


### Discussion

- `EXPAND` and `NOTEXPAND` could be fully removed from the implementation 
    as they are not directly exposed.
  -  `{EXPAND: "A"}` produces the same result than `['A']`
  - `{NOTEXPAND: "A"}` produces the same result than `{NOT: ['A']}`

- To force strict equality to a stream Id without including its 
    children the internals uses  `EQUAL`, `NOTEQUAL`, `IN` and `NOTIN` 
  - The implementation could be simplified by using only `IN` and `NOTIN`  
     as they will be converted at last stage to `EQUAL` and `NOTEQUAL` if 
     they contains only one item.
  - If these notions should be exposed to the API we might consider exposing 
     only  `IN` and `NOTIN`. To avoid confusion we might consider to choose 
     other terms to explicitely quote that **childs will not be considered**

### Exposed API side 

- `['A','B']` => Matches the streams of any of their children
- `{NOT: ['A','B']}` => Does not match any of the streams or any of their children
- `{OR: [selector1, selector2, ...]}` Any of the selector should be satisfied
- `{AND: [selector1, selector2, ...]}` All of the selector must be satisfied

### Sugar and conversion

- `['A','B']` => `{OR: [{EXPAND: "A"}, {EXPAND: "B"}]}`  
- `{NOT: ['A','B']}` => `{OR: [{NOTEXPAND: "A"}, {NOTEXPAND: "B"}]}`
- `{EXPAND: ["A"]}` => `{IN: ['A', 'childA1', 'childA2']}`
- `{NOTEXPAND: ["A"]}` =>  `{NOTIN: ['A', 'childA1', 'childA2']}`

###  Internal Syntax

- On single streamId
  - `{EQUAL: "streamid"}` One the event's streamIds must be equal to `streamId`
  - `{NOTEQUAL: "streamid"}` None of the event's streamIds must be equal to `streamId`
  - `{EXPAND: "streamid"}` One the event's streamIds must be equal to `streamId` or one of it's childrens
    - This is converted to `{IN: ["streamId", "child1", "child2"]}` or
      `{EQUAL: "streamId"}` if streamId has no child.
  - `{NOTEXPAND: "streamid"}` None of the event's streamIds must be equal to `streamId` or one of it's childrens. 
    - This is converted to `{NOTIN: ["streamId", "child1", "child2"]}` or
      `{NOTEQUAL: "streamId"}` if streamId has no child.
- On multiple streamIds
  - `{IN: ["streamId1", "streamId2", ... ]}` One the event's streamIds must be equal to `streamId1` or `streamId2`, ...
  - `{NOTIN: ["streamId1", "streamId2", ... ]}` None of the event's streamIds must be equal to `streamId1` or `streamId2`, ...

##### Aggregators

- `{OR: [selector1, selector2, ...]}` Any of the selector should be satisfied
- `{AND: [selector1, selector2, ...]}` All of the selector must be satisfied

## Test Cases

TBD

## Out of scope

Do we include 
