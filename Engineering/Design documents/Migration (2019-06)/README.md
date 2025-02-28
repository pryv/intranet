# Migration of Dockerized Pryv.IO

These procedures describe the steps to migrate a **Pryv.IO** platform.

Cores should be migrated first followed by the Register servers. 

Since URL endpoints on which apps are based depend on DNS resolving, the source machines will be updated to proxy to the destination ones for the duration it takes for DNS caches to be updated.

1. [Core migration](./migrate-core.md)
2. [Register migration](./migrate-register.md)
