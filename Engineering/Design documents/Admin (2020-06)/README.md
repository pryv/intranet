
|         |                       |
| ------- | --------------------- |
| Authors | Ilia Kebets |
| Reviewers |  |
| Date    | June 19th 2020 |
| Version | 1  |

# Admin tools for Pryv.io

## Motivation

Allow customers to change platform settings through an API instead of requiring a root access to the machine(s) where data is stored.

## Proposition

We propose to have a way to update the configuration of a running Pryv.io platform through an API, the changes would then propagated to the services which will reboot loading the new parameters.

This administration API will handle multiple roles which will define what configuration parameters they have access to. A strong authentication will be used, providing the required security.

## Deliverables

### API

- GET /admin/settings (already available)
- PUT /admin/settings (already available)
- GET /updateStatus (some way to notify that the services have been rebooted)
- POST /auth/login
- POST /auth/logout
- GET /users
- POST /users
- PUT  /users/:username
- DEL /users/:username

### Docs

- API reference:
  - config.get
  - config.put
  - config.getUpdate
  - config.login
  - config.logout
  - config.createUser
- Update [Pryv.io setup guide](https://api.pryv.com/customer-resources/pryv.io-setup/)
- Update INSTALL.md

### Frontend

- Show config with updates
- sign in
- Change username/password
- create user

## Web app

- format using JSON.stringify(txt, null, 2) *
- check if JSON is malformed
  - can be done API side: it shouldn't crash, but it should return a readable error for the web app to display. 400 & readable error
  - Add check that there are required fields: vars? or the ones below?

### docs 

(comments in platform.yml)

- either refactor platform.yml to include them
- or add them in some external resource

### Status

- automatic check from web app
- or reference to documentation to perform manual validation

#### Checks

Insert list of params and their checks

- 
- 

### Export/import

Add button to download configuration, and one to upload 1

### Display produced values

This is close to the Status task, as displaying the produced values means fetching them, then 

## Configuration/template

remove reporting from variables in platform.yml

## API

### Role management

The Highest roles are "admin", they allow access to all settings + the ability to add/remove users.

We'll define roles with read/write permissions per config parameter.

There can be multiple "admin" users

#### Onboarding

1. When booting Pryv.io, in the terminal, a randomly generated admin key is shown. It must be communicated to the platform admin
2. Once he logs into it, he must generate a new password and enter a username, so that we can identify the person behind the account.

#### Role ideas

For each user, you would get a set of categories that correspond to the config categories with R/W/Nothing

### Authentication

1. whitelist IP
2. token with short expiration time
3. MFA (reuse our component if possible)

### Configuration

1. Backup before update

## Update propagation

The main difficulty of this task is to define how an update to the config API will be propagated to the different services.

We have mainly 2 types of them:

- Our node services and custom images (NGINX, MongoDB)
- Default docker images
  - Influx

## Discussion

### Configuration parameters

- single https://github.com/pryv/config-template-pryv.io/blob/central/pryv.io/single-node/config-leader/conf/platform.yml
- cluster https://github.com/pryv/config-template-pryv.io/blob/central/pryv.io/cluster/config-leader/conf/platform.yml

### Problems

3. update propagation
   1. API change
   2. Leader notifies
   3. Follower fetches
      1. writes config to disk for debug in case of failure
   4. Provide config to services and reload them
      1. How does follower provides them a new config
      2. how does follower tells them to reload
   5. Services notify admin that they have been updated, possibly with hash of config

4. *optional* Replace leader-followers symmetric keys with asymmetric ones
5. Review resilience of system (check how kubernetes does this)

### Setup - for later

1. Setup parameters
2. Setup SSL
3. Deploy