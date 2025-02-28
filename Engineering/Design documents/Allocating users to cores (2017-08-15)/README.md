# Dynamic allocation of users on Cores by Register

This document introduces specifications to handle the allocation of users on cores for Pryv installations with multiple cores.
Then it presents the current implementation and proposes a basic and an advanced solutions to reach the specifications.

This is version 1 (15.08.2017) of the document and the author is Thiébaud Modoux.

## Specifications

In cases where a Pryv installation is distributed on multiple cores, we would like Register to allocate new users on cores for a single location **fairly**.
By fairly, we have these properties in mind:

* A new user must be created among the N (>> 1, << 1000) cores that have the least users.
* Any one of the eligible cores (see above) can be taken, the strategy for choosing can be random.

This goal can be reached by following the following steps:

* Retrieve all available cores for specified location
* Retrieve cores size (by users count) and sort them by increasing order
* Choose one core randomly among the N (>> 1, << 1000) emptiest cores

Related links: [Podio](https://podio.com/pryvcom/py/apps/projects/items/167), [Trello](https://trello.com/c/10nMOrLX/71-register-allocate-cores-to-users)

## Current implementation

The [current strategy](https://github.com/pryv/service-register/blob/1810ba0e646fe4ee07ef347cf0efc6a644573cd8/source/utils/dataservers.js#L51-L53) for allocating new users on cores chooses randomly between all available cores (specified in a config file).

Thus, the desired changes will affect mainly the **getHostForHosting()** function in the following file: _getsource/utils/dataservers.js_.

## Basic solution

The first implementation could take benefit of an existing function in Register, **[getServers](https://github.com/pryv/service-register/blob/1810ba0e646fe4ee07ef347cf0efc6a644573cd8/source/storage/users.js#L92)**, that generates a list of cores alongside with the users count.

Thus, we can call this function at user creation to compute the users count for all cores and then choose randomly among the emptiest ones.

Despite the fact that this solution causes the fewest changes in existing code, it suffers from the following disadvantages:

* We have to deal with two separated realities:
  * The list of all available cores, stored in the config file ([net:aaservers](https://github.com/pryv/service-register/blob/master/dev-config.json#L62))
  * The list of all non-empty cores alongside with users count, computed by going through all _user:server_ pairs in Redis, using the **getServers** function
* The users count computation will run for each cores every time a new user is created

## Advanced solution

We present here an improved solution to overcome the disadvantages of the basic solution that we presented previously.

We would like to go through the following steps only once (probably at Register start-up):

* Read the list of available cores in the config file
* Copy this list in Redis and initialize a users count to 0 for each cores
* Increase/decrease the users count at user creation/deletion
* At user creation, simply retrieves the user count for each cores and choose the candidate core randomly among the emptiest

In order to centralized all computations in Redis, we could take benefit from the following [ZSET Redis commands](https://redis.io/commands):

* ZINCRBY command to update users count
* ZSCORE/ZRANGEBYSCORE to retrieve the users count for each cores

In a last step, we would like to add the following optional properties to core entries:

* A maximum number of users that each core can tolerate, we will not allocate new users on cores that reached this limit
* An availability flag that allows to deactivate a core, we will not allocate new users on deactivated cores

# Questions

1. How does the feature work for the server administrator? Adding servers? Removing servers? Code needs to take this into account. 