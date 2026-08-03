# opencode-jail

A single-file Bash launcher that runs [OpenCode](https://opencode.ai) inside a Docker sandbox, keeping Node and all dependencies off your host machine. The `opencode` process is restricted to the directory from which `opencode-jail` is launched, but retains scoped access to the Docker Compose project rooted in that same directory.

## How it works

On first run, `opencode-jail` builds a Docker image from an embedded Dockerfile (Node 22 on Debian Bookworm with OpenCode and common CLI tools pre-installed). Subsequent runs reuse the cached image. If you edit the script — for example to add packages — the image is automatically rebuilt via `md5sum` change detection.

### Compose-scoped Docker access via a filtering shim

The host Docker socket is bind-mounted into the jail, but the real `docker` CLI is moved aside and a shim is installed at `/usr/local/bin/docker`. The shim filters every invocation against the label

```
com.docker.compose.project.working_dir = $(pwd)
```

so `opencode` can only see, exec into, log, stop, restart, etc. containers belonging to the compose project rooted at the launch directory. Host-wide operations (`run`, `create`, `build`, `image`, `network`, `volume`, `system`, `swarm`, …) are refused outright. `docker compose` is permitted but scope-escape flags (`-f`, `-p`, `--project-directory`, `--env-file`) are blocked.

> The shim is a **soft fence**, not a sandbox primitive. Anything that can talk to `/var/run/docker.sock` directly (raw `curl`, a statically-linked docker binary, etc.) bypasses it. Treat it as a guard rail for an obedient agent, not a security boundary against an adversary.

### Compose network auto-attach

If a compose project is already running with its `working_dir` equal to `$(pwd)`, `opencode-jail` enumerates the networks those containers are on and attaches the jail container to each one, so opencode can reach services by their compose service name (e.g. `http://test-web:8080`). Extra networks can be attached with `-n/--network NAME` (repeatable).

If no compose project is running, the jail starts on the default bridge only and prints a warning; bring your stack up with `docker compose up -d` first.

### Container hardening

The jail container runs with `--cap-drop=ALL`, `--security-opt=no-new-privileges`, and as the host user's UID/GID. The docker socket gid is added via `--group-add` so the unprivileged user can still use the socket through the shim.

## Prerequisites

- Docker installed and running

## Installation

```bash
cp opencode-jail ~/.local/bin/opencode-jail
chmod +x ~/.local/bin/opencode-jail
```

## Usage

```
opencode-jail [OPTIONS] [-- ARGS...]

Options:
  -e, --env KEY=VAL    Pass an environment variable to the container (repeatable)
  -n, --network NAME   Attach the jail container to an extra docker network (repeatable)
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

# Force a full image rebuild
opencode-jail --rebuild

# Rebuild without Docker layer cache
opencode-jail --rebuild --no-cache

# Attach the jail to an additional docker network
opencode-jail -n my-other-network

# Pass extra environment variables into the container
opencode-jail -e MY_API_KEY=abc123 -e DEBUG=1

# Pass arguments through to opencode
opencode-jail -- --help
```

If `--shell` is used while a jail container from the same working directory is already running, the script `docker exec`s into it instead of starting a new one.

## Configuration

### Environment variables

| Variable | Purpose |
|---|---|
| `GITHUB_TOKEN` | GitHub CLI authentication (`gh`) |
| `EDITOR` | Default editor (`vim`) |

Additional variables can be forwarded with `-e KEY=VALUE`.

### Adding packages

To install extra packages into the image, edit the `CUSTOM_APT_PACKAGES` variable near the top of the script:

```bash
CUSTOM_APT_PACKAGES="jq dnsutils openssh-client vim git tmux"
```

Save the file and the image will rebuild automatically on the next run.

## Volume mounts

| Host path | Container path | Purpose |
|---|---|---|
| `$(pwd)` | same path | Current working directory (read-write) |
| `/var/run/docker.sock` | `/var/run/docker.sock` | Host docker socket (filtered by shim) |
| `~/.opencode-home/.opencode` | `/home/node/.opencode` | OpenCode state |
| `~/.opencode-home/.local/state` | `/home/node/.local/state` | OpenCode local state |
| `~/.local/share/opencode` | `/home/node/.local/share/opencode` | OpenCode data and auth (shared credentials) |
| `~/.opencode-home/.config/opencode` | `/home/node/.config/opencode` | OpenCode config |
| `~/.opencode-home/.cache` | `/home/node/.cache` | OpenCode cache |
| `~/.vimrc` | `/home/node/.vimrc` (read-only) | Vim configuration |
| `~/.vim/` | `/home/node/.vim/` (read-only) | Vim plugins and runtime files |

The current directory is mounted at its exact host path so that compose volume mounts with relative paths resolve identically inside and outside the jail.

### Shared credentials

`~/.local/share/opencode` is mounted directly from the real host home so that API keys and auth tokens stored by OpenCode on the host are automatically available inside the jail without re-authenticating.

## Docker Compose support

Any compose project rooted at the launch directory works transparently:

- `docker ps` inside the jail is auto-filtered to project containers
- `docker exec <svc>`, `docker logs <svc>`, `docker compose ps|logs|restart`, etc. all operate against the running stack on the host
- Services are reachable by compose service name because the jail is attached to their networks

## Architecture

```
Host
└── opencode-jail (bash script)
    ├── builds/reuses Docker image: opencode-jail
    ├── detects compose project containers + networks for $(pwd)
    └── runs jail container
          ├── /var/run/docker.sock bind-mounted
          ├── /usr/local/bin/docker → filtering shim
          ├── /usr/local/libexec/docker-real → real docker CLI (hidden)
          ├── attached to each detected compose network
          ├── workspace mounted read-write at its host path
          ├── runs as host UID:GID, cap-drop=ALL, no-new-privileges
          └── opencode config dirs mounted from ~/.opencode-home/
```

## Included tools

The Docker image ships with these CLI tools alongside OpenCode:

- `gh` — GitHub CLI
- `docker` CLI (via shim) and `docker compose` plugin
- `vim`
- `openssh-client`, `dnsutils`, `jq`, `curl`

## Testing

`compose.yml` at the repo root defines a tiny `test-web` service that listens on port 8080 and returns `OK`. To verify the shim and network auto-attach:

```bash
# On the host
docker compose up -d

# Then, in the same directory
opencode-jail --shell

# Inside the jail
docker ps                           # should list jail-test-web only
curl http://test-web:8080           # should print OK
docker run --rm alpine echo nope    # should be refused by the shim
```

Tear it down on the host with `docker compose down`.

## License

MIT
