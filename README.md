# FreeDRS

FreeDRS is a distributed resource system written in [Mettle](https://github.com/The-Mettle-Project/Mettle). Today it is a file server and command-line client with users, roles and a sandboxed virtual file system. The goal is a cluster of sandboxed nodes that share storage and compute.

## Built on

| Piece | Package |
|---|---|
| Sessions, supervision, offloaded blocking work | alph |
| Users, roles and file metadata | MeoDB server, through flux_orm |
| Passwords (Argon2id) and login lockout | raize_auth, cryptex |
| Files, time, sockets, terminal | Mettle stdlib (`std/file`, `std/fs`, `std/time`, `std/socket`, `std/term`) |
| Reading the config | json (registry) |
| Writing the config | codec (JSON writer) |
| Command-line flags | args |
| Server log | log |
| `serverinfo` command | sysinfo |
| Optional TLS | mtls |

## Layout

```
mbuild.mettle        build file: bins, profiles, deps, meodb service
src/auth/            login, signup, guest, role checks
src/config/          server_config.json loading and saving
src/db/              flux_orm models and store
src/vfs/             root-confined virtual file system
src/server/          alph server: boss, acceptor, session actors, commands
src/client/          freedrs_client CLI
src/setup/           freedrs_setup: default roles and admin user
tests/               mbuild test suites (unit and end-to-end)
docs/                wire protocol
```

## Build and run

All of it goes through [mbuild](https://github.com/Lex-Dellimere/mbuild). The path dependencies expect the Mettle workspace layout (`../meodb`, `../flux_orm`, `../raize_auth`, `../cryptex`, `../alph`, `../codec`, `../args`, `../log`, `../sysinfo`, `../mtls` next to this repo).

```
mbuild build
mbuild test
mbuild run freedrs_setup        # starts meodb, asks for port, directory and admin password,
                                # writes server_config.json, creates roles and the admin user
mbuild run freedrs_server       # starts meodb, then the server (port from server_config.json, else 5555)
mbuild run freedrs_client --no-services -- 127.0.0.1:5555 --user admin
```

Every program takes `--help` and `--version`.

| Program | Flags |
|---|---|
| `freedrs_setup` | `--meodb IP:PORT`, `--admin-user NAME` |
| `freedrs_server` | `--config FILE`, `--port N`, `--root DIR`, `--meodb IP:PORT`, `--tls-cert FILE`, `--tls-key FILE` |
| `freedrs_client` | `[HOST[:PORT]]`, `--guest`, `--user NAME`, `--pass PASSWORD`, `--tls`, `--ca FILE` |

The server logs to stderr; `METTLE_LOG=debug` changes the level.

## TLS

Connections are plain TCP unless the server is given a certificate. The certificate file is a PEM chain, leaf first, and the key a PEM private key:

```
mbuild run freedrs_server -- --tls-cert cert.pem --tls-key key.pem
mbuild run freedrs_client --no-services -- myhost:5555 --tls --user admin
mbuild run freedrs_client --no-services -- myhost:5555 --tls --ca ca.pem --user admin
```

The client checks the certificate against the system roots, or against `--ca FILE` when given, and the host name you connect to must be in the certificate. A TLS server accepts only TLS clients and the other way round. The TLS code comes from the mtls package, which has not been audited.

The client protocol is described in [docs/PROTOCOL.md](docs/PROTOCOL.md).
