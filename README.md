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

Next: connections and authentication (SCRAM-SHA-256, md5, cleartext,
trust; TCP and unix sockets), simple and extended queries with typed
results, a connection pool.

Not yet: TLS, LISTEN/NOTIFY as an API, COPY, cancel requests.

## Development

`quarz test` runs the tests, `quarz test --doc` the documentation's
examples. Licensed under MIT No Attribution (LICENSE).
