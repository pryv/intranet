|         |                       |
| ------- | --------------------- |
| Authors | Pierre-Mikael Legris, Ilia Kebets |
| Reviewers | |
| Date    | June 25th 2020 |
| Version | 1 |

# System streams

## Motivation

1. Define additional unique fields (currently only username/email)
2. Create permissions on those values, which are currently only accessible by personnal accesses

## Proposition

We will offer access to **account** and **profile** data through the **streams/events API** so we can define **permissions** for them. 

There will be configurable default profile streams whose events will be set in their respective profile streams as well as the **indexed** stream at account creation.

The **indexed** stream defines fields that are unique across a platform. At some point they might be used as **identifiers** for login.

Indexed values should be queryable accross separate cores as their constraints need to be enforced at user creation.

## Deliverables

### API

- user create on core
- get hostings route that include cores hostnames to make user create calls on them

### Config

- indexedFields

### Docs

- modify user creation flow : https://api.pryv.com/reference-system/#account-creation
- Add route to create user on core
- deprecate user create on register

## Design

### Architecure

As we intend to move several API methods to the core API from register, we will include this migration into this task. Mainly this is motivated to unify the codebase from cluster and DNS-less setups.

Methods to migrate:

- user create

### User creation

#### Flow

The user creation flow will be as following:

1. fetch hostings from reg
   - returns cores hostnames
   - if there are several cores behind a hosting, returns the hostname of the less populated one
2. the create user call will be done on the core hostname, with the same method signature as the current one.
3. core receives call
4. core makes booking on reg
   - if already used, return error
5. core saves it
6. core notifies register

#### Method signature

- username
- email
- other indexable values defined in the platform config
- (password) - not indexed

#### Data representation

username, email and other indexable values will be part of the following streams:

- system/Profile/email + system/indexed
- system/Profile/username + system/indexed
- System/profile/fieldName + system/indexed

#### Retro-compatibility

We will maintain the current method in register, but mark it as deprecated in the API docs.

### Default streams

the stream **system** (name to be confirmed as to not conflict with existing installations) will be set by default, with the following children:

- system
  - profile
    - username - indexed
    - email - indexed
    - StorageUsed
      - dbDocs
      - attachedFiles
    - customStream1 - indexed
    - customStream2 - indexed
    - ...
    - customStreamN - indexed
    - Alias (next steps) - indexed values by default
  - indexed

Custom streams can be: phoneNumber, insuranceNumber, ...

Values provided at account registration will be set there.

Events

Values will be stored as events. the indexed one will also be part of the **indexed** stream at creation

Running events will be considered active, which allows to have (for example) multiple active emails for example. the API allows to fetch all running events.

Type:

Strings only, API stringifies numbers. They will have the following Pryv.io types:

- username/string
- Email/string
- 

### Making an event indexed

For events that are in a `system/profile/XXX` stream, you can index them by adding `indexed` to its `streamIds`.

### Storage

We will define a new collection on core for idnexed values which will be shared by all users of a core, this will allow to enforce the uniqueness constraint even after we segment user databases again.

## Migration

1. account to system/profile
2. Profile to system/profile
3. add indexed shared collection

## Iterations

### 1. create user on core

1. just user creation, without the new flow with register, which will be done last (see 4.)

### 2. System streams

1. on account creation and profile manipulation, edit the streams data
2. migrate account and profile data to streams
3. editing through 1 API should update the 2nd representation

### 3. Indexed values

1. add collection to core's mongo to enforce uniqueness
2. add *indexed* system stream

### 4. update user creation flow

1. include calls where core books and notifies register

2. add route on register

## Test cases

Please add link to feature branch's test cases, or the branch name and test tags

## Discussion

### Retro-compatibility

#### system stream id namespace

As the streamId `system` might be already used by some of our clients, we might have to define a streamId that was forbidden by the current naming.

### Storage

1. Should we migrate account and profile data to streams?

2. or should we keep them where they are and do a separate storage call when querying streams?

Option 1 requires more code to change, but should be a smaller codebase in the long term, I would go for 1

