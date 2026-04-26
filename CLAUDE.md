# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This is the Redis server (C codebase). Five binaries are produced from `src/` and they all share `redis-server` as the actual ELF — `redis-sentinel`, `redis-check-rdb`, and `redis-check-aof` are installed as copies/symlinks that change behavior based on `argv[0]`. `redis-cli` and `redis-benchmark` are independent binaries.

Bundled dependencies live under `deps/` (`jemalloc`, `lua`, `hiredis`, `linenoise`, `hdr_histogram`). The system only needs `libc`, a C compiler, and (for tests) `tclsh` 8.5+.

## Build

```sh
make                       # default build (jemalloc on Linux, libc malloc elsewhere)
make BUILD_TLS=yes         # link OpenSSL; required for ./runtest --tls
make USE_SYSTEMD=yes       # link libsystemd
make MALLOC=libc           # libc | jemalloc | tcmalloc | tcmalloc_minimal
make 32bit                 # 32-bit build (needs libc6-dev-i386 on Debian/Ubuntu)
make noopt                 # -O0, useful for debugging
make valgrind              # -O0 + libc malloc, suitable for valgrind
make gcov / make lcov      # coverage build / HTML report
make V=1                   # verbose (non-colorized) build output
make REDIS_CFLAGS='-Werror' # CI uses this; warnings fail the build
```

`make distclean` is required after changing build options (32bit ↔ 64bit, MALLOC, etc.) or after pulling changes that touch `deps/`. Plain `make` does **not** rebuild dependencies on its own. Build flags are cached in `src/.make-settings` until `make distclean` clears them.

`make install` (optionally `PREFIX=...`) installs the five binaries to `$PREFIX/bin`.

## Tests

The Redis test suite is written in Tcl and driven by `tests/test_helper.tcl`. The four entry-point shell scripts at the repo root each select a different driver:

```sh
./runtest                  # main suite (unit/ + integration/)
./runtest-cluster          # cluster tests (tests/cluster/run.tcl)
./runtest-sentinel         # sentinel tests (tests/sentinel/run.tcl)
./runtest-moduleapi        # module-API tests (builds tests/modules/ first)
make test                  # = ./runtest, after building required binaries
make test-sentinel         # = ./runtest-sentinel
```

Useful `runtest` flags (forwarded to `test_helper.tcl`):

```sh
./runtest --single unit/type/list      # run one unit
./runtest --only <test-name>           # run one named test (repeatable)
./runtest --skipunit unit/expire       # skip a unit
./runtest --list-tests                 # list all available units
./runtest --tls                        # run against TLS (needs make BUILD_TLS=yes
                                       # and ./utils/gen-test-certs.sh first)
./runtest --valgrind                   # run under valgrind
./runtest --clients <n>                # parallelism (default 16)
./runtest --stop                       # halt on first failure
./runtest --verbose / --quiet
```

The set of units run by default is the `::all_tests` list at the top of `tests/test_helper.tcl`; `--single` is the way to run anything outside that list (e.g. moduleapi tests).

## Architecture

`README.md` contains an extended internals tour — read it before making non-trivial changes. The high-level shape:

- **Entry point**: `src/server.c` — `main()`, `initServer()`, the `aeMain()` event loop, `serverCron()` (periodic tasks), `beforeSleep()` (per-loop hook), `call()` (executes a command in a client context), and the global `redisCommandTable` that maps command name → handler.
- **Global state**: the `struct redisServer server` global in `src/server.h` holds databases (`server.db`), the command table, the client list, and replication state. `struct client` holds per-connection state (`querybuf`, `argc`/`argv`, output `reply`/`buf`, `db`).
- **Commands**: each command is a `void fooCommand(client *c)` function that reads `c->argv` and calls `addReply*()` to respond. The function is registered in `redisCommandTable` in `server.c`. Generic key-level commands (DEL, EXPIRE, …) live in `db.c`; type-specific commands live in `t_string.c`, `t_list.c`, `t_set.c`, `t_zset.c`, `t_hash.c`, `t_stream.c`. `db.c` also exposes `lookupKeyRead/Write`, `dbAdd`, `setKey`, `dbDelete`, `emptyDb` — use these rather than poking the dict directly.
- **Networking**: `networking.c` — `createClient`, `readQueryFromClient` (reader), `processInputBuffer` → `processCommand` (in `server.c`), `addReply*` family (writers), `writeToClient`/`sendReplyToClient`, `freeClient`. `connection.c` + `tls.c` abstract plain TCP vs TLS.
- **Event loop**: `ae.c` is a self-contained reactor with backend-specific files (`ae_epoll.c`, `ae_kqueue.c`, `ae_evport.c`, `ae_select.c`) selected at compile time.
- **Persistence**: `rdb.c` (snapshots) and `aof.c` (append-only file) both rely on `fork()` for background work. `call()` is responsible for feeding executed commands into the AOF.
- **Replication**: `replication.c` implements both master and replica sides, including SYNC/PSYNC. `replicationFeedSlaves()` propagates writes. Replicas are represented as special `client` objects.
- **Cluster / Sentinel**: `cluster.c` is the Redis Cluster implementation (read the cluster spec before editing). `sentinel.c` is compiled into the same binary and activated when the binary is invoked as `redis-sentinel`.
- **Objects**: `robj` (`object.c`) is the type+encoding+refcount wrapper for values. Use `incrRefCount`/`decrRefCount`; many places now skip `robj` and pass plain `sds` strings for hot paths.
- **Core data structures** (each is its own self-contained file): `sds.c` (strings), `dict.c` (incrementally rehashing hash table), `adlist.c` (linked list), `quicklist.c` + `listpack.c` + `ziplist.c` (list encodings), `intset.c`, `rax.c` (radix tree, used by streams), `t_stream.c`, `hyperloglog.c`, `geohash*.c`.
- **Other notable files**: `scripting.c` (Lua), `module.c` (modules API), `evict.c` + `expire.c` + `lazyfree.c` (memory/key lifecycle), `acl.c` (ACLs), `pubsub.c`, `multi.c` (transactions), `bio.c` (background I/O threads), `defrag.c`, `tracking.c` (client-side caching).

When adding a command: implement `fooCommand(client *c)` in the appropriate file, add a row to `redisCommandTable` in `server.c` (name, function, arity, flag string, key-spec fields), and add tests under `tests/unit/` (or `tests/unit/type/` for type-specific commands).

## Modules

Module sources are under `src/modules/` (the `hello*.c` examples plus `testmodule.c`). The module test suite (`./runtest-moduleapi`) builds `tests/modules/` first via `make -C tests/modules`. The exhaustive list of moduleapi units it runs is in `runtest-moduleapi` itself — add new module tests there.

## Conventions

- New work targets the `unstable` branch (per `CONTRIBUTING` and `README.md`); upstream PRs against any other branch will be redirected.
- BSD license — every new source file should include the BSD header (see existing files for the exact wording).
- CI builds with `-Werror`; warnings will break the build on GitHub Actions even if local builds succeed.
