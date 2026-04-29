# Testing KeyDB

## Basic Test Commands

- Run one unit test: `./runtest --single unit/replication`
- Run one integration test: `./runtest --single integration/replication-2`
- Run TLS tests: `./utils/gen-test-certs.sh && ./runtest --tls`

## Docker on Windows: Replication Test Caveat

When running Linux containers with the repository bind-mounted from a Windows
filesystem (for example `-v D:/RajaS/KeyDB:/work`), replication tests can fail
with errors like:

- `Failed trying to load the MASTER synchronization DB from disk`

This is often a filesystem/runtime artifact from the bind mount, not
necessarily a replication regression.

Use container-local storage for reliable replication results:

```bash
docker run --rm -v D:/RajaS/KeyDB:/src ubuntu:24.04 bash -lc "apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y build-essential tcl pkg-config uuid-dev libssl-dev libcurl4-openssl-dev zlib1g-dev git libcurl4 && rm -rf /tmp/keydb && cp -a /src /tmp/keydb && cd /tmp/keydb && git submodule update --init --recursive && make -j4 && ./runtest --single integration/replication-2"
```
