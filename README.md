# pi-cell

Run the **whole pi agent inside a bubblewrap sandbox** — an isolated filesystem,
process space, and network policy, with no Docker daemon and no VM.

- **Filesystem:** pi sees only a private tmpfs root with three host paths:
  the repo (`rw` → `/workspace`), a state volume (`rw` → `/state`), and
  read-only mounts of `/usr`, `/etc`, `/sys`, `/opt`, and your bun install.
  `~/.ssh`, `~/.aws`, your real `~/.pi/agent`, other users — all invisible.
- **Processes:** own PID/IPC/user namespaces; sandbox dies with the launcher.
- **Network:** shared namespace, but an **nftables ruleset restricted to the
  sandbox uid** allows only DNS (53), TLS (443), and the ports you configure
  (e.g. your database). Every other packet from the sandbox is dropped.
- **Credentials:** your host `~/.pi/agent/auth.json` is **bind-mounted live**
  into the sandbox (no copy; ACLs grant the sandbox user read). Keys are
  readable by pi by design — and egress is port-based (53 + 443 to any host),
  so the ruleset is **not** an exfiltration control. Use a proxy or a
  destination allowlist if that matters to you.

## Quickstart

```bash
# 1. install deps (bubblewrap, nftables, acl) — or let setup do it
sudo pi-cell setup --install

# 2. edit config (set REPO at minimum)
sudoedit /etc/pi-cell/config      # REPO=/path/to/your/repo, optional DB_ADDR/DB_PORT/EXTRA_PORTS

# 3. re-run setup so repo ACLs get applied
sudo pi-cell setup

# 4. launch pi inside the sandbox (interactive TUI)
sudo pi-cell run

# later: hand files back to you after a session, or remove the ruleset
sudo pi-cell chown
sudo pi-cell teardown             # revokes ACLs; --purge removes user + config
```

`check` runs without root: `pi-cell check`.

## Programmatic use

`run` is a normal command, so a script or an orchestrator can drive it:

```bash
sudo -n pi-cell run --cell build-1 -- -p "run the tests and fix what fails"
echo "exit=$?"          # pi's exit code
```

- **stdout is only the agent's output** — the launcher's own messages go to
  stderr, so `$(...)` and JSON consumers are safe.
- **`--cell NAME`** gives that run its own `HOME` (`/state/cells/NAME`): separate
  pi sessions, `auth.json` bind target and bun cache, so several cells can run
  at once. Without it `HOME` is `/state` — one long-lived cell, as before.
  Names must match `[A-Za-z0-9][A-Za-z0-9_.-]*`.
- **`--repo NAME`** binds `$REPO_ROOT/NAME` as `/workspace`, so one install
  serves many projects (one repo per job). `NAME` is a single directory
  component; `REPO_ROOT` is root-configured and setup applies the ACLs to it so
  job dirs created later inherit them.
- **`--prompt-file PATH`** takes the prompt from a file instead of argv: it is
  staged into the cell and never appears in the host's `/proc/*/cmdline`, and a
  prompt starting with `-` cannot parse as a flag.
- **`--timeout SECONDS`** kills the cell's process group (exit code 124) so a
  hung job cannot hold the queue.
- **`rm-cell NAME`** deletes one cell's `HOME` — needed because
  `/state/cells/*` is sandbox-owned and only root can prune it.
- **Nothing else comes from the caller.** Repo, mounts and egress policy stay
  root-configured on purpose: a user-chosen `--repo` would be a privilege
  expansion, not a convenience.
- **Cancel** by killing the `sudo`/`pi-cell` process group; `--die-with-parent`
  takes the sandbox down with it.
- **Passwordless spawning**: one narrow sudoers line, pointing at a root-owned,
  non-user-writable copy of the script — and never for `setup`/`teardown`/`chown`:
  `you ALL=(root) NOPASSWD: /usr/local/bin/pi-cell run`
- **Parallel cells on one repo** should each use a `git worktree`. They share
  the `pi-sandbox` uid and a single egress policy: they are separated by
  mount/pid/ipc namespaces and per-cell `HOME`, not by uid.

### From a container

A container cannot sudo, so it submits work through a directory it already has
mounted. `contrib/` is the host side of that: one dispatcher, two path units.

```
spool/<id>/job.prompt   -> start a cell for repo <id>   (serial, one at a time)
spool/<id>/cancel       -> stop that cell
```

```bash
sudo install -m 0755 contrib/spool/pi-cell-spool /usr/local/bin/
sudo install -m 0644 contrib/systemd/pi-cell-*.path \
                    contrib/systemd/pi-cell-*.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now pi-cell-start.path pi-cell-cancel.path
```

The result is not returned over a socket: the cell writes into its repo
(`status.json`, `render.mp4`), the same channel the caller already watches, and
the exit status lands in `journalctl -u pi-cell-job-<id>`. Nothing new is
exposed — the container only writes into a directory under `REPO_ROOT`.

## Trust boundary

| Path | Policy | Inside |
|---|---|---|
| your repo | **rw** | `/workspace` (ACLs let both you and `pi-sandbox` edit) |
| state volume (`/var/lib/pi-cell/state`) | **rw** | `/state` (pi sessions, extensions, bun cache, `HOME`) |
| `~/.bun`, `/usr`, `/usr/local`, `/opt`, `/etc`, `/sys` | **ro** | `/bun`, `/usr`, … (runtime + TLS certs) |
| `$HOME`, `/tmp`, `/dev/shm`, `/run` | tmpfs | ephemeral scratch |
| everything else on the host | — | **not visible** |

| Egress | Allowed? |
|---|---|
| DNS (53, udp+tcp) | yes |
| TLS (443) — LLM API, package registries, web tools | yes |
| configured DB port / extra ports | yes (from `DB_ADDR`/`DB_PORT`/`EXTRA_PORTS`) |
| anything else from the sandbox uid | **dropped** (nftables) |

Inbound cannot be filtered per-uid on a shared netns (and a `policy drop` there
would firewall the whole host), so it is left alone; the reverse channel is
closed on the *output* side instead: replies are only accepted on flows the
sandbox uid itself started, plus loopback (so you can still reach a dev server
pi starts on `127.0.0.1:PORT`). Someone who connects *in* gets no usable
channel back.

## It's just volumes

Your instinct is right — this is the Docker volume model, minus the daemon:

| Docker | pi-cell (bwrap) |
|---|---|
| `docker run -v $PWD:/workspace` | `--bind "$REPO" /workspace` |
| named volume (`-v data:/data`) | `--bind /var/lib/pi-cell/state /state` |
| image layers | read-only binds (`/usr`, `/etc`, `~/.bun`) |
| daemon + registry + layers | none — kernel namespaces |

`pi` runs with cwd `/workspace` (the directory you passed in) and `HOME=/state`
(its volume). Files it writes under `/workspace` land in your repo; files under
`/state` land in the state volume. Everything else is a fresh tmpfs each launch.

## Credentials — a live bind-mount, no copies

`pi` reads credentials from `~/.pi/agent/auth.json`, and the sandbox's
`$HOME` is `/state` — so pi-cell **bind-mounts your host's real `auth.json`
straight into the sandbox** at `/state/.pi/agent/auth.json`. There is no copy:
the sandbox sees the very same file you do, live. Edit keys on the host and
the next sandbox command sees them immediately.

Why this needs ACLs: the sandbox runs as the `pi-sandbox` user, but
`~/.pi/agent` is `0700` and `auth.json` is `0600` owned by you. On every run
pi-cell grants the sandbox user two ACLs (both harmless — only root can
invoke `pi-sandbox`, and root already has your keys):

```bash
setfacl -m u:pi-sandbox:x  ~/.pi/agent          # traverse the 0700 dir
setfacl -m u:pi-sandbox:r  ~/.pi/agent/auth.json # read the 0600 file
```

The grant is refreshed each `run` because pi rewrites auth.json with
`chmod(0600)` (verified in pi's source: `writeFileSync` + `chmodSync`), which
zeroes the ACL mask. If you ever replace the file by hand (some editors do an
atomic rename, which also breaks a file bind), re-run `sudo pi-cell setup` or
`sudo pi-cell grant-auth`.

`AUTH_MODE=ro` (default) = sandbox reads your keys live and can't touch them.
`AUTH_MODE=rw` = sandbox can also write through (note: pi's in-sandbox
`chmod` will fail with EPERM since it doesn't own the file, so keep `ro`
unless you have a specific need). If there is no host `auth.json`, pi can
`/login` inside the sandbox and those keys live in the state volume only.
The optional `secrets.env` exists solely for providers you configure via
environment variables on the host — normally you can ignore it.

## Notes & gotchas

- **Ownership:** the sandbox runs as the system user `pi-sandbox`. Files it
  creates are owned by `pi-sandbox`. Setup installs mutual ACLs (and the
  sandbox uses `umask 002`) so you and pi can edit each other's files; to hand
  the repo back under your name: `sudo pi-cell chown` (state volume stays
  sandbox-owned — delete `$STATE_DIR` to reset it).
- **bun caches:** host `~/.bun` is mounted read-only, so `setup` writes a
  `bunfig.toml` into the state volume pointing bun's cache at `/state/bun-cache`.
- **No Node:** this machine has no system node; JS tooling runs via bun
  (`bunx`, `bun --node`). Install other toolchains *inside* the sandbox
  (persisted in `/state`) or add them to the ro mounts.
- **Credentials are readable by pi** — that's inherent (it needs them to call
  the LLM). Egress is *port*-based (53 + 443 to any destination), so an injected
  agent can still POST them anywhere: documented limit, not a bug. Values from
  `secrets.env` are bind-mounted read-only at `/run/secrets.env` and sourced by
  the launcher, so they never show up in `ps` / `/proc/*/cmdline`.
- **The repo is the real exit.** Anything pi writes under `/workspace` runs as
  *you* later: `.git/config` (`core.fsmonitor`, `core.pager`, `diff.external`),
  `.git/hooks/*`, `package.json` scripts, `.vscode/tasks.json`. Review the diff
  before you run repo tooling in that tree.
- **Nested user namespaces** are blocked (`--disable-userns`) because that is the
  main kernel-LPE surface. Set `ALLOW_NESTED_USERNS=1` in the config to run an
  inner bwrap-based sandbox (e.g. Claude Code's Linux sandbox) inside the cell.
- **DNS via Tailscale MagicDNS** works out of the box (it's just port 53).
- **TUI & Ctrl-C** work through bwrap; if you ever hit a signal quirk, drop
  `--unshare-pid` in the `run` command.
- **Extensions:** `~/.pi/agent` inside the sandbox starts fresh in the cell's
  `HOME` (`/state`, or `/state/cells/NAME` with `--cell`). Reinstall/copy your
  extensions there if you need them.

## Upgrade path

The launcher's sandbox backend is one layer. When you want defense-in-depth
syscall filtering (the whole pi process has the full syscall surface inside),
swap the bwrap exec for **hakoniwa** (Rust: namespaces + seccomp + cgroups +
Landlock in one crate) — same mounts, same policy, typed API. The nftables
ruleset and config don't change.
