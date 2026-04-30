# Memtier Stress Test Result

## Environment
- Repository: `keydb-source-code`
- Branch: `main`
- OS: Ubuntu 24.04 container
- CPU: 2 cores
- RAM: ~8 GB
- Built binaries:
  - `src/keydb-server`
  - `/tmp/memtier_build/memtier_benchmark/memtier_benchmark`

## Server setup
- KeyDB server launched with:
  - `./src/keydb-server --port 6379 --save "" --appendonly no --dir /tmp --databases 16 --maxclients 1000 --loglevel warning`

## Benchmark setup
- Memtier benchmark command:
  - `/tmp/memtier_build/memtier_benchmark/memtier_benchmark --server=127.0.0.1 --port=6379 --protocol=redis --threads=4 --clients=50 --pipeline=10 --ratio=1:1 --key-maximum=10000 --key-minimum=1 --data-size=128 --requests=100000 --randomize --hide-histogram --out-file /tmp/memtier_result.txt --json-out-file /tmp/memtier_result.json`

## Summary results
- Total operations: `20,000,000`
- Throughput: `291,838.22 ops/sec`
- Average latency: `6.835 ms`
- p50 latency: `6.207 ms`
- p99 latency: `22.399 ms`
- p99.9 latency: `36.095 ms`
- Bandwidth: `44,981.54 KB/sec`
- Connection errors: `0`

## Notes
- KeyDB server was stopped after the benchmark run.
- Raw output files generated during the test:
  - `/tmp/memtier_result.txt`
  - `/tmp/memtier_result.json`
- Server log:
  - `/tmp/keydb-server.log`

## Multi-master 4-node replication test

### Cluster setup
- 4 KeyDB instances on ports `6380`, `6381`, `6382`, and `6383`
- Each instance started with:
  - `./src/keydb-server --port <port> --dir /tmp/keydb_mm/<node> --save "" --appendonly no --databases 16 --loglevel warning --active-replica yes --multi-master yes --server-threads 1`
- Ring replication configured:
  - `6380` replicaof `127.0.0.1 6383`
  - `6381` replicaof `127.0.0.1 6380`
  - `6382` replicaof `127.0.0.1 6381`
  - `6383` replicaof `127.0.0.1 6382`
- Verified all nodes reported `role: active-replica` and `master_link_status: up`.

### Benchmark setup
- Memtier benchmark command:
  - `/tmp/memtier_build/memtier_benchmark/memtier_benchmark --server=127.0.0.1 --port=6380 --protocol=redis --threads=4 --clients=50 --pipeline=10 --ratio=1:1 --key-maximum=10000 --key-minimum=1 --data-size=128 --requests=100000 --randomize --hide-histogram --out-file /tmp/memtier_mm_result.txt --json-out-file /tmp/memtier_mm_result.json`

### Summary results
- Total operations: `20,000,000`
- Throughput: `63,112.05 ops/sec`
- Average latency: `31.000 ms`
- Bandwidth: `9,717.77 KB/sec`
- Hits/sec: `25,421.45`
- Misses/sec: `6,134.57`
- Connection errors: `0`

### Notes
- Raw output file from the multi-master test:
  - `/tmp/memtier_mm_result.json`
- Cluster server logs:
  - `/tmp/keydb_mm/0.log`
  - `/tmp/keydb_mm/1.log`
  - `/tmp/keydb_mm/2.log`
  - `/tmp/keydb_mm/3.log`
- The 4-node multi-master instances were stopped after the benchmark.
