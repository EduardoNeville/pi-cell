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
  into the sandbox (no copy; ACLs grant the sandbox user read); the egress
  ruleset — not key placement — is the exfiltration control.

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
sudo pi-cell teardown             # add --purge to also remove user + config
```

`check` runs without root: `pi-cell check`.

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

Inbound: loopback accepted (so you can reach a dev server pi starts on
`127.0.0.1:PORT`), established/related replies accepted, everything else to
the sandbox uid dropped.

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
  the LLM). The egress ruleset is what prevents exfiltration; don't disable it.
- **DNS via Tailscale MagicDNS** works out of the box (it's just port 53).
- **TUI & Ctrl-C** work through bwrap; if you ever hit a signal quirk, drop
  `--unshare-pid` in the `run` command.
- **Extensions:** `~/.pi/agent` inside the sandbox starts fresh in `/state`.
  Reinstall/copy your extensions there if you need them.

## Upgrade path

The launcher's sandbox backend is one layer. When you want defense-in-depth
syscall filtering (the whole pi process has the full syscall surface inside),
swap the bwrap exec for **hakoniwa** (Rust: namespaces + seccomp + cgroups +
Landlock in one crate) — same mounts, same policy, typed API. The nftables
ruleset and config don't change.
