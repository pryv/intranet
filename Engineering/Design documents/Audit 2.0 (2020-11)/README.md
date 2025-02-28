|         |                       |
| ------- | --------------------- |
| Authors | Ilia Kebets |
| Reviewers | Pierre-Mikael Legris |
| Date    | November 13th 2020 |
| Version | 1                  |

# Audit logs 2.0

## Motivation

The current implementation of audit logs: [service-router](https://github.com/pryv/service-router) is unmaintainable as it's written in Rust. The Web service that offers to access these logs [service-audit](https://github.com/pryv/service-audit) reads from the syslog which does not work after a few thousand lines, as it timeouts.

The goals of reimplementing Pryv.io audit are:  

1. rewrite auditing as API method middleware in service-core to write into a message queue  
2. write a consumer that reads & writes it into rsyslog as was done before  
3. write a consumer that reads and writes it into an indexed SQLite database used by service-audit  

## Proposition

As `api-server` implements multiple transports (HTTP and Socket.io), in order to track all API method calls, we need to work below express which is used by HTTP only. Therefore, we will execute it for all [API.js call stacks](https://github.com/pryv/service-core/blob/1.6.3/components/api-server/src/API.js#L165).  
It can be built upon 2 middlewares in & out to respect the previous process that wrote 2 entries, however, it is highly likely that the entry without the API response was useless from the get go.
Therefore it should be a single middleware registered at the end of the stack so it has the HTTP response status.

As we hash the token in these audit logs, we might implement an external **cache for hashed tokens** that will be shared by multiple `api-server` processes

We will use the NATS message queue as it is used by HFS and Webhooks already. The middleware will write to a `Audit` channel to which different consumers such as syslog and Sqlite will subscribe.

## Deliverables

1. Upgrade for service-core that writes audit logs into NATS
2. NATS consumer that writes to rsyslog
3. NATS consumer that writes to Sqlite
4. Upgrade for service-audit that reads form Sqlite

### Middleware

- token hash should be stored with the Access
- unknown tokens should not compute hash, but print `"unkown token"`. There is no loss of information compared to the hash of an unknown token.

### Consumers

A common base for:
- registering
- message parsing

#### Syslog



#### Sqlite

- 1 SQLite database per account

## Discussion

### MQ producer implementation

Having it implemented in as a service-level middleware requires to implement it for each web-service, but the implementation on the production side should be lightweight, and modular so we could reuse most of the code between our web services.

### MQ vs native

Using a message queue decorrelates the audit logging from the action, introducing a risk that a performed action is not logged in case the logging systems (MQ and consumers) are not working.
To prevent this, we might implement the full audit logging or part of it (only the one writing to SQLite) into core.
We will need to handle concurrent writes to SQLite databases which might be a source of database corruption. To investigate: [locking in SQLite 3](https://sqlite.org/lockingv3.html)

### Message format

- Should it be done by the Middleware or by consumers?
- We might want to select audit log format in consumers configuration if we wish to have something else than what we have now
  - forwardedFor: is HTTP header linguo, represents source/orign

## Iterations

### 1. Technology PoC

- Middleware with raw data
- Consumer writing in a per-user SQLite database

### 2. TBD

## Test Cases

simple API call containing all non-null requested fields (https://api.pryv.com/reference/#audit-log), write consumer that connects to NATS and reads the content
- valid token
- unauthorized token (for error)
- unknown token

## Out of scope

TBC
