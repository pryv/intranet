|           |                      |
| --------- | -------------------- |
| Authors   | Simon Goumaz         |
| Reviewers | Pierre-Mikaël Legris |
| Date      | 2022-03              |
| Version   | ?                    |

# ==WIP== Friendlier install, update & maintenance

## Current situation

### Issues

Development:
- Config is mostly duplicated with `singlenode` and `cluster` setups, so most of the work gets done twice
- Publishing and downloading releases is tedious (download, decompress, etc.)

Installation & update:
- Instructions are inconveniently dispatched across several documents
- Following the step-by-step instructions is error-prone and generally not a smooth, consistent experience
- We expose the internals of config sync (leader, follower)

Maintenance:
- Centralized config setup should not rely on the public SSL certificate (e.g. it all fails when the cert expires)


### Structure and behavior

Existing structure:
- config: `service}/conf/{service}.json`
- others: `{service}/{other}/…/{file}` (e.g. `/mail/templates/reset-password/en/html.pug`)
- data (not touched): `{service}/data/…`

Existing test cases (to be covered/removed):
- config-leader:
	- …
- config-follower:
	- writes files to disk, ignoring _data_ files
	- restarts specified services on `/notify`

Older notes on config leader/follower:
- `leader/platform.yml` obsolete? why is it there? master is `template/…/template-platform.yml` right?
- `leader/dev-config.json` put in `conf` folder or something?

### Available design history

- [Install simplification (2018-07-12)](../Install%20simplification%20(2018-07-12)/README)


## Solutions

### Goals

Installing must look like:
1. Install 'Pryv.io platform node' package on one machine and run it with `--init-platform` (returns connection info)
2. Setup base platform settings locally
3. Install the same package on possible other machines and run it with `--join-platform={init machine connection info}`
4. Open platform admin app and set further settings (or possibly do it from the command line)

When publishing a new release:
- The release (i.e. the new service images versions and up-to-date configs) as well as possible config migrations must be in a single repo (no more monkeying around between config-leader and config-template)
- Corollary: the only time we should update services/apps in charge of the update process (currently config-leader & config-follower) is when the update process itself changes

Configuration:
- All settings are platform-level (as currently), but some settings can be overriden at the machine-level (but both platform- and machine-level settings are managed via the same UI e.g. platform admin app)
    - Example use case of hospitals having their own cores and wanting different audit settings

### Ideas & questions

==TODO review & sort==

- Q: if we use a distributed DB like etcd for config and platform-level data (users…), how to deal with the limitation on the number of writer nodes?
    - Just limit the number of nodes in a platform (could be fine as a first step judging from the currently deployed platforms)
    - Have a limited number of full (writing) nodes and an unlimited number of read-only (observer replica) nodes, with logic for the latter to dispatch writes to the "nearest" full node
- Remove dedicated reg/DNS machines: all nodes play the same roles
	- ==Q: how to deal with DNS round-robin/load balancing to avoid one node taking most of the DNS traffic==
	- ==Q: except DNS?==
	- ==Q: need to evaluate impact on machine resources==
- Evolve config leader/follower to a single `service-config` (or `service-node`, …) doing watch & sync
	- ==Q: worth making it a publishing service the other services subscribe to, thus deciding themselves what to do when configuration changes (update settings, reboot…)? (or just stick to basic "reboot on any change")==
	- See nconf for possibilities of distributed conf store with write & change notifications (e.g. etcd); conf sources then:
		- read-only: test, argv, env
		- base (YAML)
		- extras: distributed store
- Build a CLI to install, update & manage the platform, replacing the scripts, ssl-renew-certificate etc. ([example of distributing a CLI as a Docker image](https://medium.com/oracledevs/creating-an-oracle-cloud-infrastructure-cli-toolkit-docker-image-35be0ca71aa))
	- Interacts with Docker tools for start/restart/stop and maybe other common tasks like status, logs etc.
	-
- Quickfixes for config leader/follower:
    - Fix "default" setup (2 cores, 2 regs…): make it configurable (preferred) or just change it to what's actually the most common setup (1 core…)
	- Automatically set all internal auth keys in the `init...` scripts (review all `REPLACE_ME` values)
	- Use a dedicated self-issued, internal SSL, not the public one used by the platform
	-



### Related existing tools

- Distributed config DB ⚠️ most if not all DBs (at least all Raft-based ones) recommend a maximum number of nodes, for example [7 for etcd](https://etcd.io/docs/v3.5/faq/#what-is-maximum-cluster-size) (potential issue with big distributed platforms)
	- [etcd](https://etcd.io/docs/v3.5/install/) ([npm: etcd3](https://www.npmjs.com/package/etcd3)): looks best if we only need (namespaced) key-value
    - [rqlite: The lightweight, distributed relational database built on SQLite](https://github.com/rqlite/rqlite/) – standalone DB engine (unlike Dqlite below)
	- [Dqlite](https://dqlite.io/docs), [canonical/dqlite (C lib)](https://github.com/canonical/dqlite) – difference with rqlite: in-process
	- [losfair/mvsqlite: Distributed, MVCC SQLite that runs on FoundationDB.](https://github.com/losfair/mvsqlite)
	- Cluster management solutions considered but rejected (overkill within Pryv.io: leave that option to our customers): Nomad, MicroK8s, K8s
	- [rqlite](https://github.com/rqlite/rqlite) distributed sqlite
- Programmatic docker-compose: [PDMLab/docker-compose: Manage Docker-Compose via Node.js](https://github.com/PDMLab/docker-compose), [bollard - Rust](https://docs.rs/bollard/latest/bollard/),
- Unified build/CI/deployment:
	- [Dagger](https://dagger.io/)
	- [Nix](https://nixos.org/)
		- [NixOS Guide: Declarative and reproducible developer environments](https://nixos.org/guides/declarative-and-reproducible-developer-environments.html)
		- [repository template to get you started with Nix.](https://github.com/nix-dot-dev/getting-started-nix-template)


## Past related work

- [Centralized config (2019-07)](../Centralized%20config%20(2019-07)/README)
- [Service orchestration (2017-11-07)](../Service%20orchestration%20(2017-11-07)/README)
