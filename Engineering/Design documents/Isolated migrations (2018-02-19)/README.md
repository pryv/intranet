|              |                       |
| ------------ | --------------------- |
| Author       | Thiébaud Modoux (Pryv) |
| Version      | 1 (19.02.2018)         |
| Distribution | Internally            |

# Old situation

Api-server would migrate the database on startup. Since this was in the same process that the server was launched with and since we now have several of these processes, we have several concurrent migrations, which is bad/not working.

# Adopted solution

Through [PR #113](https://github.com/pryv/service-core/pull/113), we decided to extract the migration code into a ['bin/migrate'](https://github.com/pryv/service-core/blob/ee465f3b69ef7e880c1f8e47380d2619bdbb9186/components/api-server/bin/migrate) script that only migrates and does nothing else.

The docker image now [runs this script through **runit**](https://github.com/pryv/service-core/blob/ee465f3b69ef7e880c1f8e47380d2619bdbb9186/build/core/runit/core#L10) and only then start the server processes.

# Remaining questions

In api-server, we removed [the waitForConnection call on storage](https://github.com/pryv/service-core/pull/113/files#diff-0f84471e50820d9ed71c9284e0e2ea1fL68) from server code and put it in the migration script. Should we still wait for the storage connection also in server code? Is it really necessary (mongodb should wait for a connection in any case) ? Could we remove the call to waitForConnection everywhere (also in migrate script) ?

Let's see how the current implementation behaves and remembers this questions if anything goes wrong. 