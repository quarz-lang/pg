# Working on pg

A cog of Quarz (github.com/quarz-lang/quarz): its rules apply here.
Code, comments, commit messages and the docs are in English.

## Layout

- `Cog.toml` (`module = "PG"`), `src/` (declarations only), `test/*_test.qz`
  (`require "test"`, Test::Case), README.md, LICENSE.
- The wire protocol is written in Quarz (not libpq), so that a query only
  suspends its task.

## Before committing

```sh
quarz check
quarz test              # interpreted
quarz test --release    # compiled
quarz test --doc        # the doc comments' examples
quarz doc --check
```

Use a `quarz` built from Quarz's main (`cargo build --release` there).
The integration tests (test/server.qz) use `PG_TEST_URL` when it is set
(CI's `postgres:16` service sets it), else a throwaway cluster made with
`/usr/lib/postgresql/*/bin/initdb` and `pg_ctl` (random port, temporary
directory, removed in the test class's shutdown); with neither they are
omitted with a message.

- Work on a branch and open a pull request; stage files by explicit path.
- Review every commit with a fresh subagent given the diff, and fix what
  it finds that needs no decision.
- Never print secrets (passwords, tokens).
- Do not put a `Claude-Session:` line in commit messages, nor a Claude
  session URL in pull request descriptions.
