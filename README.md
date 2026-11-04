# FreeDRS

FreeDRS is a distributed resource system written in [Mettle](https://github.com/The-Mettle-Project/Mettle). Today it is a file server and command-line client with users, roles and a sandboxed virtual file system. The goal is a cluster of sandboxed nodes that share storage and compute.

## Built on

| Piece | Package |
|---|---|
| Sessions, supervision, offloaded blocking work | alph |
| Users, roles and file metadata | MeoDB server, through flux_orm |
| Passwords (Argon2id) and login lockout | raize_auth, cryptex |
| Files, time, sockets | Mettle stdlib (`std/file`, `std/time`, `std/socket`) |
| Config | json (registry) |

## Layout

```
mbuild.mettle        build file: bins, profiles, deps, meodb service
src/auth/            login, signup, guest, role checks
src/config/          server_config.json loading
src/db/              flux_orm models and store
src/vfs/             root-confined virtual file system
src/server/          alph server: boss, acceptor, session actors, commands
src/client/          freedrs_client CLI
src/setup/           freedrs_setup: default roles and admin user
tests/               mbuild test suites (unit and end-to-end)
docs/                design, wire protocol
```

## Build and run

All of it goes through [mbuild](https://github.com/Lex-Dellimere/mbuild). The path dependencies expect the Mettle workspace layout (`../meodb`, `../flux_orm`, `../raize_auth`, `../cryptex`, `../alph` next to this repo).

```
mbuild build
mbuild test
mbuild run freedrs_setup        # starts meodb, creates roles and the admin user
mbuild run freedrs_server       # starts meodb, then the server on :5555
mbuild run freedrs_client --no-services -- 127.0.0.1:5555 --user admin
```

The client protocol is described in [docs/PROTOCOL.md](docs/PROTOCOL.md).
