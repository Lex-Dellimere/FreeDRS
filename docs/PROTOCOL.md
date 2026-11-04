# FreeDRS wire protocol

Line-based text over TCP (default port 5555), `\n`-terminated UTF-8.

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
| `upload <local> [remote]` | server: `READY: Send file data (base64 encoded). END with 'EOF' on a new line.` (no CMD_END yet); client sends the base64 data (one line) then line `EOF`; server then `SUCCESS: File uploaded: <entry>` or `ERROR: No file data received`, + `CMD_END`. Remote path defaults to basename of local; relative paths are joined to cwd; leading `/` stripped; `..` rejected (path traversal) |
| `download <p>` | `FILE_DATA:`, one base64 line, `END_FILE`, then `SUCCESS: File sent`, `CMD_END`. Missing -> `ERROR: File not found: <p>` |
| `whoami` | `You are client #<id> (<user>)` |
| `help` | `Available Commands:` / `====` + role's command list (or `All commands available (admin)`) + examples |
| `refresh` | `SUCCESS: File system metadata reloaded` |
| `listclients` | `Connected clients:` + `  Client #<id>: <user> [<role>]` lines, or `No clients connected` |
| `clientinfo <id>` | `Client #<id>: <user> (connected: <date>)` / `ERROR: Client not found` / `ERROR: Invalid client ID` |
| `kick <id>` | `SUCCESS: Client #<id> kicked.` (closes the session) |
| `setrole <user> <role>` | `SUCCESS: Role updated for <user>` / `ERROR: User not found` |
| `listroles` | comma-joined role names |
| `createrole <role> <cmd,cmd,..>` / `deleterole <role>` | `SUCCESS: Role created/updated` / `SUCCESS: Role deleted` |
| `deleteuser <u>` / `renameuser <old> <new>` | `SUCCESS: User deleted` (`ERROR: User not found`) / `SUCCESS: User renamed` (`ERROR: Failed to rename user (may already exist)`) |
| `resetsystem` | wipes users+roles, re-seeds roles, reloads VFS: `SUCCESS: System reset complete` |
| `shutdown` | `SUCCESS: Server shutting down`; process exits after 500 ms |

**Roles.** Table `roles(name, commands)`; commands is comma-separated verbs, `*` = all. Seeded: `admin = *`, `guest = list,pwd,cd,help,whoami,quit,exit`. Unknown role falls back to the guest set. Roles are checked by verb only. Users table: `users(id, username UNIQUE, password_hash, role)`.


**Other replies.**
- Auth: `ERROR: Invalid username or password (password needs 8 or more characters)` on signup with a short password, and `ERROR: Too many failed logins. Try again later` once lockout kicks in.
- Limits: `ERROR: Line too long`, `ERROR: Server full` (refused at connect) and `ERROR: Server busy, try again` (the request could not be processed).
- Files: `ERROR: Invalid path` for paths that leave the root and `ERROR: Already exists: <path>` when `mkdir` targets an existing entry.
