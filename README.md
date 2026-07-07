# docker-conf

Docker client configuration and CLI plugins, written because the default
docker CLI output was apparently designed for a 43-inch ultrawide mounted
in portrait mode, and I only have so many monitors.

This is my `$DOCKER_CONFIG` directory, versioned. It contains the client
config, the daemon config, a network bootstrap definition, and a set of
CLI plugins that make `docker` commands print things a human being can
read without induction training.

## The problem

Run `docker ps` with a Plex container and receive this in return:

```
plex   7dff68fcbccc   5a2dca89f4b5   Up 39 hours   0.0.0.0:3005->3005/tcp, 0.0.0.0:8324->8324/tcp, 0.0.0.0:32410->32410/udp, 0.0.0.0:32400->32400/tcp, 0.0.0.0:32469->32469/tcp, 5353/udp, 0.0.0.0:32412-32414->32412-32414/udp, 0.0.0.0:1901->1900/udp
```

One line. 280 characters. The ports are sorted by vibes. Every wildcard
bind is listed twice because IPv6 exists. The image is called
`5a2dca89f4b5` because the tag quietly moved to a newer image and docker
decided that was now your problem. And the container has been up for
"39 hours", a level of precision previously reserved for sundials.

`docker pull` is no better at scale: want to update your local images?
Enjoy pulling them one at a time to discover that 33 of 34 were already
current — while Docker Hub counts every polite inquiry against your rate
limit and eventually slams the door.

## The plugins

All plugins live in `cli-plugins/` and register as regular docker
subcommands. Python 3.11+, no dependencies outside the standard library,
because adding a `requirements.txt` to fix `docker ps` felt like losing.

### `docker pps` — ps, but legible

```
NAMES   ID            IMAGE                                       STATUS      PORTS
plex    7dff68fcbccc  lscr.io/linuxserver/plex:latest (outdated)  ● 1d 15:32  0.0.0.0:1901->1900/udp
                                                                              0.0.0.0:3005->3005/tcp
                                                                              0.0.0.0:8324->8324/tcp
                                                                              0.0.0.0:32400->32400/tcp
                                                                              ...
valkey  0d4d6926b5a9  valkey/valkey:9-alpine                      ● 01:08     6379/tcp
```

- One port mapping per line, sorted by host port. Duplicate `[::]`
  wildcard binds are collapsed, because listing every port twice is not
  "IPv6 support", it is stuttering.
- Status is a colored dot and the actual elapsed time (`● 1d 15:32`),
  computed from `StartedAt` instead of docker's artisanal "39 hours".
  Green is healthy, yellow is starting/paused, red means you have
  reading to do.
- Untagged image IDs are resolved back to the image name the container
  was started from, flagged `(outdated)` so you know a recreate is due.
- `docker pps -v [name]` prints the long form: mounts, networks, labels,
  the works — the stuff `docker inspect` hides inside a JSON haystack.

### `docker upgrade` — apt upgrade for your images

Updates local images the way every package manager on earth figured out
decades ago:

1. List local images.
2. Check all of them for updates **in parallel**, by comparing digests
   with registry HEAD requests — which Docker Hub does not count against
   your pull rate limit, unlike the manifest GETs that got you here.
3. Pull **only the outdated ones**, sequentially, with docker's native
   layer progress bars streaming to your terminal like nature intended.

`--dry-run` reports what would be pulled and touches nothing.
`--filter` narrows by substring, `--workers` tunes check concurrency.
buildx is used only as a fallback for private images.

### `docker img` — images grouped by registry

`docker images` sorted as a flat pile is fine if you own four images.
This groups them by registry and repository so `ghcr.io`, `lscr.io`,
and Docker Hub images stop pretending to be one big happy list.

### `docker net` — networks from a config file

Declares bridge networks in `networks.toml` (subnets, gateways,
internal flags, labels) and creates them idempotently with
`docker net --init networks.toml`. Also `--view` for a readable
network inspect that shows which containers are attached and their
IPs, without spelunking through `docker network inspect` JSON.

### `docker nvidia` — GPU sanity check

Three-step verification that the daemon is up, the NVIDIA container
toolkit is installed, and a CUDA container can actually see the GPU.
For when "it worked before the driver update" strikes again.

## Config files

| File | Purpose |
|------|---------|
| `config.json` | Client config: creds via `secretservice`, saner default `ps`/`images` formats |
| `daemon.json` | Daemon config: log rotation, DNS, CDI, nvidia runtime |
| `networks.toml` | Network definitions consumed by `docker net --init` |
| `cli_helper.py` | Shared plumbing for the plugins (colors, subprocess wrapper, metadata) |

Local state (`buildx/`, `model/`, `.token_seed`, `compose.env`) lives in
this directory at runtime but is gitignored — docker insists on scribbling
next to its config, and versioning its diary helps no one.

## Install

```sh
git clone https://github.com/delfianto/docker-conf ~/.config/docker
export DOCKER_CONFIG="$HOME/.config/docker"   # put this in your shell rc
```

Docker discovers plugins in `$DOCKER_CONFIG/cli-plugins` automatically.
Run `docker pps` and enjoy output that fits on hardware you actually own.

## Requirements

- Docker with the CLI plugin system (anything from this decade)
- Python ≥ 3.11
- buildx (optional, only for `docker upgrade` against private registries)
