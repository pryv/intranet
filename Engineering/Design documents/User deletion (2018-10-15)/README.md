|           |                                               |
| --------- | --------------------------------------------- |
| Author    | Kaspar Schiess (kaspar@pryv.com)              |
| Reviewers | Pierre-Mikael Legris, Ilia Kebets (Version 1) |
| Version   | 3 (20181029)                                  |



Allow Deleting a User
{: .doc_title }

Extension of Pryv.IO
{: .doc_subtitle }



# Summary

This document specifies how we intend to extend and fix the 'delete user' functionality in the existing 'pryv-cli' application. In particular, we propose to extend the existing code with code that also deletes the user on 'registry' - and finally also deletes high frequency data, if there is such data. 

The goal is to have a valid - if a bit cumbersome - way to delete users that we can teach to our clients. 



| Version | Changes                                                   |
| ------- | --------------------------------------------------------- |
| 1       | Draft.                                                    |
| 2       | Includes changes from revision by Ilia and Pierre-Mikael. |
| 3       | Adds a bit about the user deletion route in registry.     |



------



# Overview

We create a 'delete-user' tool that you can launch as follows from the command line on a 'core', provided you change to the configuration root for Pryv.IO first: 

```shell
# This makes the next bit look nice; it also explains where the script comes from.
$ alias delete-user="docker run -v .:/app/conf/:ro --network foo_backend -ti pryv/cli:latest $*"

# Actual deletion. 
$ delete-user fhuber
```

After running this to completion, these points are true: 

* User data has been completely wiped from MongoDB. 
* User data has been completely wiped from InfluxDB. 
* User data has been wiped from registry. 
* User data has been wiped from the file system. 

# Design

Here's the current functionality of the pryv CLI application: 

1. Find the user locally
2. Let the user confirm the username by typing it in.
3. Find attachments
4. Remove user from users collection
5. Remove other collections (is this complete?)
6. Remove all files
7. Say it worked.

The application currently doesn't do anything else but delete users. 

We propose to **add this missing functionality**: 

* A check checking all the connections to all subsystems. This allows to not start deleting unless deletion will succeed. 
* Is the collection list up to date? Verify if the existing delete is still correct in light of new functionality in Pryv.IO core. 
* Delete data from InfluxDB if available and configured. 
* Delete data from registry after everything has worked. 
* Allow remote control. Our clients will want to include the small cli utility into a bigger 'user deletion' process. To this aim, it would be nice to create a non-interactive mode of the tool. 

The subsequent sections will provide more detail on these points above.

## Configuration (and How to get at it)

To configure the deletion properly, we need a useful answer to the following dilemma: 

* Configuration is distributed across hfs.json and core.json, both in separate directories. 
* The names that are used there ('mongodb', 'influxdb') do not correspond to docker container names (which will be prefixed with the installation name, often derived from the directory the config is in).
* To access the systems, we have two choices: 
  * a) access from the host system via internal network. To do so, we would need to figure out the IP address of the target machine on the internal network, probably via 'docker inspect container_name'. This would - again - necessitate knowledge of the container names. 
  * b) Access from inside a docker container, linked to the target containers. This time, the names in the configuration would work ('mongodb', 'influxdb') - but in order to link up a cli container after the fact with these other containers, we'd have to use the container names anyway. 
* So either way: The user of the script needs to know which containers are affected in order to be able to issue the correct deletion command. If he gets this wrong - and links / points to another MongoDB database having no relation to the Pryv IO install - he will possibly delete data that wasn't supposed to be deleted. 

We will derive a solution from this: 

```shell
$ docker network ls
NETWORK ID          NAME                       DRIVER              SCOPE
cc4c05929362        bridge                     bridge              local
c914af270b5e        host                       host                local
ea9db615d9ec        none                       null                local
75ad091847e1        previewpryvtech_backend    bridge              local
682f17531957        previewpryvtech_frontend   bridge              local

$ docker run  --network previewpryvtech_backend -ti ubuntu /bin/bash
$ apt update && apt-get install netcat
$ nc -v -v mongodb 27017
$ nc -v -v influxdb 8086
# (both work)
```

The above snippet connects the container (ubuntu:latest in this case) to the backend network in my sample installation. Once the container is connected to the correct user defined bridge network ([docker documentation](https://docs.docker.com/network/bridge/#manage-a-user-defined-bridge)), it can access 'mongodb' and 'influxdb' via docker internal DNS resolution. This means that we need to run our CLI application in the following environment: 

* Separate container, called 'pryv/cli'. This is new. 
* Container needs to be attached to the 'backend' network of the installation; if there is more than one, the admin needs to choose here. 
* Container needs access to the configuration files of 'core' and 'hfs' - we suggest to achieve this by mapping `-v .:/app/conf/:ro'. This means we map all configuration files into the pryv/cli container. 

In our code, we will implement some smarts to find the various configuration files (paths might vary a bit) and to read them using the original configuration classes, implementing the same defaults. This will guarantee that our connection strings are good, **provided the connected network is the correct network**. 

If the admin messes up and really has two pryv instances on the same machine, he might delete the wrong user's data. However, during preflight we check if the user really exists. So to delete the wrong data, the admin needs to mess up and be unlucky - the user needs to be present in the wrong system as well. We accept this risk. 

## Preflight Check

Before starting the deletion, we will check all remote systems; if we can access them and have the correct access permissions, we will allow deletion to start. Note that it might be hard in practice to verify the permissions; in that case, we will reduce the check for simplicities sake. 

## Deleting Data from the Registry

To delete data from the registry, the tool will obtain the registry for the installation via the configuration file for 'core'. It will then issue a DELETE request to the route '/users/:username'. If this route returns 2XX status code, the user will have been permanently deleted from the registers registry. 

We will protect this route by requiring the 'system' role. 

Conventions in the 'users' route file are a bit mixed; but we're going to use the user creation as a guideline. We will create the following API routes:

```
DELETE /users/:username

Deletes the user given by `username`. If you want to delete the user 
only from 'registry', pass in 'onlyReg=true'. To perform a dry run 
without deleting anything, pass in 'dryRun=true'. 
```

For now, we will not implement the 'onlyReg=false' deletion that would delete all user data by just calling DELETE on registry. We include the parameter now to be able to easily add it in the future, should we wish for it. Setting it to false or not sending it along will provoke an error for now. 

## Remote Control

To allow remotely calling the script from another script, we will provide a non-interactive mode of the CLI tool that deletes users without asking back. The documentation will say: 

```shell
 --no-interaction   Disables user interaction; namely will not ask back before deleting 
                    a user. This is equivalent to answering 'yes' everywhere where the
                    script asks 'are you sure?'.
```



# Tests / Quality Control

## Automated

### Registry

Call to DELETE /users/:username: 

* Succeeds if the username exists. 
* Fails if the username is missing already. 
* Needs 'system' role
* Fails otherwise

## Manual

* If one of the items in preflight check fail then the deletion is not started. To produce this, stop one of the relevant containers and attempt to delete a user. 
* If '--no-interaction' is given, deletion starts right away without asking for confirmation. 
* Data is deleted from registry. To produce, create a user, delete it and verify it is gone from the redis database. 
* Data is deleted from InfluxDB. To produce, create some data in influxDB, delete user and then verify the data is gone. 
* Data is gone from MongoDB. Create a user and then delete it, no collections should remain in MongoDB. 





# Plan

We intend to create/modify the following artefacts:

* delete-user command in Pryv.IO CLI:
  * Also delete from InfluxDB: 4h
  * Also delete from Registry: 2h
* Retarget the Pryv.IO CLI to the new environment: 
  * Add smarts to find configuration files: 2h
  * Add config file reading: 3h
  * Add preflight check on all subsystems: 2h
  * Allow no-interaction use: 1h
* Registry: 
  * Add route to allow deletion of users with the system role: 4h



This first imprecise effort estimation yields 18h of work; we need to allocate about a week of uninterrupted work to get to review with this feature and about 3 days for the review and release. This means we can promise the feature to our customer on the 29.10.2018 at the earliest. Delays are to be expected because we concurrently do client installation work. 

