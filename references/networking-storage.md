# Networking & storage

How to reach containers, wire up networks/DNS, and persist data with `container`. The
recurring theme: each container is its own VM with its own IP on a virtual network, so the
host↔container and container↔container paths differ from Docker-on-a-shared-VM.

## Reaching a container from the host

Three ways, in rough order of preference:

1. **Publish a port** (most portable, works on macOS 15 and 26):

   ```bash
   container run -d --rm -p 8080:80 nginx:latest         # localhost:8080 -> container:80
   container run -d --rm -p 127.0.0.1:8080:8000 node:latest ...   # bind to a specific host IP
   container run -d --rm -p '[::1]:8080:8000' node:latest ...     # IPv6 loopback
   ```
   Spec: `[host-ip:]host-port:container-port[/tcp|udp]`. If a container is on multiple
   networks, published ports forward to the interface on the *first* network.

2. **Container IP** directly — `container ls` shows it (e.g. `192.168.64.3`); browse to it.
3. **Local DNS domain** (nice hostnames). Create a domain once (needs sudo):

   ```bash
   sudo container system dns create test
   container run -d --name my-web-server --rm web-test
   # now my-web-server.test resolves to the container IP
   ```
   Or set it as the default in `~/.config/container/config.toml` under `[dns] domain = "test"`.

## User-defined networks (macOS 26+ only)

`container system start` creates a `default` vmnet network. Create isolated networks with:

```bash
container network create foo
container network create foo --subnet 192.168.100.0/24 --subnet-v6 fd00:1234::/64
container network list
container run -d --name web --network foo --rm web-test
container network delete foo            # once no containers are attached
```

Networks are **mutually isolated** — a container on `foo` cannot reach one on `default`.
Default subnets for new networks can be set in `config.toml`:

```toml
[network]
subnet = "192.168.100.1/24"
subnetv6 = "fd00:abcd::/64"
```

> On **macOS 15** the `container network` subcommands don't exist, `--network` errors, all
> containers share the single `default` network, and **container-to-container traffic is
> blocked**. Multi-container networking requires macOS 26.

## Access a host service from a container

macOS security constraints make this finicky (creating a localhost domain disables Private
Relay, and the packet-filter rule is dropped on restart). Map a host-reachable IP to a
domain name used inside containers:

```bash
sudo container system dns create host.container.internal --localhost 203.0.113.113
container run -it --rm alpine/curl curl http://host.container.internal:8000
```

Pick an `--localhost` IP unlikely to collide: documentation ranges `192.0.2.0/24`,
`198.51.100.0/24`, `203.0.113.0/24`, or the `172.16.0.0/12` private range.

## Custom MAC address

```bash
container run --network default,mac=02:42:ac:11:00:02 ubuntu:latest
```

Format `XX:XX:XX:XX:XX:XX` (colons or hyphens). Set the low two bits of the first octet to
`10` (locally administered, unicast). If unset, `container` generates one whose first
nibble is `f` (`fX:...`), so choosing a non-`f` first nibble avoids collisions.

## SSH agent forwarding

`--ssh` mounts the macOS SSH auth socket into the container, so you can clone private repos
with passwordless auth. It's equivalent to
`-v "${SSH_AUTH_SOCK}:/run/host-services/ssh-auth.sock" -e SSH_AUTH_SOCK=/run/host-services/ssh-auth.sock`,
but it also re-points the socket automatically across logout/login + container restart.

```bash
container run -it --rm --ssh alpine:latest sh
# inside: ssh-add -l   # lists your host keys
```

## Volumes (persistent storage)

Volumes outlive containers. Create explicitly, or implicitly via a `-v` reference.

```bash
container volume create myvol                       # named volume
container volume create --opt journal=ordered myvol # ext4 journaling: ordered (safe default)
container volume create --opt journal=writeback:64m myvol   # fastest, least safe, 64M journal
container volume create --opt journal=journal --opt size=10g myvol  # safest, explicit size
container run -v myvol:/var/lib/postgresql/data postgres:16
container volume ls
container volume inspect myvol
container volume rm myvol            # fails if still in use
container volume prune               # remove all unreferenced volumes; reports space freed
```

Journaling modes (ext4): `ordered` (metadata journaled, data-before-metadata — safe
default), `writeback` (metadata only, no data ordering — fastest), `journal` (data +
metadata — safest, highest write amplification). Optional `:<size>` sets journal size.

### Anonymous volumes — the Docker trap

`-v /path` or `--mount type=volume,dst=/path` (no source) creates an **anonymous** volume
named `anon-<uuid>`. Unlike Docker, these do **NOT** auto-clean with `--rm` — you must
delete them yourself:

```bash
container run -v /data alpine          # creates anon-<uuid>
VOL=$(container volume list -q | grep anon)
container run -v $VOL:/data alpine     # reuse it
container volume rm $VOL                # manual cleanup
```

## Bind mounts (share host files)

```bash
container run -v ${HOME}/Desktop/assets:/content/assets docker.io/python:alpine ls /content/assets
# equivalent --mount form:
container run --mount source=${HOME}/Desktop/assets,target=/content/assets,readonly python:alpine ...
```

`--volume` is `host-path:container-path[:ro]`. `--mount` is comma-separated `key=value`
(`type=`, `source=`/`src=`, `target=`/`dst=`, `readonly`). Bind mounts share host data into
the VM and persist it across runs.

## tmpfs and shared memory

```bash
container run --tmpfs /scratch alpine ...     # in-memory mount at /scratch
container run --shm-size 1G postgres:16        # size of /dev/shm (default may be small)
```
