---
name: apple-container
description: >-
  Expert guidance for Apple's `container` CLI. the open-source tool for running,
  building, and managing Linux containers and persistent Linux "container machines"
  as lightweight VMs on Apple silicon Macs. Use when the user wants to run containers
  or Linux dev environments on a Mac with Apple `container`, replace Docker Desktop,
  Colima, Lima, or OrbStack, use commands like `container run`, `container build`,
  `container machine create`, or `container system start`, or configure ports,
  volumes, networks, registries, DNS, or container networking on macOS. Also use for
  diagnosing Apple `container` runtime, VM, port, DNS, or networking issues. Do not
  use for Docker Desktop, Podman, Kubernetes-specific workflows, or generic Dockerfile
  authoring unrelated to Apple `container` on Mac.

---

# Apple `container` expert

`container` is Apple's open-source CLI (Swift, Apple-silicon-native) that runs **each
Linux container inside its own lightweight virtual machine**, and consumes/produces
standard OCI images. It is the Mac-native alternative to Docker Desktop, Colima, Lima,
and OrbStack. Repo: https://github.com/apple/container — latest release line: 1.x.

Your job with this skill is to be a precise, hands-on expert: give the user the exact
commands, surface the Mac-specific gotchas they won't find in generic Docker tutorials,
and never invent flags. When a detail isn't in this file, read the matching reference
in `references/` rather than guessing.

## The one mental model that explains everything

`container` is **not** Docker-on-a-shared-VM. Generic container tools boot one big Linux
VM and pack every container into it. `container` boots a **separate micro-VM per
container**. Internalize this because it explains nearly every difference a Docker user
will trip over:

- **Isolation & security** come from the VM boundary, so each container is strongly isolated.
- **`0.0.0.0` inside a container is safe** — external machines can't reach the container's
  virtual network — but it also means containers get their own IPs (e.g. `192.168.64.3`)
  rather than sharing the host's localhost by default. Reaching a container from the host
  is via that IP, an optional local DNS domain, or `--publish`.
- **Memory is allocated per VM** and (today) freed memory isn't fully returned to macOS;
  long-lived memory-heavy containers may need an occasional restart.
- Two products share the engine: **containers** (app-shaped, usually ephemeral) and
  **container machines** (persistent Linux environments). Pick the right one — see below.

## CRITICAL: check the macOS version first — it gates real functionality

`container` is *designed for and supported on macOS 26+*. It runs on macOS 15 but with
limitations the maintainers will not fix. Before giving networking-heavy advice, know
which the user is on (`sw_vers -productVersion`). On **macOS 15**:

- The `container network` subcommands are **unavailable**; `--network` on `run`/`create`
  errors. Every container attaches to the single `default` vmnet network.
- **Container-to-container networking does not work** — containers are isolated from each
  other on the virtual network (this breaks the "curl one container from another" pattern).
- Container IP assignment is fragile (the network is created when the first container
  starts), so containers can occasionally come up with no network.
- Use `--publish` to reach a service from the host; don't rely on container DNS or
  cross-container calls.

On **macOS 26+** all of the above works: user-defined isolated networks, container-to-
container traffic, stable IPs. When the user is on 15 and wants multi-container networking,
tell them plainly that it needs macOS 26 — don't hand them commands that will fail.

## Step 0 — is it installed and running?

Almost every task fails at the same place: the tool isn't installed or the system service
isn't started. Check, and guide setup, before anything else:

```bash
command -v container && container --version    # installed?
container system status                         # service up?
```

- **Not installed:** download the signed `.pkg` from
  https://github.com/apple/container/releases (latest 1.x), double-click, enter admin
  password (installs under `/usr/local`). Requires an Apple-silicon Mac.
- **Installed but not started:** `container system start`. On first run it offers to
  download a recommended Linux kernel — accept it (`Y`) or the engine has nothing to boot.
- Upgrades/downgrades go through `/usr/local/bin/update-container.sh` (stop the service
  first with `container system stop`). Uninstall: `/usr/local/bin/uninstall-container.sh`
  (`-k` keeps user data, `-d` deletes it).

Don't fabricate a Homebrew formula or a `brew install container` line — the supported path
is the signed installer. If you can't verify the install, say so and give the install steps.

## Decision: container vs. container machine

This is the most important choice you'll help the user make. Get it right and the rest
follows.

| Use a **container** (`container run`) when… | Use a **container machine** (`container machine create`) when… |
|---|---|
| Running a service or app: a web server, Postgres, Redis, a build job | You want a **persistent Linux dev environment** that survives stop/start |
| The unit of work is *one application* | The unit of work is *a Linux box* you live in |
| You want it ephemeral (`--rm`) or detached (`-d`) | You want your **Mac `$HOME` mounted in** and your **host user**, not root |
| You're reproducing a datacenter workload locally | You want to edit code in a Mac IDE and compile/run inside Linux, or run real services under `systemd`/`init` |

A container machine boots the image's **init system**, maps your **username and home
directory** into Linux, and persists its filesystem. It's the answer to "I want a Linux
machine on my Mac," not "I want to run nginx." Full guide: `references/container-machines.md`.

## Core workflows (the 80%)

### Run a container

```bash
# interactive shell
container run -it --rm ubuntu:latest /bin/bash

# detached service, publish a host port to the container port
container run -d --name web -p 8080:80 --rm nginx:latest

# resources, env, and a persistent named volume
container run -d --name db \
  -e POSTGRES_PASSWORD=secret \
  --cpus 4 --memory 4G \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 postgres:16
```

Defaults per container: 4 CPUs, 1 GiB RAM (overridable with `--cpus`/`--memory` or in
`config.toml`). `--rm` removes the container on exit; `-d` detaches; `-it` gives an
interactive TTY. Reach a detached service from the host via the published port
(`localhost:8080`) or the container's IP (`container ls` shows it).

### Build an image

```bash
container build -t my-app:latest .                       # Dockerfile or Containerfile in .
container build -f docker/Dockerfile.prod -t my-app:prod .
container build --arch arm64 --arch amd64 -t my-app .     # multi-platform (amd64 runs via Rosetta)
```

The first build silently starts a **builder VM** (BuildKit, 2 CPU / 2 GiB by default).
For heavy builds, pre-size it: `container builder start --cpus 8 --memory 16G`. To resize a
running builder you must `container builder stop && container builder delete` then start it
again.

### Create a container machine

```bash
container machine create alpine:latest --name dev      # boots it
container machine run -n dev                            # shell as YOUR host user, $HOME mounted
container machine run -n dev -- uname -a                # run one command
container machine set-default dev                       # then drop -n
container machine set -n dev cpus=4 memory=8G           # takes effect after stop+start
```

### Manage what's running

```bash
container ls -a                 # list (default: running only; -a includes stopped)
container exec -it web sh       # shell into a running container
container logs -f web           # follow stdout; add --boot for VM/init boot logs
container stats                 # live top-like resource view
container stop web && container rm web
container system df             # disk used by images/containers/volumes
```

## Going deeper — read the reference that matches the task

Keep this file as the map; load the detail on demand so you stay accurate instead of
recalling flags from memory:

- **`references/command-reference.md`** — the complete CLI surface: every subcommand,
  flag, and argument for `run`/`create`/`build`/`exec`/`image`/`network`/`volume`/
  `registry`/`machine`/`system`/`builder`, with the exact option formats (mount syntax,
  `--publish` spec, capabilities, `--ulimit`, etc.). Go here whenever you need a precise
  flag or aren't 100% sure one exists.
- **`references/container-machines.md`** — persistent Linux dev environments: lifecycle,
  home-mount modes, bring-your-own image (systemd-enabled Dockerfile), the
  `/etc/machine/create-user.sh` first-boot hook, and the VS Code remote workflow.
- **`references/networking-storage.md`** — publishing ports (IPv4/IPv6), user-defined
  networks (macOS 26+), local DNS domains, reaching a host service from a container,
  custom MAC addresses, `--ssh` agent forwarding, named/anonymous volumes and ext4
  journaling, and bind mounts.
- **`references/config-and-troubleshooting.md`** — `~/.config/container/config.toml`
  schema (default CPU/RAM, registry domain, DNS domain, kernel, subnets), Rosetta toggle,
  nested virtualization, the memory-ballooning caveat, and a symptom→fix troubleshooting
  table (including the macOS 15 networking failures).

## How to behave as this expert

- **Verify before prescribing.** When relevant, check install state, service status, and
  macOS version rather than assuming. A command that errors because the service is down
  wastes the user's time and trust.
- **Don't invent flags or a Homebrew install.** If you're unsure a flag exists, open
  `references/command-reference.md`. The supported install is the signed `.pkg`.
- **Translate Docker honestly.** `container run` mirrors `docker run` closely (`-it`,
  `-d`, `-p`, `-v`, `-e`, `--rm`, `--name` all work), so port Docker commands directly —
  but call out the real differences: per-container VMs, per-container IPs, the macOS-26
  networking gate, no Docker-Compose (run/script the services yourself), and anonymous
  volumes that do **not** auto-remove with `--rm`.
- **Pick container vs. machine deliberately** and say why in one line, so the user learns
  the distinction instead of just getting commands.
- **Prefer copy-pasteable command blocks** with brief inline explanation over prose. Show
  the cleanup step (`stop`/`rm`/`prune`) when you start something long-lived.
