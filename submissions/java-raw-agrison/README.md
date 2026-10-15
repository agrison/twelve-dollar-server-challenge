# java-raw-agrison — Java 27, no framework, FFM binding to a self-compiled SQLite

The API in ~1,900 lines of plain Java on the JDK alone: no framework, no server library, no JSON
library, no SQLite driver. Requests are parsed in place out of a direct `ByteBuffer`, and every byte
that comes from the database is serialised by SQLite itself (`json_object`) and copied straight from
SQLite's own buffer into the response buffer.

| | |
|---|---|
| Language / runtime | Java 27 — OpenJDK 27 GA (`27+35`), pinned by SHA-256 in `install.sh` |
| Framework | none (JDK only: `ServerSocketChannel`, virtual threads, `java.lang.foreign`) |
| Database / driver | SQLite **3.54.0**, compiled from the pinned amalgamation on the box, bound with **FFM (Panama)**. No JDBC |
| JSON | **SQLite** writes every value (`json_object`); a hand-written strict RFC 8259 scanner reads the JWT payload |
| **Nginx or direct** | **Direct** on `0.0.0.0:80`. It also runs behind Nginx unchanged — it only reads `HOST`/`PORT` |
| Dependencies | none beyond the JDK and the SQLite amalgamation, both pinned with checksums |

## Why direct

Nginx costs 14–20% of the only vCPU, and `bench/nginx.conf` turns on `gzip` for JSON over 1 KB —
which the feed response is, so that CPU is spent compressing on the same core that serves requests.
Going direct means owning up to ~15,000 keep-alive connections instead. With virtual threads an idle
connection is a parked continuation plus a 2 KB read buffer, so idle connections cost memory, not CPU
— the trade this design is built around. Nothing in the code assumes either mode.

## Layout

```
k6 ──TCP──► :80 ──► acceptor (platform thread)
                      └─► one virtual thread per connection: read, parse in place, dispatch
                            │   borrow Out  (80 KB render buffer + native scratch, pool of 64)
                            │   borrow Ctx  (SQLite connection + statements + HMAC, pool of 4)
                            │     GET  → sqlite3_step → memcpy json_object bytes → release Ctx
                            │     POST → verify JWT + validate body → release Ctx → queue WriteOp
                            ▼                                              │
                       one write() per response                            ▼
                                                      writer (platform thread):
                                                      BEGIN IMMEDIATE; N ops; COMMIT; unpark each
```

`src/app/`: `Main` (bootstrap/accept), `Connection` (per-connection loop), `Http` (parsing),
`Router`, `Handlers` (the five endpoints), `Sqlite` (FFM binding), `Db` (connections, statements,
pragmas), `Ctx`, `Out`, `Writer`/`WriteOp` (group commit), `Jwt`, `JsonScan`, `Responses` (constants).

## The optimizations, and why

**SQLite builds the JSON.** The feed is one statement returning 20 `json_object` rows; Java writes
`{"posts":[`, the commas and `]}` and copies each row with `MemorySegment.copy` directly from the
pointer `sqlite3_column_text` returned. No Java string, no object, no escaping code of my own — and
the escaping is SQLite's, which is what the golden files were generated with.

**FFM instead of JDBC.** JDBC would mean `ResultSet`s, `String`s and synchronisation for data that
only needs to be copied from one buffer to another. Every binding is a `static final MethodHandle`
invoked with `invokeExact`, so C2 inlines the call. The short read-side calls (`bind`, `step`,
`column_*`, `reset`) use `Linker.Option.critical`, which skips the thread-state transition; the
writer thread deliberately does **not**, because its `COMMIT` can fsync.

**Compiling SQLite myself** allows `-O2 -march=native` plus `SQLITE_THREADSAFE=2` (no per-call
mutexes; one connection is only ever used by one thread at a time), `DEFAULT_MEMSTATUS=0`, `DQS=0`
and omitting code this server never calls. Rule 10 is satisfied: it is built from source on the box.

**Two pools, deliberately different sizes.** A `Ctx` (SQLite connection + persistent statements +
`Mac`) is expensive, and the code between borrowing and returning it never parks — no locks, no
logging, no socket I/O — so four of them saturate a single carrier thread. An `Out` (the render
buffer) *is* held across the socket write, where a slow client can park us, so there are 64.

**One `write()` per response, no memmove.** The body is rendered from offset 128 of the render
buffer; once its length is known the status line and `Content-Length` are written *backwards*,
right-aligned against the start of the body. Header and body are then one contiguous range, so each
response is a single `write()` with an exact `Content-Length`. Status lines, error bodies and JSON
frames are pre-encoded `byte[]`; integers are written as ASCII digits by hand.

**Nothing on the hot path allocates.** No `String` anywhere in parsing or routing: methods, paths,
header names and the JWT are compared as bytes at offsets into the connection's buffer. The HMAC is
taken over the token bytes copied into a per-`Ctx` array (no `ByteBuffer` view object), and the
base64url decoder and encoder are hand-written into reused buffers.

**Group commit.** Writes go to one dedicated platform thread, so a `COMMIT` that triggers a WAL
checkpoint and an fsync never blocks the carrier thread serving reads. Whatever queued up during the
previous transaction is committed together, and each request waits for *its own* commit before
responding — a 201 always means the row is on disk (rule 6: WAL + `synchronous=NORMAL`).

**Query shapes.** `CROSS JOIN` pins the join order (posts as the outer loop) so the plan never
depends on statistics, and `EXPLAIN QUERY PLAN` confirms `SCAN p USING INDEX posts_created_at_id_idx`
with no temp B-tree for the `ORDER BY`. The body of `POST /posts` is validated *and trimmed* by
SQLite (`json_valid`, `json_type`, `trim`, `length`), which reproduces Node's semantics exactly
without a line of parsing code, and `INSERT … RETURNING json_object(…)` returns the created post's
JSON from the same statement that writes it.

### JVM flags (`start.sh`)

| Flag | Why |
|---|---|
| `-XX:+UseSerialGC` | On 1 vCPU, G1/ZGC pay for concurrent threads and write barriers with nothing to gain |
| `-Xms512m -Xmx512m -Xmn256m`, `AlwaysPreTouch` | No heap resizing mid-benchmark, pages already faulted in; a large young gen with a tiny live set means young collections are rare |
| `+UseTransparentHugePages` | Fewer TLB misses on the heap (Ubuntu's THP is `madvise`, so the JVM flag is enough — no kernel tuning) |
| `+UseCompactObjectHeaders` | 8-byte headers: less memory and fewer cache misses |
| `virtualThreadScheduler.parallelism=1` | One carrier thread for one vCPU: no contention, no cross-core cache traffic |
| `jdk.trackAllThreads=false`, `-XX:-UsePerfData` | Skips per-virtual-thread registration and the mmapped hsperfdata file |
| `--enable-native-access=ALL-UNNAMED` | FFM without warnings |

Tunables, all with defaults that can be overridden without touching code:
`-Dapp.readers` (SQLite read connections, 4), `-Dapp.outs` (render buffers, 64).

## What I measured

`test/run.sh` passes **42/42** on a fresh copy of the seed database.

### The scripts, on a clean Ubuntu 24.04 x86_64 container

Exactly what CI and the droplet run, from a bare `ubuntu:24.04` with nothing installed. The host is
arm64, so x86_64 ran emulated: the timings are proof the steps work, not performance figures.

| Step | Result |
|---|---|
| `install.sh` (root) | 36 s — JDK tarball SHA-256 verified, `OpenJDK 27+35` reported by `java -version` |
| `build.sh` (normal user) | 21 s — amalgamation SHA-256 verified, `libsqlite3.so` compiled with `-march=native`, then `javac 27 --release 27` |
| `start.sh` | all ten Linux-only JVM flags accepted, no warning on stderr |
| `test/test.sh` | **42/42** |

### On a 1 vCPU / 2 GB Ubuntu 24.04 container

Docker, `--cpus=1 --memory=2g --memory-swap=2g`, native arm64 (no emulation), the pinned JDK 27 and
the self-compiled `libsqlite3.so`, with k6 in a sibling container so the generator never competes
for the server's single core. **This is a proxy, not the droplet**: one core of an Apple M4 Pro is
substantially faster than a shared DigitalOcean vCPU, and container-to-container networking is
cheaper than 15k real clients over the internet. Read it for headroom and scaling shape, not as a score.

| k6 `VUS` | req/s | p95 | p99 | errors | server RSS |
|---|---|---|---|---|---|
| 1,000 | 80 | 1.51 ms | 2.21 ms | 0.00% | — |
| 5,000 | 386 | 1.11 ms | 1.73 ms | 0.00% | 618 MB |
| 10,000 | 769 | 1.14 ms | 1.86 ms | 0.00% | 649 MB |
| 20,000 | 1,495 | 1.03 ms | 1.77 ms | 0.00% | 702 MB |
| **20,000, full 5-minute hold** | **1,868** | **1.09 ms** | **2.39 ms** | **0.00%** | **785 MB** |

The 5-minute hold at 20,000 VUs passed all three thresholds on 747,379 requests with 100% of checks
succeeding, zero failures and not one line in the server log. p99 of 2.39 ms against the 1 s budget
is a 418x margin, and latency does not degrade between 1,000 and 20,000 VUs.

`start.sh` to the first 200 on `/health`: **281 ms** on one vCPU, which includes `AlwaysPreTouch`
faulting in the whole 512 MB heap and opening five SQLite connections. Rule 10 allows 60 s.

The headroom number: during a 20,000 VU run the server used **mean 23.7% (peak 31%) of its single
vCPU** at 864 MB of the 2 GB. So on this core it is CPU-idle at 20k users, and memory grows by
roughly 14 KB per connection including kernel socket memory — about 1.1–1.3 GB extrapolated to
30–40k connections.

### Microbenchmarks (Apple M4 Pro, macOS, one carrier thread)

| Measurement | Result |
|---|---|
| `GET /feed` (20 rows, ~5 KB, 20 `count(*)`) | 48,100 req/s |
| `GET /posts/:id` | 145,800 req/s |
| `POST /posts` (c=30) | 25,500 req/s |
| Allocation on the GET paths | **43 bytes/request, identical for `/nope` (no SQL at all) and `/feed` (20 rows + 20 native copies)** — the handlers allocate nothing measurable; the residue is the JDK's blocking-NIO/virtual-thread read path. 200k GETs produced **0** `jdk.ObjectAllocationSample`, `ObjectAllocationInNewTLAB`, `YoungGarbageCollection` and `VirtualThreadPinned` events |
| RSS over a 200 s mixed read+write run | flat from t=60 s onwards (±0.1%), WAL capped at 16.5 MB while inserting ~933k posts |
| Write correctness under concurrency | 400 concurrent likes from 400 distinct users → 400x201 and `like_count` reads back as exactly 400; 300 concurrent creates → 300 distinct ids, every `author` matching its own token |

**Feed shape, A/B tested** (two servers, same machine, three runs each): 20 `json_object` rows framed
by Java — **48,100 req/s** — against the whole document built in SQLite with
`json_object('posts', json_group_array(...))` — **35,400 req/s**. Both produce byte-identical golden
output; the second re-parses and re-serialises every row and materialises a 5 KB string inside
SQLite. The row-by-row version is what ships.

**Still to be measured on the droplet itself**: `parallelism` 1 vs 2, the two pool sizes,
`-Xmn`/heap sizing, `wal_autocheckpoint`, group-commit batch size, and direct vs behind Nginx.

## Rules

- **No caching across requests (rule 5).** Every request reads SQLite while it is being served.
  There is no response, row or query-result cache, and no memory of verified tokens: every request
  recomputes the HMAC and rechecks `exp`. What is reused is only SQLite's own page cache and `mmap`,
  prepared statements (plans, not data), and constant response bytes.
- **Durable writes (rule 6).** WAL with `synchronous=NORMAL`; the response is written only after the
  `COMMIT` that contains it returns.
- **Schema untouched (rule 3).** No index, trigger or `ANALYZE` (which would write `sqlite_stat1`).
- **Nothing benchmark-specific (rule 7).** No load-test detection, no hard-coded responses.
- **No system tuning (rule 11).** Only in-process settings: JVM flags, pragmas, SQLite build flags.

## Building

`install.sh` (root, once) installs `build-essential` and the pinned JDK 27 into `/opt/jdk-27`.
`build.sh` (normal user) downloads the SQLite amalgamation, verifies its SHA-256, compiles
`native/libsqlite3.so` and then `out/` with `javac --release 27`. `start.sh` runs the server in the
foreground. `install.sh` picks the x86_64 or aarch64 JDK tarball from `uname -m`, each pinned with
its own published SHA-256; the benchmark box is x86_64.

All three scripts were verified end to end on a clean **Ubuntu 24.04 x86_64** container (JDK 27+35,
`javac --release 27`, `-march=native` on the amalgamation, every Linux JVM flag accepted, 42/42) and
again natively on arm64 under the 1 vCPU / 2 GB cap above. They also work on macOS with
`JDK_HOME`/`JAVA_HOME` set, which is how the microbenchmarks were taken.
