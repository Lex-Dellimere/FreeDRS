# FreeDRS wire protocol

Line-based text over TCP (default port 5555), `\n`-terminated UTF-8. The server can also run the same protocol inside TLS (`--tls-cert` and `--tls-key`); nothing in the protocol changes, the greeting is sent once the TLS handshake is done.

**Connect / auth phase**. Server sends two greeting lines, then loops reading lines:
- `=== Distributed File System ===` then `Available: login <user> <pass> OR signup <user> <pass> OR guest`
- `guest` (case-insens) -> `SUCCESS: Logged in as GUEST`, `Type 'help' for available commands`. Session user `guest_<rand0-9999>`, role `guest`.
- `login <u> <p>` / `signup <u> <p>`: split on whitespace, max 3 parts (password may contain spaces); <3 parts -> `ERROR: Invalid format. Use: login <user> <pass>, signup <user> <pass> OR guest`; other verb -> `ERROR: Unknown command. Use login or signup.`
- login ok -> `SUCCESS: Welcome back, <user> [<role>]` + help-prompt line. Bad -> `ERROR: Invalid username or password` (stays in loop). DB fault -> `ERROR: Database error: <msg>`.
- signup ok -> `SUCCESS: User registered. Please login.`, 500 ms later server closes (client reconnects and logs in). New users get role `guest`. Duplicate -> `ERROR: Username already exists` (stays in loop).

**Command phase.** Client sends one line; server replies with N response lines then a literal `CMD_END` line. Command verb lowercased; args = rest of line after first whitespace. Flow per command: `ping` -> `PONG` then `CMD_END` as usual (no permission check; client pings every 5 s, declares dead after 12 s without PONG, and swallows PONG) -> role check -> `ERROR: Permission denied. Your role does not allow this command.` -> dispatch. Empty line -> `ERROR: Empty command`. Unknown -> `ERROR: Unknown command. Type 'help' for available commands.` Internal failures -> `ERROR: <msg>`. After `quit`/`exit` the server replies `Goodbye!`+`CMD_END` and closes.

| Command | Reply |
|---|---|
| `pwd` | `Current directory: <cwd>` (cwd starts `/`, per-session) |
| `cd <dir>` | `SUCCESS: Changed directory to: <dir>` (no existence check; usage error `ERROR: Usage: cd <directory>`) |
| `list [dir]` | `Files in <dir\|root>:` + one `  <path> [file\|directory] <n>B - <owner> (<date>)` line per entry whose path starts with `<dir>/`; empty -> `No files found in <dir\|root>` |
| `myfiles` | `Your files:` + same lines (by clientId), or `You don't have any files.` |
| `mkdir <p>` | `SUCCESS: Directory created: <path>` |
| `delete <p>` | `SUCCESS: File deleted` / `ERROR: File not found` / `ERROR: Permission denied` (owner clientId mismatch and owner != `admin`) / `ERROR: Directory not empty` |
| `upload [--lz4] [--resume] <local> [remote]` | `READY: <max-chunk-bytes>` (with ` lz4` when LZ4 is on), no CMD_END yet; then the transfer below. Remote path defaults to the basename of local; relative paths are joined to cwd; leading `/` stripped; `..` rejected (path traversal) |
| `download [--lz4] <p> [offset]` | `FILE_DATA: <n>` (with ` lz4`), data lines, `END_FILE <sha256-hex>`, then `SUCCESS: File sent` and `CMD_END`. A last word of digits is the offset (send `0` for a path that ends in a number). Missing -> `ERROR: File not found: <p>` |
| `whoami` | `You are client #<id> (<user>)` |
| `help` | `Available Commands:` / `====` + role's command list (or `All commands available (admin)`) + examples |
| `refresh` | `SUCCESS: File system metadata reloaded` |
| `listclients` | `Connected clients:` + `  Client #<id>: <user> [<role>]` lines, or `No clients connected` |
| `serverinfo` | `Server info:` then `  Uptime: <d>d <h>h <m>m`, `  CPUs: <n>`, `  Memory: <used> used of <total>` and `  Disk: <free> free of <total> (<mount>)`, the file system that holds the VFS root. A value that cannot be read is `unavailable` |
| `clientinfo <id>` | `Client #<id>: <user> (connected: <date>)` / `ERROR: Client not found` / `ERROR: Invalid client ID` |
| `kick <id>` | `SUCCESS: Client #<id> kicked.` (closes the session) |
| `setrole <user> <role>` | `SUCCESS: Role updated for <user>` / `ERROR: User not found` |
| `listroles` | comma-joined role names |
| `createrole <role> <cmd,cmd,..>` / `deleterole <role>` | `SUCCESS: Role created/updated` / `SUCCESS: Role deleted` |
| `setquota <user> <bytes>` / `setmaxfile <role> <bytes>` | `SUCCESS: Quota updated for <user>` / `SUCCESS: Max file size updated for <role>` (`ERROR: User not found`, `ERROR: Role not found`) |
| `deleteuser <u>` / `renameuser <old> <new>` | `SUCCESS: User deleted` (`ERROR: User not found`) / `SUCCESS: User renamed` (`ERROR: Failed to rename user (may already exist)`) |
| `resetsystem` | wipes users+roles, re-seeds roles, reloads VFS: `SUCCESS: System reset complete` |
| `shutdown` | `SUCCESS: Server shutting down`; process exits after 500 ms |

**Roles.** Table `roles(name, commands, max_file)`; commands is comma-separated verbs, `*` = all. Seeded: `admin = *`, `guest = list,pwd,cd,help,whoami,quit,exit`. Unknown role falls back to the guest set. Roles are checked by verb only. Users table: `users(id, username UNIQUE, password_hash, role, quota)`. `max_file` is the largest upload the role allows and `quota` the bytes a user may store (the sizes of the files they own); 0 means the default (4 GiB, 10 GiB). Setup gives the admin role and the admin user 1 TiB each. File entries keep the SHA-256 of their bytes.

**Streaming transfers.** A data line is the base64 (standard alphabet, with padding) of at most `max-chunk-bytes` (65536) raw bytes. Every line but the last carries a multiple of 3 bytes, so each decodes on its own. With LZ4 each line is instead one LZ4 frame (Press, level 1) of at most 65536 compressed bytes that inflates to at most 65536 bytes, again a multiple of 3 except the last.

- Upload: after `READY`, the client sends `SIZE <n>`. The server checks the role's `max_file` and the user's quota (stored bytes, less the file being replaced if it is theirs, plus their other uploads in progress) before any data, and answers `OFFSET <k>` or `ERROR: File too large: ...` / `ERROR: Quota exceeded: ...` and `CMD_END`. Then the data lines from byte k, then `EOF <sha256-hex>` (the hash of the whole file). Answer: `SUCCESS: File uploaded: <entry>` or `ERROR: Hash mismatch, upload discarded`, then `CMD_END`. The server writes into a temp file and renames it into place only after the hash matches and the file is synced.
- `--resume`: when the user has a partial upload of the same path and size (a connection that broke after some data), `OFFSET` gives where it stopped and the client goes on from there. Partial uploads are kept for 24 hours and are lost when the server restarts. Without `--resume` a new upload replaces the partial one.
- Download: `FILE_DATA: <n>` gives the bytes that follow (file size minus offset); `END_FILE` carries the SHA-256 of the whole file (the stored one when there is one). A read error ends the data with an `ERROR:` line and `CMD_END`. `ERROR: The stored file does not match its hash` after `END_FILE` means the server's copy is damaged.
- Bad input ends the session after `ERROR: <why>` and `CMD_END`: a missing or malformed `SIZE`, bad base64, a line over 87384 characters or with more than 65536 bytes, a short line that is not the last, data beyond `SIZE`, an LZ4 line that is not a valid frame or inflates past 65536 bytes. A transfer with no progress for 60 s ends with `ERROR: Transfer timed out` and the connection closes.


**Other replies.**
- Auth: `ERROR: Invalid username or password (password needs 8 or more characters)` on signup with a short password, and `ERROR: Too many failed logins. Try again later` once lockout kicks in.
- Limits: `ERROR: Line too long` (over 128 KiB; the session closes), `ERROR: Server full` (refused at connect) and `ERROR: Server busy, try again` (the request could not be processed).
- Files: `ERROR: Invalid path` for paths that leave the root or name the server's temp directory `.freedrs-tmp`, and `ERROR: Already exists: <path>` when `mkdir` targets an existing entry.
