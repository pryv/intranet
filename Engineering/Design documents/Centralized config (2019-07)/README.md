<img src="https://api.pryv.com/design-typora/images/logo-512.png" width="140"/>

|         |                           |
| ------- | ------------------------- |
| Author  | Thiébaud Modoux (Pryv SA) |
| Reviewer|                           |
| Date    | 22.07.2019                |
| Version | 1                         |

<div class="document_title">Pryv.io centralized configuration</div>

# Introduction

Through this project, we aim to simplify the first deployement steps of a Pryv.io platform, i.e. the distribution and setup of the configuration files for the main Pryv.io components (cores, register, ...).

This means that each of these components will ultimately pull its configuration at startup and will reboot when changes occured.

We also want to make secrets management opaque from a user/admin point of view, by automating the generation and sharing (between Pryv.io components) of secrets (symmetric keys, SSL certificates...).

## Objectives

### Iteration 1

At the end of this iteration, we expect the following:
- Core configuration file (core.json) is hosted by service-config-leader
- We can delete core configuration file (core.json) from cores
- Service-config-follower pulls core configuration from service-config-leader
- Service-config-follower distributes this configuration in core containers, which are able to start
- Service-config-follower is able to start and restart other containers
- Configuration is minimal (core.json)

### Iteration 2

At the end of this iteration, we expect the following:
- Service-config-leader serves configuration for all Pryv.io components
- Service-config-follower pulls and distributes configuration for all Pryv.io components
- One user-facing configuration sums up dynamic values
- All configurations are minimal
- This includes:
  - core/core
  - core/audit
  - core/hfs
  - core/mongodb
  - core/nginx
  - core/preview
  - register/dns
  - register/mail
  - register/redis
  - register/nginx
  - static/nginx

### Iteration 3

At the end of this iteration, we expect the following:
- Secrets are auto-generated
- Secrets are auto-shared
- Secrets are settable (via ENV variables?)
- Each component has its own secrets (e.g. "ssoCookieSignSecret", "filesReadTokenSecret" are not different among all cores/registers)

### Iteration 4 (additional features)

At the end of this iteration, we expect the following:
- A configuration change triggers the reboot of according container
- Decide on a strategy for detecting changes (config serial, regular pulls, push?)
- Configuration is self-documented, e.g. Redis config: https://raw.githubusercontent.com/antirez/redis/3.0/redis.conf

## Design

### Service-config-leader

This service will be provided with a single and simple configuration file, which is logic from a user/admin point of view (not from developers point of view, no duplicate values). Additionnaly, it will centralize the configuration templates for all Pryv.io components.

From these files, it will prepare and serve the final configuration for each Pryv.io component, through an Express API, e.g.:
> GET /conf/:component, e.g. GET /conf/core

Once called, it will prepare a configuration for the given component from the corresponding template (adapting some values, e.g. replace DOMAIN with the actual domain), and send it back as JSON.

Each Pryv.io platform will have one service-config-leader (probably hosted on register machine).

#### Questions

- How do the server authenticate client components?
- Configuration values are stored in Redis?
- Schemas? Convict/Nconf?

### Service-config-follower

This service will be orchestrating the distribution of configuration files among the Pryv.io containers, thus each machine of a Pryv.io platform will have one such service. The main role of this service is to fetch configurations from service-config-leader and to place them in the configuration folder of each Pryv.io containers.

The service can be broken down into three parts:
1. A client pulling the centralized configurations from service-config-leader
2. A tool that sends reboot signals to the Pryv.io processes/containers (will be used by 1. and 3.)
3. A server that listen to configuration updates pushed by service-config-leader (implemented in a later iteration)

#### Questions

- Where will the client in core run?
- How will the client distribute config files to all containers?
- How will the client make the other containers boot?

## Changes

### Cluster

#### New services: service-config-leader and service-config-follower

Each Pryv.io machine (cores, reg-master, reg-slave, static) now hosts a follower configuration service, which comes with the following new files:

- ${PRYV_CONF_ROOT}/config-follower/conf/config-follower.json
  
- ${PRYV_CONF_ROOT}/config-follower/config-follower.yml

- run/stop-config-follower

On the other hand, a single leader configuration service is hosted on reg-master, it comes with the following new files:

- ${PRYV_CONF_ROOT}/config-leader/conf/config-leader.json
  
- ${PRYV_CONF_ROOT}/config-leader/config-leader.yml
  
- run/stop-config-leader

#### Centralized configuration in the leader

Configuration template files for the Pryv platform is now centralized within the leader. Thus, the data folder of the leader should contain these files for all followers (i.e. for each Pryv.io roles: core, reg-master, reg-slave, static) as follows:

- The configuration folders previously distributed among all Pryv.io machines in ${PRYV_CONF_ROOT}/${ROLE} are now centralized within the leader in ${PRYV_CONF_ROOT}/config-leader/data/${ROLE}

- The Dockerfiles previously distributed among all Pryv.io machines in ${PRYV_CONF_ROOT}/${ROLE}.yml are now centralized within the leader in ${PRYV_CONF_ROOT}/config-leader/data/${ROLE}/pryv.yml

- The SSL certificates previously distributed among all Pryv.io machines in ${PRYV_CONF_ROOT}/${ROLE}/nginx/conf/secret are now centralized within the leader in ${PRYV_CONF_ROOT}/config-leader/data/${ROLE}/nginx/conf/secret

- Platform-specific variables that will be replaced throughout the configuration template files can be set in ${PRYV_CONF_ROOT}/config-leader/conf/config-leader.json within the 'platorm' setting object

#### Followers fetch configuration files from the leader

Each follower will fetch the configuration that the leader prepared, for the Pryv.io machine its responsible for.
Once a follower gets the configuration files, it places them in:

  - ${PRYV_CONF_ROOT}/pryv/, previously was ${PRYV_CONF_ROOT}/${ROLE}/

#### Leader and followers communication

Leader and followers need to be able to communicate bidirectionally, so some changes to the DNS and Nginx configuration have been made:

- New DNS entry in ${PRYV_CONF_ROOT}/config-leader/data/reg-master/dns/conf/dns.json so that the leader is publicly reachable at lead.${DOMAIN}

- New Nginx server in ${PRYV_CONF_ROOT}/config-leader/data/reg-master/nginx/conf/site-443.conf so that calls targeting lead.${DOMAIN} are proxied to the leader container

- New Nginx server in ${PRYV_CONF_ROOT}/config-leader/data/${ROLE}/nginx/conf/site-443.conf that listens to /notify calls coming from the leader and proxies them to the follower container

- Each follower should be declared in ${PRYV_CONF_ROOT}/config-leader/conf/config-leader.json within the followers list, specifiying its symmetric key, role (core, reg-master, reg-slave or static) and url.

- Similarly, each follower configuration ${PRYV_CONF_ROOT}/config-follower/conf/config-follower.json should contain the symmetric key and the leader url.

#### Various

- Adapt ${ROLE}.logrotate.conf, ensure-permissions-${ROLE}, README.md, INSTALL.md

- Replace run-${ROLE} and stop-containers with run-pryv and stop-pryv scripts

- Changes to Dockerfiles (all pryv.yml):
  - Upgrade to version 3.5
  - Paths for Docker volumes are now shorter since the /${ROLE} root folder is not specified anymore
  - Networks (frontend, backend) and containers now have fixed names

### Single-node

#### New services: service-config-leader and service-config-follower

A singlenode Pryv.io machine now hosts one follower and one leader configuration service, they come with the following new files:

- ${PRYV_CONF_ROOT}/config-leader/conf/config-leader.json and ${PRYV_CONF_ROOT}/config-follower/conf/config-follower.json

- ${PRYV_CONF_ROOT}/config-leader/config-leader.yml and ${PRYV_CONF_ROOT}/config-follower/config-follower.yml

- run/stop-config-leader and run/stop-config-follower

#### Centralized configuration in the leader

Configuration template files for the Pryv platform is now centralized within the leader. Thus, the data folder of the leader should contain these files as follows:

- The configuration folders previously in ${PRYV_CONF_ROOT}/pryv are now centralized within the leader in ${PRYV_CONF_ROOT}/config-leader/data/singlenode

- The Dockerfile previously in ${PRYV_CONF_ROOT}/pryv.yml is now centralized within the leader in ${PRYV_CONF_ROOT}/config-leader/data/singlenode/pryv.yml

- The SSL certificates previously in ${PRYV_CONF_ROOT}/pryv/nginx/conf/secret are now centralized within the leader in ${PRYV_CONF_ROOT}/config-leader/data/singlenode/nginx/conf/secret

- Platform-specific variables that will be replaced throughout the configuration template files can be set in ${PRYV_CONF_ROOT}/config-leader/conf/config-leader.json within the 'platorm' setting object

#### Followers fetch configuration files from the leader

The follower will fetch the configuration that the leader prepared and places the files in:

  - ${PRYV_CONF_ROOT}/pryv/, in the same folder they previously were

#### Leader and followers communication

Even if the follower and the leader will communicate locally (container to container), the following changes are still necessary:

- New DNS entry in ${PRYV_CONF_ROOT}/config-leader/data/reg-master/dns/conf/dns.json so that the leader is publicly reachable at lead.${DOMAIN}

- New Nginx server in ${PRYV_CONF_ROOT}/config-leader/data/reg-master/nginx/conf/site-443.conf so that calls targeting lead.${DOMAIN} are proxied to the leader container

- The follower should be declared in ${PRYV_CONF_ROOT}/config-leader/conf/config-leader.json within the followers list, specifiying its symmetric key, role (singelnode) and url (local container).

- Similarly, each follower configuration ${PRYV_CONF_ROOT}/config-follower/conf/config-follower.json should contain the symmetric key and the leader url (local container).

#### Various

- Adapt ensure-permissions, README.md, INSTALL.md

- Replace stop-containers with stop-pryv script

- Changes to Dockerfiles (all pryv.yml):
	- Upgrade to version 3.5
	- Paths for Docker volumes are now shorter since the /pryv root folder is not specified anymore
	- Networks (frontend, backend) and containers now have fixed names

## Migration steps

Here are some instructions to follow in order to migrate to the new leader-follower configuration setup.

### Cluster

0) Update docker-compose:  

```
pip uninstall docker-compose
VERSION=1.24.1
DESTINATION=/usr/local/bin/docker-compose
curl -L https://github.com/docker/compose/releases/download/${VERSION}/docker-compose-$(uname -s)-$(uname -m) -o $DESTINATION
chmod 755 $DESTINATION
```

1) Stop running containers on all machines: ./stop-containers
2) Backup ${PRYV_CONF_ROOT}: mv ${PRYV_CONF_ROOT} ${PRYV_CONF_ROOT}-old
3) Create a fresh ${PRYV_CONF_ROOT} folder and untar the new configuration files on each machine
4) On leader machine, adapt/merge ${PRYV_CONF_ROOT}/config-leader/data/${ROLE}/${COMPONENT}/conf from old configurations of each machine ${PRYV_CONF_ROOT}-old/${ROLE}/${COMPONENT}/conf, same for  ${PRYV_CONF_ROOT}/config-leader/data/${ROLE}/pryv.yml on leader from ${PRYV_CONF_ROOT}-old/${ROLE}.yml of each machine
5) Setup leader and followers (keys, urls), start them: ./run-config-leader, ./run-config-follower
6) On each machine, restore old data/log folders by moving content of ${PRYV_CONF_ROOT}-old/${ROLE}/${COMPONENT}/data and ${PRYV_CONF_ROOT}-old/${ROLE}/${COMPONENT}/log in ${PRYV_CONF_ROOT}/pryv/${COMPONENT}/data and ${PRYV_CONF_ROOT}/pryv/${COMPONENT}/log
7) Run pryv: ./run-pryv

### Singlenode

0) Update docker-compose:  

```
pip uninstall docker-compose
VERSION=1.24.1
DESTINATION=/usr/local/bin/docker-compose
curl -L https://github.com/docker/compose/releases/download/${VERSION}/docker-compose-$(uname -s)-$(uname -m) -o $DESTINATION
chmod 755 $DESTINATION
```

1) Stop running containers: ./stop-containers
2) Backup ${PRYV_CONF_ROOT}: mv ${PRYV_CONF_ROOT} ${PRYV_CONF_ROOT}-old
3) Create a fresh ${PRYV_CONF_ROOT} folder and untar the new configuration files on each machine
4) Adapt/merge ${PRYV_CONF_ROOT}/config-leader/data/singlenode/${COMPONENT}/conf from old configurations ${PRYV_CONF_ROOT}-old/pryv/${COMPONENT}/conf, same for  ${PRYV_CONF_ROOT}/config-leader/data/singlenode/pryv.yml from ${PRYV_CONF_ROOT}-old/pryv.yml
5) Setup leader and follower (keys, urls), start them: ./run-config-leader, ./run-config-follower
6) Restore old data/log folders by moving content of ${PRYV_CONF_ROOT}-old/pryv/${COMPONENT}/data and ${PRYV_CONF_ROOT}-old/pryv/${COMPONENT}/log in ${PRYV_CONF_ROOT}/pryv/${COMPONENT}/data and ${PRYV_CONF_ROOT}/pryv/${COMPONENT}/log
7) Run pryv: ./run-pryv
