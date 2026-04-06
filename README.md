# opencode-jail

A single-file Bash launcher that runs [OpenCode](https://opencode.ai) inside a Docker sandbox, keeping Node and all dependencies off your host machine. The `opencode` process is restricted to the directory from which `opencode-jail` is launched, but retains full access to Docker Compose services defined in that same directory.

## How it works

On first run, `opencode-jail` builds a Docker image from an embedded Dockerfile (Node 22 on Debian Bookworm with OpenCode and common CLI tools pre-installed). Subsequent runs reuse the cached image. If you edit the script — for example to add packages — the image is automatically rebuilt via `md5sum` change detection.

### Docker-in-Docker (DinD) isolation

Rather than mounting the host Docker socket, `opencode-jail` spins up a dedicated Docker-in-Docker sidecar container for each session. The OpenCode container connects to this isolated daemon over TCP via `DOCKER_HOST`. This means:

- The OpenCode process **cannot see or interact with host containers**
- The host Docker socket is **never mounted** into the jail
- Docker Compose services launched from within the jail run inside the isolated daemon

The DinD sidecar and its private network are **scoped per launch directory** using a 12-character hash of the working directory path. Two concurrent sessions in different directories each get their own fully independent sidecar and network — they cannot interfere with each other.

When the OpenCode session ends the sidecar container and private network are automatically torn down.

## Prerequisites

- Docker installed and running

## Installation

Copy the script to somewhere on your `$PATH` and make it executable:

```bash
cp opencode-jail ~/.local/bin/opencode-jail
chmod +x ~/.local/bin/opencode-jail
```

## Usage

```
opencode-jail [OPTIONS] [-- ARGS...]

Options:
  -e, --env KEY=VAL    Pass an environment variable to the container (repeatable)
  -r, --rebuild        Force rebuild of the Docker image
      --no-cache       Rebuild the Docker image without using cache
  -s, --shell          Start a bash shell instead of opencode
  -h, --help           Show this help message

Arguments after -- are passed through to opencode.
```

### Examples

```bash
# Run opencode interactively (builds image on first run)
opencode-jail

# Open a shell inside the jail container
opencode-jail --shell

# Attach to an already-running container from the same directory
opencode-jail --shell

# Force a full image rebuild
opencode-jail --rebuild

# Rebuild without Docker layer cache
opencode-jail --rebuild --no-cache

# Pass extra environment variables into the container
opencode-jail -e MY_API_KEY=abc123 -e DEBUG=1

# Pass arguments through to opencode
opencode-jail -- --help
```

If `--shell` is used while a container from the same working directory is already running, the script `docker exec`s into it rather than starting a new one. In that case the DinD sidecar is left running — it is only torn down when the primary session exits.

## Configuration

### Environment variables

| Variable | Purpose |
|---|---|
| `GITHUB_TOKEN` | GitHub CLI authentication (`gh`) |

The token is passed through to the container automatically from your environment. Additional variables can be forwarded with `-e KEY=VALUE`.

### Adding packages

To install extra packages into the image, edit the `CUSTOM_APT_PACKAGES` variable near the top of the script:

```bash
CUSTOM_APT_PACKAGES="jq dnsutils openssh-client git tmux"
```

Save the file and the image will rebuild automatically on the next run.

## Volume mounts

| Host path | Container path | Purpose |
|---|---|---|
| `$(pwd)` | same path | Current working directory (read-write) |
| `~/.opencode-home/.opencode` | `$HOME/.opencode` | OpenCode state |
| `~/.opencode-home/.local/state` | `$HOME/.local/state` | OpenCode local state |
| `~/.local/share/opencode` | `$HOME/.local/share/opencode` | OpenCode data and auth (shared credentials) |
| `~/.opencode-home/.config/opencode` | `$HOME/.config/opencode` | OpenCode config |

The host Docker socket is **not** mounted. Docker access is provided exclusively through the DinD sidecar via `DOCKER_HOST`.

The current directory is mounted at its exact host path inside both the jail container and the DinD sidecar. This ensures that Docker Compose volume mounts (which are resolved by the Docker daemon) refer to paths that actually exist inside the sidecar.

### Shared credentials

`~/.local/share/opencode` is mounted directly from the real host home directory so that API keys and auth tokens stored by OpenCode on the host are automatically available inside the jail without re-authenticating.

## Docker Compose support

Docker Compose files in the launched directory work as expected inside the jail:

- The DinD daemon can pull images from Docker Hub and other registries
- Compose services can reference each other by service name (DNS resolution within the Compose network)
- Volume mounts using relative paths (e.g. `./src:/app/src`) resolve correctly because the working directory is mounted into the DinD sidecar at the same absolute path

## Architecture

```
Host
└── opencode-jail (bash script)
    ├── builds/reuses Docker image: opencode-standalone
    ├── starts DinD sidecar:  opencode-jail-dind-<dirhash>
    │     └── privileged container running dockerd on TCP (no TLS)
    ├── creates private network: opencode-jail-net-<dirhash>
    │     └── only the sidecar and jail container are attached
    └── runs jail container: opencode-standalone
          ├── DOCKER_HOST → tcp://<dind-ip>:2375
          ├── workspace mounted read-write
          └── opencode config dirs mounted from ~/.opencode-home/
```

## Included tools

The Docker image ships with these CLI tools alongside OpenCode:

- `gh` — GitHub CLI
- `docker` CLI and `docker compose` plugin
- `openssh-client`, `dnsutils`, `jq`, `curl`

## Testing

`docker-compose.test.yml` verifies that the DinD setup works correctly — that project files are visible via volume mounts and that Compose service networking functions inside the jail.

Run the test from within an `opencode-jail --shell` session:

```bash
docker compose -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from docker-access-test
```

Expected output:

```
=== DinD Docker Compose Test ===

--- Test 1: Volume mount (project files visible) ---
[PASS] docker-compose.test.yml found via volume mount

--- Test 2: Compose service networking ---
[PASS] Got response from test-web: OK

=== ALL TESTS PASSED ===
```

## License

MIT
