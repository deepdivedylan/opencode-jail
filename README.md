# opencode-jail

A single-file Bash launcher that runs [OpenCode](https://opencode.ai) inside a Docker sandbox, keeping Node and all dependencies off your host machine. The `opencode` process is restricted to the directory from which `opencode-jail` is launched, but retains full access to Docker containers defined in that same directory.

## How it works

On first run, `opencode-jail` builds a Docker image from an embedded Dockerfile (Node 22 on Debian Bookworm with OpenCode and common CLI tools pre-installed). Subsequent runs reuse the cached image. If you edit the script — for example to add packages — the image is automatically rebuilt via `md5sum` change detection.

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
  -r, --rebuild    Force rebuild of the Docker image
  -s, --shell      Start a bash shell instead of opencode
  -h, --help       Show this help message

Arguments after -- are passed through to opencode.
```

### Examples

```bash
# Run opencode interactively (builds image on first run)
opencode-jail

# Open a shell inside a running container
opencode-jail --shell

# Force a full image rebuild
opencode-jail --rebuild
```

If `--shell` is used while a container from the same working directory is already running, the script `docker exec`s into it rather than starting a new one.

## Configuration

### Environment variables

| Variable | Purpose |
|---|---|
| `GITHUB_TOKEN` | GitHub CLI authentication (`gh`) |

The token is passed through to the container automatically from your environment.

### Adding packages

To install extra packages into the image, edit the `CUSTOM_APT_PACKAGES` variable near the top of the script:

```bash
CUSTOM_APT_PACKAGES="git tmux sqlite3"
```

Save the file and the image will rebuild automatically on the next run.

## Volume mounts

| Host | Container | Purpose |
|---|---|---|
| `$(pwd)` | same path | Current working directory |
| `~/.opencode-home/.opencode` | `/home/node/.opencode` | OpenCode state |
| `~/.opencode-home/.local/state` | `/home/node/.local/state` | OpenCode local state |
| `~/.local/share/opencode` | `/home/node/.local/share/opencode` | OpenCode data and auth |
| `~/.opencode-home/.config/opencode` | `/home/node/.config/opencode` | OpenCode config |
| `/var/run/docker.sock` | `/var/run/docker.sock` | Docker socket |

The current directory is mounted at its exact host path so that any nested `docker` or `docker compose` commands launched from inside OpenCode resolve volume paths correctly on the host.

## Included tools

The Docker image ships with these CLI tools alongside OpenCode:

- `gh` — GitHub CLI
- `docker` CLI and `docker compose` plugin
- `openssh-client`, `dnsutils`, `jq`, `curl`, `build-essential`

## Testing Docker access

`docker-compose.test.yml` is included to verify that `opencode-jail` can reach the Docker daemon.

Run the test:

```bash
docker compose -f docker-compose.test.yml run --rm docker-access-test
```

Expected output:

```
=== Docker socket access test ===
[PASS] /var/run/docker.sock is present
[PASS] docker info succeeded
[PASS] Docker container access verified
NAMES    IMAGE    STATUS
...
=== Done ===
```

To exec into the test container manually (set `command` to `sleep infinity` in the compose file first):

```bash
docker exec -it docker-access-test sh
```

## License

MIT
