# `container` CLI — complete command reference

Condensed from the official `apple/container` docs (release line 1.x). Command and flag
availability can vary by macOS version (notably, `container network` needs macOS 26+).
`container --help` and `container <cmd> --help` are authoritative on the installed version.

Global: `container --debug <subcommand>`, `container --version`, and
`container --generate-completion-script [zsh|bash|fish]`.

## Table of contents
- [Core: run, build](#core)
- [Container lifecycle: create, start, stop, kill, delete, list, exec, export, logs, inspect, stats, copy, prune](#container-lifecycle)
- [Images: list, pull, push, save, load, tag, delete, prune, inspect](#image-management)
- [Builder: start, status, stop, delete](#builder-management)
- [Networks (macOS 26+): create, delete, prune, list, inspect](#network-management-macos-26)
- [Volumes: create, delete, prune, list, inspect](#volume-management)
- [Registry: login, logout, list](#registry-management)
- [Container machines: create, run, list, inspect, set, set-default, logs, stop, delete](#container-machine-management)
- [System: start, stop, status, version, logs, df, dns, kernel, property](#system-management)

---

## Core

### `container run [<options>] <image> [<args>...]`
Run a container from an image. Foreground by default; stdin closed unless `-i`. If a
command is given it runs instead of the image default.

Process: `-e/--env key=value` (or bare `key` to inherit from host), `--env-file <f>`,
`--gid`, `-i/--interactive`, `-t/--tty`, `-u/--user name|uid[:gid]`, `--uid`,
`--ulimit <type>=<soft>[:<hard>]`, `-w/--workdir/--cwd <dir>`.

Resources: `-c/--cpus <n>`, `-m/--memory <size>` (1 MiB granularity, suffixes K/M/G/T/P).

Management: `-a/--arch <arch>` (default arm64), `--cap-add <cap>`, `--cap-drop <cap>`
(name with or without `CAP_`, case-insensitive, or `ALL`), `--cidfile <path>`,
`-d/--detach`, `--dns <ip>`, `--dns-domain`, `--dns-option`, `--dns-search`,
`--entrypoint <cmd>`, `--init` (lightweight PID 1 that reaps zombies & forwards signals),
`--init-image <image>` (custom VM init filesystem), `-k/--kernel <path>`,
`-l/--label key=value`, `--mount type=<>,source=<>,target=<>[,readonly]`, `--name <id>`,
`--network <name>[,mac=XX:XX:XX:XX:XX:XX][,mtu=VALUE]`, `--no-dns`, `--os <os>` (default
linux), `-p/--publish [host-ip:]host-port:container-port[/tcp|udp]`,
`--platform os/arch[/variant]` (beats `--os`/`--arch`),
`--publish-socket host_path:container_path`, `--read-only`, `--rm/--remove`, `--rosetta`,
`--runtime <handler>` (default container-runtime-linux), `--ssh` (forward host SSH agent
socket), `--shm-size <size>`, `--tmpfs <path>`, `-v/--volume <src>:<dst>[:ro]`,
`--virtualization` (nested virt; needs M3+ and a virt-capable kernel).

Registry: `--scheme http|https|auto` (auto = HTTP for loopback/RFC1918/local DNS domain,
HTTPS otherwise). Progress: `--progress auto|none|ansi|plain|color`. Fetch:
`--max-concurrent-downloads <n>` (default 3).

```bash
container run -it ubuntu:latest /bin/bash
container run -d --name web -p 8080:80 nginx:latest
container run -e NODE_ENV=production --cpus 2 --memory 1G node:18
container run --init ubuntu:latest my-app
```

### `container build [<options>] [<context-dir>]`
Build an OCI image with BuildKit (in the builder VM). Looks for `Dockerfile` then
`Containerfile` if no `-f`.

`-a/--arch <v>` (repeatable for multi-arch), `--build-arg key=val`, `-c/--cpus <n>`
(default 2), `--dns*` (as above), `-f/--file <path>`, `-l/--label key=val`,
`-m/--memory <size>` (default 2048MB), `--no-cache`,
`-o/--output type=<oci|tar|local>[,dest=]` (default type=oci), `--os <v>`,
`--platform os/arch[/variant]`, `--progress auto|plain|tty`, `--pull`, `-q/--quiet`,
`--secret id=<key>[,env=<ENV>|,src=<path>]`, `-t/--tag <name>` (repeatable),
`--target <stage>`, `--vsock-port <port>` (default 8088).

```bash
container build -t my-app:latest .
container build --target production --no-cache -t my-app:prod .
container build --arch arm64 --arch amd64 -t my-app .
```

---

## Container lifecycle

### `container create [<options>] <image> [<args>...]`
Same process/resource/management/registry/fetch flags as `run`, but leaves the container
**stopped**. Start later with `container start`.

### `container start [-a/--attach] [-i/--interactive] <id>`
Start a stopped container; optionally attach stdout/stderr and stdin.

### `container stop [-a/--all] [-s/--signal <sig>] [-t/--time <sec>] [<ids>...]`
Graceful stop (default SIGTERM, then SIGKILL after `--time`, default 5s). Without ids,
nothing stops unless `--all`.

### `container kill [-a/--all] [-s/--signal <sig>] [<ids>...]`
Immediate kill (default KILL). No graceful shutdown.

### `container delete | rm [-a/--all] [-f/--force] [<ids>...]`
Delete containers. `--force` removes running ones. Needs ids or `--all`.

### `container list | ls [-a/--all] [--format json|table|yaml|toml] [-q/--quiet]`
List containers (running only unless `-a`). `-q` prints only ids.

### `container exec [options] <id> <args>...`
Run a command in a running container. Same process flags as `run` plus `-d/--detach`.
`container exec -it web sh`.

### `container export [-o <output>] <id>`
Export a **stopped** container's filesystem as a tar (stdout if no `-o`).

### `container logs [--boot] [-f/--follow] [-n <lines>] <id>`
Container stdio logs; `--boot` shows the VM boot/init log instead.

### `container inspect <ids>...`
Detailed JSON per container.

### `container stats [--format ...] [--no-stream] [<ids>...]`
Live top-like resource view; `--no-stream` for a single snapshot. Metrics: CPU %
(~100% = one core, can exceed 100%), memory used/limit, net Rx/Tx, block I/O, pid count.

### `container copy | cp <source> <destination>`
Copy between host and a **running** container. One side is `container_id:/path`.
`container cp ./config.json web:/etc/app/`.

### `container prune`
Remove stopped containers; reports space freed.

---

## Image management

`container image <sub>` (alias `container i`):
- `list | ls [--format ...] [-q] [-v]` — local images.
- `pull [--scheme ...] [--progress ...] [--max-concurrent-downloads <n>] [-a/--arch] [--os] [--platform] <ref>`
- `push [--scheme ...] [--progress ...] [-a/--arch] [--os] [--platform] <ref>`
- `save [-a/--arch] [--os] -o/--output <file> [--platform] <refs>...` — image → tar.
- `load -i/--input <file> [-f/--force]` — tar → images.
- `tag <source> <target>` — add a new reference; original stays.
- `delete | rm [-a/--all] [-f/--force] [<images>...]` — can't delete images used by a container.
- `prune [-a/--all]` — remove dangling (or with `-a` all unreferenced) images.
- `inspect <images>...` — detailed JSON.

---

## Builder management

`container builder <sub>`:
- `start [-c/--cpus <n>] [-m/--memory <size>] [--dns* ...]` — start BuildKit builder (default 2 CPU / 2048MB).
- `status [--format ...] [-q]`
- `stop`
- `delete | rm [-f/--force]`

To resize: `builder stop` → `builder delete` → `builder start --cpus N --memory M`.

---

## Network management (macOS 26+)

Not available on macOS 15. `container network <sub>`:
- `create [--internal] [--label ...] [--option key=value ...] [--plugin <p>] [--subnet <cidr>] [--subnet-v6 <cidr>] <name>`
  (`--internal` = host-only; default plugin `container-network-vmnet`).
- `delete | rm [-a/--all] [<names>...]`
- `prune` — remove networks with no containers (default/system networks preserved).
- `list | ls [--format ...] [-q]`
- `inspect <networks>...`

Running `container system start` creates the `default` vmnet network. Different networks
are mutually isolated.

---

## Volume management

Volumes are created explicitly or implicitly via `-v name:/path` (named) or `-v /path`
(anonymous, `anon-<uuid>`). Anonymous volumes do **not** auto-remove with `--rm`.

`container volume <sub>`:
- `create [--label ...] [--opt key=value ...] [-s <size>] <name>` — driver `local`.
  Driver opts: `size=<v>` (min 1 MiB; `-s` wins if both given),
  `journal=<ordered|writeback|journal>[:<size>]` (ext4 journaling mode:
  ordered = metadata-only, safe default; writeback = fastest, least safe;
  journal = data+metadata, safest).
- `delete | rm [-a/--all] [<names>...]` — can't delete a volume in use.
- `prune` — remove unreferenced volumes; reports space reclaimed.
- `list | ls [--format ...] [-q]`
- `inspect <names>...`

```bash
container volume create --opt journal=ordered myvol
container run -v myvol:/data alpine
```

---

## Registry management

`container registry <sub>` (alias `container r`):
- `login [--scheme ...] [--password-stdin] [-u/--username <user>] <server>`
- `logout <registry>`
- `list [--format ...] [-q]`

Credentials are stored (macOS Keychain). Default registry domain is `docker.io`
(configurable under `[registry]` in `config.toml`).

---

## Container machine management

`container machine <sub>` (alias `m`). Persistent Linux environments — see
`container-machines.md`.

- `create [<options>] <image>` — create and boot. `-n/--name <name>`, `--set-default`,
  `--no-boot`, `--cpus <n>`, `--memory <size>` (default: half system RAM),
  `--home-mount ro|rw|none` (default rw), plus `-a/--arch` (default host arch), `--os`,
  `--platform`, `--scheme`, `--progress`, `--max-concurrent-downloads`.
- `run [<options>] [<executable>] [<args>...]` — boot if needed, run a command or open a
  login shell. `-n/--name <id>` (default machine if omitted), `-d/--detach`, `--root`
  (run as root instead of matching host user), plus the standard process flags
  (`-e`,`--env-file`,`--gid`,`-i`,`-t`,`-u`,`--uid`,`-w`).
- `list | ls [--format json|table] [-q]` — DEFAULT column marks the default machine.
- `inspect [<id>]` — JSON (default machine if omitted).
- `set [-n/--name <name>] <key=value>...` — `cpus=<n>`, `memory=<size>`,
  `home-mount=ro|rw|none`. **Takes effect after stop + restart.**
- `set-default <id>`
- `logs [--boot] [-f/--follow] [-n <lines>] [<id>]`
- `stop [<id>]`
- `delete | rm <id>` — stops first; deletes persistent storage.

```bash
container machine create --cpus 4 --memory 8G --set-default alpine:3.22
container machine run -n my-machine -- cat /proc/cpuinfo
```

---

## System management

macOS-host-only. `container system <sub>` (alias `container s`):

- `start [-a/--app-root <p>] [--install-root <p>] [--log-root <p>] [--enable-kernel-install|--disable-kernel-install] [--timeout <sec>]`
  — start apiserver + helpers; prompts to install the recommended kernel on first run.
- `stop [-p/--prefix <prefix>]` — stop services (default launchd prefix `com.apple.container.`).
- `status [-p/--prefix ...] [--format ...]` — health check + basic system info.
- `version [--format ...]` — CLI and (if up) apiserver version. Columns: COMPONENT, VERSION, BUILD, COMMIT.
- `logs [-f/--follow] [--last <m|h|d>]` — service logs (default last 5m).
- `df [--format ...]` — disk usage for images/containers/volumes (count, active, size, reclaimable).
- `dns create [--localhost <ipv4>] <domain>` — local DNS domain (needs `sudo`).
- `dns delete | rm <domain>` — needs `sudo`.
- `dns list | ls [--format ...] [-q]`
- `kernel set [--arch amd64|arm64] [--binary <path>] [--force] [--recommended] [--tar <path|url>]`
  — install/update guest kernel. `--recommended` downloads & installs the default (wins over other flags).
- `property list | ls [--format json|toml]` — list system properties with current values.

```bash
container system start
sudo container system dns create test          # my-web-server -> my-web-server.test
container system df
```
