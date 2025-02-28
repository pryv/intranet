
# Pryv.io platform config migration tool

## Motivation

Currently, an upgrade may introduce changes in the `platform.yml` config file, which is hard to perform by our customers. As it is done by hand.
The goal is to automate this part.

## Initial setup

Config leader should initiate a git environment in the config folder to track changes to it.

Upon first deployment, one should run a script that would perform a `git init`, the script would need to be isomorphic.

After running it, it performs a first commit: `initial commit`, tagging it with the template version.

## Flow

Instead of step 2. of backing up the old `platform.yml` file, [link](https://github.com/pryv/config-template-pryv.io/blob/master/pryv.io/single-node/UPDATE.md), the file provided in the config would have a different name such as `template-platform.yml`, which will remove the risk of overwriting your old one when unpacking, same for `config-leader.json`.

Running the init script mentionned in `Initial setup` would make a copy of `template-platform.yml` without the prefix `template-`.

An available update will be found if `platform.yml` and its template have different `TEMPLATE_VERSION`.

When booting, the new config leader version, you would be prompted to connect to the admin panel, when you will be proposed to update to the new format. Until the config migration is done, config-leader prints in its logs that you need to perform the migration and calls on `GET /conf` from followers return 400 errors.

At connection, after signing into the interface successfully and obtaining a valid token, if the user has settings modification permissions, he will be asked if he wishes to upgrade the platform.yml.

If he wishes so, an upgrade will be performed, taking values from `platform.yml` and applying them into a copy of the new `template-platform.yml`.

After applying the update, all changes to the `config-leader/conf/` folder would be committed to git.

## Design

### API

1. GET /admin/migrations: check if an update is available
2. POST /admin/migrations/apply: apply update
3. GET /admin/config: download the `platform.yml`

After that, we can apply the new `platform.yml` parameters using the existing `POST /admin/notify` route

### Config migration

The feature would need to be able to operate multiple config migrations at once. We will implement a first one from 1.6.21 to 1.7.0. Then as customers request to upgrade to 1.7, we will implement the various steps.

#### single-node VS cluster

As the configuration differs between the two, the `config-leader` service will need to figure out what upgrades to apply, this can be figured from the presence of the `vars:MACHINES_AND_PLATFORM_SETTINGS:settings:SINGLE_MACHINE_IP_ADDRESS` value in the `platform.yml` file.

#### Algorithm

1. read starting version
2. read destination version
3. ensure that there exists a path of upgrades between 1 & 2
4. apply the array of upgrade functions

## Discussion

- Can we audit the config structure?
> Yes, we can compare key by key all entries, we have the template version

- use git for versioning

- versioning with git