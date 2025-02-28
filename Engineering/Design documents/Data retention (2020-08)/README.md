
|         |                       |
| ------- | --------------------- |
| Authors | Ilia Kebets |
| Reviewers |  |
| Date    | August 24th 2020 |
| Version | 1                 |


# Data retention

## Motivation

Automatically delete data according to different criteria such as creation or access time as well as data type.

## Proposition

We propose to add platform parameters that allow to define 2 data retention policy types:

### At **event-level**:

- creation date
- event type

### At **account-level**:

- creation date

- access date: last authorized API call

Additionally, the platform administrator will be able to define at what time and frequency this cleanup will be performed with [cron](https://en.wikipedia.org/wiki/Cron#Overview) precision.

Data erasal executions shall be included in the audit log.

## Deliverables

### Feature

- data-retention executed according to set parameters
- its executions should be part of the audit log:

```json
{
  "id": "cke4chr01001mgxpvbhnzu3ke",
  "type": "audit/data-retention",
  "time": 1561988300.123,
  "forwardedFor": "128.192.0.10",
  "action": "",
  "query": "created<1560556800&type=position/wgs84"
}
```



### Docs

- Data retention guide
- Configuration template descriptions

## Discussion

TBD

## Feature



## Test Cases



## Out of scope

TBD
