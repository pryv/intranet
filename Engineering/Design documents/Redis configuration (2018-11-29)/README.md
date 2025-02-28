|         |                       |
| ------- | --------------------- |
| Author  | Thiébaud Modoux |
| Date    | 20.11.2018           |
| Version | 1                    |

# Configuration for Redis 5.0.2

While updating Redis to version 5.0.2, we also reconsidered and updated the default redis configuration, based on [the last stable configuration file example](http://download.redis.io/redis-stable/redis.conf).

Through this article, we would like to summarize and explain the choices we made wile preparing [our new Redis default configuration](https://github.com/pryv/release-packaging/blob/master/build/redis/config/redis.conf). If a configuration entry is not specified here, it means that we chose to rely on the previous configuration or to use the recommended value for this entry.

## General

### NETWORK

#### bind
We do not specify any "bind" directive, so Redis is listening for connections from all the network interfaces available. 

#### protected-mode

This is a layer of security protection, in order to avoid that Redis instances left open on the internet are accessed and exploited. When set to 'yes', the server only accepts connection from clients from loopback addresses (127.0.0.1 and ::1) and from Unix domain sockets.

We consciously set **'protected-mode no'** because we want to shift this concern to Docker.

## Persistence

See [this interesting article](https://redis.io/topics/persistence) as an introduction to Redis persistence.

### SNAPSHOTTING

Snapshotting is a mechanism that will occasionally dump the DB on disk.
These DB dumps are called RDB snapshots and are saved as _redis_dump.rdb_.

#### save \<seconds> \<changes>

Save the DB on disk if both the given number of seconds and the given number of write operations against the DB occured.

Our configuration specifies the following:
- Save after **900 sec** (15min) if at least **1** key changed
- Save after **300 sec** (5min) if at least **10** keys changed
- Save after **60 sec** (1min) if at least **10'000** keys changed

#### stop-writes-on-bgsave-error

We set **'stop-writes-on-bgsave-error yes'** so that Redis will stop accepting writes if the latest background save (RDB snapshot) failed. This will make the user aware (in a hard way) that data is not persisting on disk properly, otherwise chances are that no one will notice.

If the background saving process will start working again Redis will automatically allow writes again.

With proper monitoring of the Redis server and persistence, we may want to disable this feature so that Redis will continue to work even if there are issues with disk, permissions, and so forth.

### APPEND ONLY MODE

#### appendonly

With RDB snapshots, an issue with the Redis process or a power outage may result into a few minutes of writes lost.

This is why we also enable the Append Only File (AOF) by setting **'appendonly yes'**, which provides much better durability in face of a crash, losing far less writes. The file in which the write log will be written is defined as **'appendfilename "appendonly.aof"'**.

#### appendfsync

We kept the default value **'appendfsync everysec'**, which tells the OS to fsync(), i.e. actually flush to the append only log, one time every second. This should be the right compromise between speed (let the OS flush when it wants) and data safety (flush every write).
