# Configuration & troubleshooting

## `config.toml` — system defaults

The `container` service reads TOML at startup, first-match-wins precedence:

1. User file: `~/.config/container/config.toml`
2. Optional install file: `<installRoot>/etc/container/config.toml`
3. Hardcoded defaults for any missing key

**The service reads the file once at startup — restart it after edits:**
`container system stop && container system start`. Inspect current effective values with
`container system property list`.

```toml
[build]                 # builder VM used by `container build`
rosetta = true          # use Rosetta for non-native arch during builds (set false to force native arm64)
cpus = 2
memory = "2048mb"
# image = "ghcr.io/apple/container-builder-shim/builder:<tag>"

[container]             # defaults for `container run` / `create` without --cpus/--memory
cpus = 4
memory = "1g"

[dns]
domain = "test"         # appends to container hostnames: my-web-server -> my-web-server.test (unset by default)

[kernel]                # guest kernel; defaults bump per release
# binaryPath = "opt/kata/share/kata-containers/vmlinux-<ver>"   # path INSIDE the kernel archive
# url = "https://github.com/kata-containers/kata-containers/releases/download/<ver>/kata-static-<ver>-arm64.tar.zst"

[network]               # default subnets for new networks (macOS 26+)
subnet = "192.168.100.0/24"
subnetv6 = "fd00:abcd::/64"

[registry]
domain = "docker.io"    # assumed when an image ref omits the host (alpine -> docker.io/library/alpine)

[vminit]
# image = "ghcr.io/apple/containerization/vminit:<tag>"

# [plugin.<id>]          # plugin-scoped config; schema defined by each plugin
```

### MemorySize format
Quoted string, numeric prefix + **binary** unit suffix (powers of 1024, even when written
`kb`/`mb`/`gb`): `b`, `k|kb|kib`, `m|mb|mib`, `g|gb|gib`, `t|tb|tib`, `p|pb|pib`. A bare
integer parses as bytes. Case-insensitive. `"2g"` re-emits as `"2gb"`.

### CIDR format
Quoted: IPv4 `"192.168.100.0/24"`, IPv6 `"fd00:abcd::/64"`. Invalid CIDRs are rejected at load.

## Rosetta & multi-arch

- `container build --arch arm64 --arch amd64 -t img .` builds multi-platform; the amd64
  variant runs under **Rosetta** translation on Apple silicon.
- To forbid x86_64 emulation in builds, set `[build] rosetta = false`.
- `--rosetta` on `run`/`create` enables Rosetta inside that container.

## Nested virtualization

`--virtualization` on `run`/`create` exposes virt to the container. Requires an **M3 or
newer** Mac and a virt-capable guest kernel (`-k/--kernel <path>`). Without support you'll
see `Error: ... nested virtualization is not supported on the platform`. Verify with
`dmesg | grep kvm` inside the container.

## Memory ballooning caveat

The macOS Virtualization framework only partially supports memory ballooning. A container's
VM uses only what the app needs (so `--memory 16g` may show ~2 GiB in Activity Monitor), but
**memory freed inside the guest is not returned to macOS**. Long-lived, memory-heavy
containers may need an occasional restart to release host memory.

## Logs & diagnostics

```bash
container logs <id>            # app stdout/stderr
container logs --boot <id>     # VM boot + init (vminitd) log — start here for boot failures
container system logs -f       # service-level logs (apiserver, runtime, network helpers)
container system logs --last 1h
container inspect <id> | jq    # full container state, IP, mounts, resources
container system status        # is the service up?
container system df            # disk used / reclaimable by images, containers, volumes
```

## Troubleshooting table (symptom → cause → fix)

| Symptom | Likely cause | Fix |
|---|---|---|
| `container: command not found` | Not installed | Install the signed `.pkg` from the releases page; it lands in `/usr/local/bin`. |
| Any command hangs/errors with connection refused | Service not started | `container system start` (accept the kernel install prompt on first run). |
| First `run` prompts about a kernel | No guest kernel installed | Accept the prompt, or `container system kernel set --recommended`. |
| `--network` errors / `container network` missing | On **macOS 15** | Networks need macOS 26. Use the `default` network and `--publish`; don't rely on cross-container traffic. |
| Container has no network / wrong subnet on macOS 15 | vmnet creates the net on first container start; helper and vmnet disagree on CIDR | Restart the service so the first container re-establishes the net; ensure nothing else holds `192.168.64.0/24`. Truly fixed only on macOS 26. |
| One container can't reach another | macOS 15 isolates containers on the virtual net | Requires macOS 26. As a workaround, publish ports and talk via the host. |
| `localhost:PORT` doesn't reach the service | No `--publish`, or bound to wrong host IP | Add `-p host:container`; remember the container listens on its own IP otherwise. Check `container ls` for the IP. |
| Service in container unreachable even by IP | App bound to `127.0.0.1` inside the container | Bind the app to `0.0.0.0` (safe — the virtual net isn't exposed externally). |
| Build OOMs or is very slow | Builder VM too small (default 2 CPU / 2 GiB) | `container builder stop && container builder delete && container builder start --cpus 8 --memory 16G`. |
| Anonymous volume left behind after `--rm` | Anonymous volumes don't auto-clean (unlike Docker) | `container volume ls -q | grep anon` then `container volume rm <vol>`, or `container volume prune`. |
| Host RAM stays high after heavy container work | Ballooning limitation — freed guest memory not returned | Restart the container(s). |
| Pulls hit HTTPS against a local registry | `--scheme auto` only uses HTTP for loopback/RFC1918/local-domain hosts | Pass `--scheme http` explicitly for a non-matching local registry. |
| Private Relay broke after DNS setup | `container system dns create --localhost` disables Private Relay; rule drops on restart | Expected; re-create the domain after restart, or remove it with `container system dns delete`. |
| Need a clean slate | Accumulated stopped containers/images/volumes | `container prune`, `container image prune -a`, `container volume prune`; check `container system df` first. |

## Shell completions

```bash
container --generate-completion-script zsh  > ~/.zsh/completion/_container   # then autoload
container --generate-completion-script bash > ~/.bash_completions/container  # then source it
container --generate-completion-script fish > ~/.config/fish/completions/container.fish
```
