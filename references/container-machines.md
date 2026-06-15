# Container machines — persistent Linux dev environments

A **container machine** is a persistent Linux environment that works seamlessly on your
Mac. Unlike a container (which is modeled after an *application* and is usually ephemeral),
a container machine is modeled after a *Linux box*: it runs the image's **init system**,
its filesystem **survives stop/start**, and it automatically maps **your host username and
home directory** into Linux. `container machine` has the alias `m`.

## Why a machine instead of a container

- **Edit on the Mac, build inside.** Your repo lives in `$HOME` on macOS and is mounted at
  the same home path inside the machine. Use your macOS editor/IDE; compile and run inside
  Linux. No copy step between "I built it" and "I'm inspecting it."
- **macOS-native tooling against Linux artifacts.** Profilers, browsers, GUI debuggers, and
  screenshot tools on the Mac all see the same files the machine sees.
- **Real Linux services.** Run a database or daemon under a process supervisor —
  `systemctl start postgresql` works on images that ship `systemd`.
- **One environment per target distro.** Spin up `alpine`, `ubuntu`, `debian` machines that
  all share the same `$HOME` and dotfiles, to test across distributions.

## Quickstart

```bash
container machine create alpine:latest --name dev    # creates AND boots
container machine run -n dev whoami                  # your host username, NOT root
container machine run -n dev pwd                      # your Mac home dir, mounted in
container machine run -n dev                          # interactive shell; cd into your repos
```

`container machine run` is how you get a shell or run one command; it boots the machine
first if it's stopped.

## Lifecycle

```bash
container machine create <image> [--name <n>] [--cpus <n>] [--memory <size>] \
    [--home-mount rw|ro|none] [--set-default] [--no-boot]
container machine run -n <n> [<cmd> [args...]]   # shell, or one command; -- separates args
container machine run -n <n> --root              # run as root instead of your host user
container machine ls                             # list all; DEFAULT column marks the default
container machine inspect <n>                    # JSON detail
container machine stop <n>
container machine rm <n>                          # delete, INCLUDING persistent storage
```

Set a default to drop the `-n` flag everywhere:

```bash
container machine set-default dev
container machine run            # operates on dev
```

## Resize CPU / memory / home-mount

`container machine set` writes config to disk; **changes take effect after the next
stop + start**, not immediately:

```bash
container machine set -n dev cpus=4 memory=8G
container machine stop dev
container machine run -n dev -- nproc      # boots with the new config
```

- **Memory** defaults to **half of host RAM**.
- **home-mount** is `rw` (default), `ro`, or `none`. Use `ro` to mount your Mac home
  read-only (protect host files from the Linux side); `none` to not mount it at all.

## Bring your own machine image

Any Linux image with `/sbin/init` works as a machine. For a full environment with
`systemd` and the usual CLI tools, build an Ubuntu image like this:

```dockerfile
FROM ubuntu:24.04
ENV container container

RUN apt-get update && \
    apt-get install -y \
      dbus systemd openssh-server net-tools iproute2 iputils-ping \
      curl wget vim-tiny man sudo && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    yes | unminimize

RUN >/etc/machine-id
RUN >/var/lib/dbus/machine-id

RUN systemctl set-default multi-user.target
RUN systemctl mask \
      dev-hugepages.mount sys-fs-fuse-connections.mount \
      systemd-update-utmp.service systemd-tmpfiles-setup.service \
      console-getty.service
RUN systemctl disable networkd-dispatcher.service

RUN sed -i -e 's/^AcceptEnv LANG LC_\*$/#AcceptEnv LANG LC_*/' /etc/ssh/sshd_config
```

```bash
container build -t local/ubuntu-machine:latest .
container machine create local/ubuntu-machine:latest --name ubuntu
```

### Customizing first-boot user setup

By default `container` runs a built-in setup script on first boot to provision the user
that matches your host account. To override it, add an **executable** script at
`/etc/machine/create-user.sh` in the image. It runs once, as root, on first boot, with
these environment variables set:

- `CONTAINER_USER`, `CONTAINER_UID`, `CONTAINER_GID`
- `CONTAINER_HOME`
- `CONTAINER_MACHINE_ID`

Use this when you need custom groups, shells, sudo rules, or extra per-user provisioning.

## VS Code remote workflow (edit on Mac, run in the machine)

The repo ships an `examples/container-machine-vscode/` sample. The idea: build a machine
image with `openssh-server`, create the machine, then attach VS Code's **Remote - SSH**
(or Dev Containers) to the running Linux environment so your editor runs on macOS while the
language server, compiler, and tests run in Linux against your mounted `$HOME`. Because the
home directory is shared, files you edit in VS Code are the exact files the machine builds.

## Gotchas

- `container machine rm` **deletes the persistent storage** — back up anything that lives
  only inside the machine (data outside your mounted `$HOME` won't be on the Mac).
- `set` changes are deferred until restart; tell the user to `stop` then `run` again.
- A machine needs an init system in the image (`/sbin/init`). A bare `alpine` boots fine
  for a shell, but for `systemctl`/services you need a systemd-enabled image as above.
