# netbox

Custom [NetBox](https://github.com/netbox-community/netbox) container image, built and published by GitHub Actions.

The image extends the upstream `netboxcommunity/netbox:latest` base with extra plugins. Deployment of NetBox itself (Postgres, Redis, reverse proxy, env files, volumes) lives outside this repo — this repo only **builds and publishes the image**.

## Image

Published to GitHub Container Registry:

```
ghcr.io/metril/netbox
```

### Tag scheme

| Tag                | When it moves                                    |
| ------------------ | ------------------------------------------------ |
| `latest`           | Each push to `main`                              |
| `latest-plugins`   | Each push to `main` (compatibility alias)        |
| `main`             | Each push to `main`                              |
| `X.Y.Z`            | When tag `vX.Y.Z` is pushed (e.g. `1.0.0`)       |
| `X.Y`              | When tag `vX.Y.Z` is pushed (e.g. `1.0`)         |
| `X`                | When tag `vX.Y.Z` is pushed (e.g. `1`)           |
| `sha-<short>`      | Every build                                      |

For production, **pin to a `X.Y.Z` tag** so upstream base updates don't surprise you.

### Pull

```sh
docker pull ghcr.io/metril/netbox:1.0.0
```

Multi-arch: `linux/amd64`, `linux/arm64`.

## What's in the image

- Base: `netboxcommunity/netbox:latest`
- Plugins (see [plugin_requirements.txt](plugin_requirements.txt)):
  - [`netbox-interface-synchronization`](https://github.com/Cyclops256/netbox-interface-synchronization)
  - [`netbox_topology_views`](https://github.com/mattieserver/netbox-topology-views)
- Plugin enablement: [configuration/plugins.py](configuration/plugins.py), copied to `/etc/netbox/config/plugins.py` at build time.
- Static files are collected during the build (`manage.py collectstatic`).

## Adding or changing plugins

1. Add the package name to [plugin_requirements.txt](plugin_requirements.txt).
2. Add the plugin's Python module name to the `PLUGINS` list in [configuration/plugins.py](configuration/plugins.py). Add any plugin-specific settings to `PLUGINS_CONFIG`.
3. Commit, push to `main`. CI rebuilds and republishes `latest`.
4. When ready to ship, cut a release (see below).

## Cutting a release

```sh
git tag -a v1.1.0 -m "v1.1.0 - <summary>"
git push origin v1.1.0
gh release create v1.1.0 --generate-notes
```

The tag push triggers the workflow, which publishes `1.1.0`, `1.1`, and `1` image tags.

If your `git config user.email` is a private address, GitHub will reject the push. Either make your email public in [GitHub settings](https://github.com/settings/emails), or set the noreply form for this repo:

```sh
git config user.email "1517921+metril@users.noreply.github.com"
```

## Build workflow

Defined in [.github/workflows/build.yml](.github/workflows/build.yml). Triggers:

- **Push to `main`** — publishes `latest`, `latest-plugins`, `main`, `sha-<short>`.
- **Tag `v*`** — publishes the semver tags above.
- **Manual** (`workflow_dispatch`) — on-demand rebuild.
- **Weekly cron** (Sundays 06:00 UTC) — picks up upstream `netboxcommunity/netbox:latest` updates without needing a code change.

The workflow uses the repo's `GITHUB_TOKEN` to push to GHCR — no manually managed registry secrets.

## Running NetBox with this image

Deployment is intentionally **not** part of this repo. The image is a drop-in replacement for `netboxcommunity/netbox` in any deployment that already runs NetBox (Compose, Kubernetes, Nomad, etc.). At minimum NetBox needs:

- Postgres
- Redis (queue + cache)
- This image, with `/etc/netbox/config/` populated by your deployment (the image only ships `plugins.py`)
- An `rqworker` container running the same image with `manage.py rqworker`
- Mounted volumes for `media/` and `reports/`

See the upstream [`netbox-docker`](https://github.com/netbox-community/netbox-docker) project for a reference deployment.
