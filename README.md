# pg

A PostgreSQL client for [Quarz](https://quarz.dev), written in Quarz: it
speaks the server's wire protocol (version 3) over Quarz's sockets, so a
query suspends only the task that waits for it.

```toml
# Cog.toml of an application
[dependencies]
PG = { git = "https://github.com/quarz-lang/pg", tag = "v0.1.0" }
```

Work in progress. What exists:

- `PG::Wire`: big-endian integers and NUL-terminated strings, and
  `PG::MessageReader`, which reads one backend message at a time.
- `PG::Frontend`: the messages a client sends (startup, password and
  SASL, simple and extended queries, sync, terminate).
- `PG::Backend`: the messages a server sends, decoded into an enum.

- `PG.connect(url)` / `PG.connect(host:, port:, user:, password:, ...)`:
  TCP or a unix socket (a `host` starting with `/`), authentication by
  SCRAM-SHA-256 (`PG::Scram`), md5, cleartext (over a unix socket, or
  with `allow_cleartext: true`) or trust, `connect_timeout`. Server
  errors raise `PG::ServerError` (SQLSTATE `code`, `detail`, `hint`, ...).

- TLS over TCP, through Quarz's `TLSSocket` (rustls), as `sslmode`
  (keyword or URL parameter) says; `conn.tls?` tells.

  | `sslmode` | |
  |---|---|
  | `disable` | plain |
  | `allow` | plain; when the server refuses the session (`hostssl` only), once more over TLS |
  | `prefer` (the default) | TLS when the server speaks it, else plain |
  | `require` | TLS, else `PG::ConnectionError` |
  | `verify-ca`, `verify-full` | TLS, the server's certificate and name verified |

  `sslrootcert` names the roots: a PEM file, or `system` for the
  system's store (then `sslmode` is `verify-full` unless given, and a
  weaker one than `verify-ca` is an `ArgumentError`). Without it,
  `~/.postgresql/root.crt` is read when it exists. With roots, `allow`,
  `prefer` and `require` verify the server as `verify-full` does (so a
  stale `root.crt` makes `require` fail); without them they encrypt
  without verifying, and `verify-ca` and `verify-full` fail. A unix
  socket never uses TLS, whatever `sslmode` says.

  ```ruby
  PG.connect("postgres://app@db.example/shop?sslmode=verify-full&sslrootcert=/etc/pg/ca.pem")
  ```

- `conn.exec(sql)` (simple protocol; several statements: the last
  statement's result; `exec_all` returns all of them) and `conn.query(sql, params)`
  (extended protocol, `$1`... sent apart from the SQL: nil, Int, Float,
  String, Bool, Bytes, Time, Symbol, or an `Array` of one of them),
  returning a `PG::Result`: columns, `PG::Row`s with the server's text
  (`row["name"]`) and typed readers (`int`, `float`, `string`, `bool`,
  `bytes`, `time`, each with a `?` form for NULL; for arrays `ints`,
  `floats`, `strings`, `bools`, `times`), `command_tag`, `affected`.
  After an error the connection stays usable.
- Arrays. An `Array` parameter is one parameter, a one-dimensional
  PostgreSQL array in text format (`{1,2,3}`, `{"b,c",NULL}`), however
  long: `WHERE id = ANY($1)` with 70,000 ids is one parameter and one
  prepared statement, not 70,000 (the protocol's limit is 65535
  parameters). Its elements are of one class (`ArgumentError` for mixed
  classes or a nested array) and nil is NULL. Its type:

  | Elements | Sent as | A bare `unnest($1)`, `SELECT $1` |
  |---|---|---|
  | Int | `int8[]` | needs no cast |
  | Bool | `bool[]` | needs no cast |
  | Bytes | `bytea[]` | needs no cast |
  | Float, String, Symbol, Time | untyped, as their scalars | needs one: `$1::text[]` |
  | none (empty, or nils only) | untyped | needs one: `$1::bigint[]` |

  Untyped, the server infers the type from the context, so that the
  column decides, as for a scalar: `= ANY($1)` beside a `uuid`, `date`,
  enum, `citext`, `numeric`, `real` or `timestamp` column compares as
  that column does (typed, a `text[]` would not meet a `uuid` column and
  a `timestamptz[]` would shift a `timestamp` column's values by the
  session's time zone). Scalar parameters are untyped, as before. The
  array readers return
  `Array[T?]` (NULL elements as nil), with a `?` form for a NULL array;
  an array of more than one dimension raises `ArgumentError` (`row[...]`
  gives its text).
- `conn.transaction { |c| ... }`: BEGIN, then COMMIT, or ROLLBACK when
  the block raises; `on_notice { |fields| ... }`.

```ruby
conn = PG.connect("postgres://ann:secret@localhost/shop")
conn.query("SELECT id, name FROM users WHERE id = $1", [42]).each do |row|
  puts "#{row.int("id")} #{row.string("name")}"
end
conn.close
```

Errors: `PG::Error` is the root of every error the cog raises.
`PG::ServerError` is what the server reported (SQLSTATE `code`,
`detail`, `hint`, `position`; a refused login included, 28P01 for a wrong
password); `PG::ConnectionError` (with `PG::AuthenticationError` and
`PG::ProtocolError` under it) means the connection failed or cannot be
used any further; `PG::TransactionError` and `PG::NullError` are misuses.


- `query` prepares each SQL text once per element types of its array
  parameters (Parse fixes the parameters' types; their lengths do not
  matter, so one statement serves every length; an `= ANY($1)` used
  with both empty and non-empty arrays holds two) and keeps the
  `statement_cache_size` (default 100; 0 for a pooler in transaction
  mode) most recently used statements; one the server no longer has
  (26000) or says changed under it (0A000, outside a transaction) is
  prepared again and the call retried once.

- `PG::Pool.open(url, size:, checkout_timeout:) { |pool| ... }` (or
  `PG::Pool.new` and `#close`): connections shared by tasks on any
  worker. An idle connection is kept as a Shareable `PG::Lease` (its
  socket and session state); `pool.with { |c| ... }` makes a
  `PG::Connection` of one in the calling task, runs an ordinary block,
  and gives it back (rolled back first if the block left a transaction
  open; closed instead if it broke or a call was cut short, and replaced
  when needed). `pool.exec`, `pool.query`, `pool.transaction` are
  shortcuts. `checkout_timeout` bounds the wait for a free connection
  (`PG::PoolTimeout`). Prepared statements stay with their connection.
  `PG::Connection#release` and `.adopt` move a single connection between
  tasks the same way.

```ruby
PG::Pool.open("postgres://app@localhost/shop", size: 8) do |pool|
  Task.scope do |s|
    100.times { |i| s.spawn { pool.query("INSERT INTO hits VALUES ($1)", [i]) } }
  end
end
```

- `PG::Testing`, for the tests of a program or cog that uses pg:
  `PG::Testing.with_cluster { |url| ... }` runs the block against
  `PG_TEST_URL`'s server when that is set, or else a throwaway cluster
  made with PostgreSQL's own programs (`initdb`, `pg_ctl`) in a
  temporary directory, removed afterwards; `PG::Testing::Unavailable`
  when there is neither. `PG::Testing.start` and `start_or_reason`
  keep one server for a test class (`startup`/`shutdown`, omitting the
  tests when there is none). Given `certificate:` and `private_key:`
  (PEM), the cluster is always a throwaway one, with TLS on.

Not yet: LISTEN/NOTIFY as an API, COPY, cancel requests, client
certificates, channel binding.

## Differences from libpq

For now, to be revisited:

- A URL without a host connects to localhost over TCP (libpq: the
  default unix socket).
- One host per URL (libpq takes a comma-separated list).
- The default user is `$PGUSER`, then `$USER` (libpq: the operating
  system's user name).
- `verify-ca` verifies the server's name too, as `verify-full` does
  (stricter than libpq: rustls has no chain-only verification), and so
  do `allow`, `prefer` and `require` when there are roots (libpq
  verifies them as `verify-ca`): a server reached by a name or address
  its certificate does not list is refused. For the same reason
  `sslrootcert=system` accepts `verify-ca` (libpq: `verify-full` only).
- No client certificates (`sslcert`, `sslkey`, `sslpassword`), no
  revocation lists (`sslcrl`), no channel binding (`channel_binding`
  may be `disable` or `prefer`, and the client uses SCRAM-SHA-256 even
  when the server offers SCRAM-SHA-256-PLUS), no direct TLS negotiation
  (`sslnegotiation=postgres` only): the others raise `ArgumentError`.
- With `prefer`, a server that agrees to TLS and then fails the
  handshake (a stale `root.crt` included) is an error (libpq tries
  again in plain).
- A SCRAM iteration count above 1,000,000 is refused unless the
  connection raises `max_scram_iterations` (keyword or URL parameter).

## Development

`quarz test` runs the tests, `quarz test --doc` the documentation's
examples. The integration tests use `PG::Testing`: `PG_TEST_URL` when it
is set; otherwise, where PostgreSQL's server programs are installed
(`/usr/lib/postgresql/*/bin`), a throwaway cluster in a temporary
directory (SCRAM-SHA-256, a random port on 127.0.0.1), removed
afterwards; with neither, they are omitted. The TLS tests
(`test/tls_test.qz`) always make clusters of their own, with the
certificates of `test/fixtures/tls/`, and are omitted without `initdb`. Licensed under MIT No Attribution (LICENSE).

To develop another cog against a local checkout of pg, write
`Cog.local.toml` beside that cog's `Cog.toml` (git should ignore it;
`quarz cog init` lists it in `.gitignore`):

```toml
[patch]
PG = { path = "../quarz-pg" }
```

That cog's lock ignores the file: it keeps pg's git source.
