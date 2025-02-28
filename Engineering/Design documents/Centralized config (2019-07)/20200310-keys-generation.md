#Follower-Leader keys generation

Currently, follower-leader keys are generated upon the [build](https://github.com/pryv/config-template-pryv.io/blob/central/pryv.io/cluster/scripts/build).

We thus have key pairs in:

- LEADER/config-leader/conf/config-leader.json
- FOLLOWER/config-follower/conf/config-follower.json

**These files should be backed up in the update process.**

As we wish to be able to add followers dynamically, we need





# Internals key generation

We have [internals keys](https://github.com/pryv/config-template-pryv.io/blob/central/pryv.io/cluster/config-leader/conf/config-leader.json#L18) that we don't wish our customers to set as they are system keys used by our services to authentify calls between them.

These are set to `SECRET` when delivered to customers. Upon each leader boot, they are replaced by new cryptographic keys, therefore if after a boot, only a register is updated (without the cores), some of their keys won't match, thus breaking Pryv.io functionality.

## Proposed solution

1. Generate strong keys @ build - done
2. Tell our customers to backup this file (additionally to `platform.yml`) before updating - done
3. Remove keys generation from leader - TBD



