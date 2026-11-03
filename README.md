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
  errors raise `PG::Error` (SQLSTATE `code`, `detail`, `hint`, ...).

```ruby
conn = PG.connect("postgres://ann:secret@localhost/shop")
puts conn.server_version
conn.close
```

Errors: `PG::Error` is what the server reported (a refused login
included: SQLSTATE 28P01 for a wrong password); `PG::ConnectionError`
(with `PG::AuthenticationError` and `PG::ProtocolError` under it) means
the connection failed or cannot be used any further.


Next: simple and extended queries with typed results, a connection
pool.

Not yet: TLS, LISTEN/NOTIFY as an API, COPY, cancel requests.

## Differences from libpq

For now, to be revisited:

- A URL without a host connects to localhost over TCP (libpq: the
  default unix socket).
- One host per URL (libpq takes a comma-separated list).
- The default user is `$PGUSER`, then `$USER` (libpq: the operating
  system's user name).
- No TLS: `sslmode` may be `disable`, `allow` or `prefer` only.
- A SCRAM iteration count above 1,000,000 is refused.

## Development

`quarz test` runs the tests, `quarz test --doc` the documentation's
examples. The integration tests connect to `PG_TEST_URL` when it is set;
otherwise, where PostgreSQL's server programs are installed
(`/usr/lib/postgresql/*/bin`), they make a throwaway cluster in a
temporary directory (SCRAM-SHA-256, a random port on 127.0.0.1) and
remove it afterwards; with neither, they are omitted. Licensed under MIT No Attribution (LICENSE).
