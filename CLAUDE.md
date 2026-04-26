# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This is the Redis server (C codebase). Five binaries are produced from `src/` and they all share `redis-server` as the actual ELF — `redis-sentinel`, `redis-check-rdb`, and `redis-check-aof` are installed as copies/symlinks that change behavior based on `argv[0]`. `redis-cli` and `redis-benchmark` are independent binaries.

Bundled dependencies live under `deps/` (`jemalloc`, `lua`, `hiredis`, `linenoise`, `hdr_histogram`, `fpconv`, `xxhash`). Top-level `modules/` holds the bundled extension modules shipped with Redis 8+ (`redisbloom`, `redisearch`, `redisjson`, `redistimeseries`, `vector-sets`); these are independent of `src/modules/` (which only contains the `hello*.c` example modules used by the moduleapi tests). The system needs `libc`, a C compiler, and (for tests) `tclsh` 8.5 / 8.6 / 8.7 / 9.0.

## Build

```sh
make                       # default build (jemalloc on Linux, libc malloc elsewhere)
make BUILD_TLS=yes         # statically link OpenSSL into redis-server (./runtest --tls)
make BUILD_TLS=module      # build redis-tls.so as a runtime-loadable module
make USE_SYSTEMD=yes       # link libsystemd
make MALLOC=libc           # libc | jemalloc | tcmalloc | tcmalloc_minimal
make SANITIZER=address     # address | undefined | thread sanitizer build
make 32bit                 # 32-bit build (sets SKIP_VEC_SETS=yes; needs libc6-dev-i386)
make noopt                 # -O0, useful for debugging
make valgrind              # -O0 + libc malloc, suitable for valgrind
make gcov / make lcov      # coverage build / HTML report
make V=1                   # verbose (non-colorized) build output
make REDIS_CFLAGS='-Werror' # CI uses this; warnings fail the build
```

`make distclean` is required after changing build options (32bit ↔ 64bit, MALLOC, SANITIZER, BUILD_TLS, etc.) or after pulling changes that touch `deps/`. Plain `make` does **not** rebuild dependencies on its own. Build flags are cached in `src/.make-settings` until `make distclean` clears them.

`make install` (optionally `PREFIX=...`) installs the five binaries to `$PREFIX/bin`.

## Tests

The Redis test suite is written in Tcl and driven by `tests/test_helper.tcl`. The four entry-point shell scripts at the repo root each select a different driver, and there are matching `make` targets that build the prerequisites first:

```sh
./runtest        / make test           # main suite (unit/, unit/type/, unit/cluster/, integration/)
./runtest-cluster                      # cluster tests (tests/cluster/run.tcl)
./runtest-sentinel / make test-sentinel
./runtest-moduleapi / make test-modules # builds tests/modules/ first
make test-cluster                      # = ./runtest-cluster, after building deps
```

`tests/test_helper.tcl` no longer hard-codes `::all_tests`: it auto-globs every `*.tcl` under `unit/`, `unit/type/`, `unit/moduleapi/`, `unit/cluster/`, and `integration/`. Dropping a new file in one of those directories is enough — no list to update. The moduleapi suite is the exception: `runtest-moduleapi` lists each unit explicitly with `--single`, so add new module tests there.

Useful `runtest` flags (forwarded to `test_helper.tcl`):

```sh
./runtest --single unit/type/list      # run one unit
./runtest --only <test-name>           # run one named test (repeatable)
./runtest --skipunit unit/expire       # skip a unit
./runtest --list-tests                 # list all available units
./runtest --tls                        # run against TLS (needs make BUILD_TLS=yes
                                       # and ./utils/gen-test-certs.sh first)
./runtest --tls-module                 # same but with BUILD_TLS=module
./runtest --valgrind / --tsan          # run under valgrind / ThreadSanitizer
./runtest --cluster-mode               # run main suite against a cluster topology
./runtest --singledb                   # run against a single-db server
./runtest --force-resp3                # force RESP3 protocol
./runtest --log-req-res                # capture req/res for schema linter
./runtest --debug-defrag               # extra defrag assertions
./runtest --clients <n>                # parallelism (default 16)
./runtest --stop                       # halt on first failure
./runtest --tags '-slow'               # filter by tag (CI uses this)
./runtest --verbose / --quiet / --dump-logs
```

## Architecture

`README.md` contains an extended internals tour — read it before making non-trivial changes. The high-level shape:

- **Entry point**: `src/server.c` — `main()`, `initServer()`, the `aeMain()` event loop, `serverCron()` (periodic tasks), `beforeSleep()` (per-loop hook), `call()` (executes a command in a client context). The global `redisCommandTable` is declared `extern` in `server.c` and **defined in `src/commands.def`**, which is generated from the JSON files under `src/commands/` (one JSON per command) by `utils/generate-command-code.py`. To add or edit a command, write/modify the JSON, regenerate `commands.def`, and implement the C handler.
- **Global state**: the `struct redisServer server` global in `src/server.h` holds databases (`server.db`), the command table, the client list, replication state, and the I/O-thread machinery. `struct client` holds per-connection state (`querybuf`, `argc`/`argv`, output `reply`/`buf`, `db`).
- **Commands**: each command is still a `void fooCommand(client *c)` function that reads `c->argv` and calls `addReply*()` to respond. Generic key-level commands (DEL, EXPIRE, …) live in `db.c`; type-specific commands live in `t_string.c`, `t_list.c`, `t_set.c`, `t_zset.c`, `t_hash.c`, `t_stream.c`. `db.c` also exposes `lookupKeyRead/Write`, `dbAdd`, `setKey`, `dbDelete`, `emptyDb` — use these rather than poking the kvstore directly.
- **Networking**: `networking.c` — `createClient`, `readQueryFromClient` (reader), `processInputBuffer` → `processCommand` (in `server.c`), `addReply*` family (writers), `writeToClient`/`sendReplyToClient`, `freeClient`. `connection.c` + `socket.c` / `unix.c` / `tls.c` form the connection abstraction (TLS can be statically linked or built as `redis-tls.so`).
- **I/O threads**: `iothread.c` runs reads/writes on worker threads when `io-threads > 1`. Each thread has its own pending-clients lists; the main thread still executes commands and owns server state. Keep that boundary in mind when touching client/output paths.
- **Event loop**: `ae.c` is a self-contained reactor with backend-specific files (`ae_epoll.c`, `ae_kqueue.c`, `ae_evport.c`, `ae_select.c`) selected at compile time.
- **Persistence**: `rdb.c` (snapshots) and `aof.c` (append-only file) both rely on `fork()` for background work. `call()` is responsible for feeding executed commands into the AOF.
- **Replication**: `replication.c` implements both master and replica sides, including SYNC/PSYNC. `replicationFeedSlaves()` propagates writes. Replicas are represented as special `client` objects.
- **Cluster / Sentinel**: `cluster.c` is the public cluster facade; `cluster_legacy.c` is the gossip-based implementation, `cluster_slot_stats.c` tracks per-slot metrics, `cluster_asm.c` handles agreed slot mapping. `sentinel.c` is compiled into the same binary and activated when invoked as `redis-sentinel`.
- **Storage abstraction**: `kvstore.c` shards a logical database into many `dict`s (one per cluster slot) so cluster-mode rebalancing and per-slot iteration are O(slot) rather than O(db). All `server.db[i].keys/expires/...` are `kvstore *`. `dict.c` is the underlying incrementally-rehashing hash table; `entry.c` (`kvobj`) is the in-db value wrapper.
- **Objects**: `robj` (`object.c`) is the type+encoding+refcount wrapper. Use `incrRefCount`/`decrRefCount`. Many hot paths now skip `robj` and pass plain `sds` strings directly.
- **Expiration**: `expire.c` (active/passive expiration), `ebuckets.c` (radix-tree-of-buckets used for both key TTLs and per-hash-field TTLs), `lazyfree.c` (background freeing), `evict.c` (`maxmemory` policies).
- **Scripting**: `eval.c` (legacy `EVAL`/`EVALSHA`), `functions.c` + `function_lua.c` (`FUNCTION` API), `script.c` + `script_lua.c` (shared Lua runtime). `scripting.c` no longer exists — it was split into the files above.
- **Other notable files**: `module.c` + `redismodule.h` (modules API), `acl.c`, `pubsub.c`, `multi.c` (transactions), `bio.c` (background I/O threads — distinct from `iothread.c`), `defrag.c`, `tracking.c` (client-side caching), `latency.c`, `slowlog.c`, `call_reply.c` + `resp_parser.c` + `logreqres.c` (reply schema infra), `gcra.c` (token-bucket rate limiting), `hotkeys.c`, `keymeta.c`, `vector.c` (vector sets backbone).
- **Core data structures** (each is its own self-contained file): `sds.c` (strings), `dict.c`, `kvstore.c`, `adlist.c` (linked list), `quicklist.c` + `listpack.c` + `ziplist.c` (list encodings), `intset.c`, `rax.c` (radix tree, used by streams), `t_stream.c`, `hyperloglog.c`, `geohash*.c`, `ebuckets.c`, `fwtree.c`, `mt19937-64.c`.

When adding a command: implement `fooCommand(client *c)` in the appropriate `t_*.c` (or `db.c` for generic key-level ops), add `src/commands/<name>.json` describing arity / flags / key specs / arguments, regenerate `src/commands.def` via `utils/generate-command-code.py`, and add tests under `tests/unit/` (or `tests/unit/type/` for type-specific commands).

## Modules

- `src/modules/` — `hello*.c` example modules plus `testmodule.c`. The module test suite (`./runtest-moduleapi`) builds `tests/modules/` first via `make -C tests/modules`. The exhaustive list of moduleapi units it runs is in `runtest-moduleapi` itself — add new module tests there.
- Top-level `modules/` — bundled official modules merged into Redis 8: `redisbloom`, `redisearch`, `redisjson`, `redistimeseries`, `vector-sets`. Built and packaged with the server.
- `redismodule.h` is the public modules API header.

## Conventions

- New work targets the `unstable` branch (per `CONTRIBUTING.md`).
- License: Redis 8+ is tri-licensed under **RSALv2 / SSPLv1 / AGPLv3** (see `LICENSE.txt`); contributions follow the Software Grant and CLA in `CONTRIBUTING.md`. Older files may still carry BSD3 headers (see `REDISCONTRIBUTIONS.txt`); new files should use the tri-license header — copy the form used in recently added files.
- CI builds with `-Werror`; warnings will break the build on GitHub Actions even if local builds succeed. CI matrix includes ASan/UBSan/TSan, TLS module, MSan (clang), 32-bit, libc-malloc, macOS, and external-server runs.
